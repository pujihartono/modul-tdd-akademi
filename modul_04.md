
# Modul 4: Enrollment, Flash Messages & Dashboard (TDD)

**Tujuan**: Fitur enrollment kursus dengan Service Pattern, flash messages, dan dashboard.
**Hasil Visual**: Tombol Enroll berfungsi, Flash Messages, Dashboard menampilkan kursus enrolled.

### Apa itu Database Transaction?

Bayangkan kamu sedang melakukan transfer uang antar rekening bank:

1.  Kurangi saldo rekening A sebesar Rp 100.000
2.  Tambah saldo rekening B sebesar Rp 100.000

**Masalah:** Jika langkah 1 berhasil, tapi langkah 2 gagal (karena error jaringan), maka uang Rp 100.000 "hilang"!

**Solusi:** Kita bungkus kedua operasi dalam satu **Transaction**. Jika ada satu saja yang gagal, seluruh operasi di-**rollback** (dibatalkan).

### ACID Principles

Transaction database harus memenuhi 4 prinsip ACID:

| Prinsip         | Penjelasan                                           | Contoh di LMS                                       |
| --------------- | ---------------------------------------------------- | --------------------------------------------------- |
| **A**tomicity   | Semua operasi berhasil, atau tidak ada yang berhasil | Jika pembuatan enrollment gagal, jangan kirim email |
| **C**onsistency | Data tetap valid sebelum dan sesudah transaction     | Stok kursus tidak boleh negatif                     |
| **I**solation   | Transaction berjalan terpisah, tidak saling ganggu   | Dua user mendaftar bersamaan tidak membuat duplikat |
| **D**urability  | Setelah commit, data permanen walau sistem crash     | Setelah enrollment tersimpan, tidak hilang          |

### Mengapa Transaction Penting di LMS Kita?

Dalam proses pendaftaran kursus, ada beberapa operasi yang harus atomic:

1.  Buat record enrollment
2.  (Nanti) Kirim email konfirmasi
3.  (Nanti) Update statistik kursus
4.  (Nanti) Proses pembayaran

Jika operasi 3 atau 4 gagal, kita tidak ingin enrollment tetap tersimpan tanpa pembayaran!

---

## Langkah-langkah Implementasi

### Langkah 1: Memahami Syntax DB::transaction()

Laravel menyediakan facade `DB` untuk transaction:

```php
use Illuminate\Support\Facades\DB;

// Basic usage
DB::transaction(function () {
    // Semua query di sini dianggap satu kesatuan
    User::create([...]);
    Profile::create([...]);
    // Jika ada error, otomatis rollback
});

// With return value
$enrollment = DB::transaction(function () use ($user, $course) {
    $enrollment = Enrollment::create([...]);
    // ... operasi lain ...

    return $enrollment; // Auto-commit jika sukses
});

// Manual control (jika perlu lebih granular)
DB::beginTransaction();
try {
    Enrollment::create([...]);
    // ... operasi lain ...
    DB::commit();
} catch (Exception $e) {
    DB::rollBack();
    throw $e;
}
```

### Langkah 2: Menerapkan Transaction di Service

**File: `app/Services/EnrollUserInCourseService.php`**

```php
<?php

namespace App\Services;

use App\Models\Course;
use App\Models\Enrollment;
use App\Models\User;
use Illuminate\Support\Facades\DB;
use Exception;

class EnrollUserInCourseService
{
    /**
     * Enroll a user in a course using a database transaction.
     *
     * @param User $user
     * @param Course $course
     * @return Enrollment
     * @throws Exception
     */
    public function execute(User $user, Course $course): Enrollment
    {
        return DB::transaction(function () use ($user, $course) {
            // === VERIFIKASI DI DALAM TRANSACTION ===

            // Verifikasi 1: Instruktur tidak boleh mendaftar
            if ($course->instructor_id === $user->id) {
                throw new Exception('Anda adalah instruktur kursus ini.');
            }

            // Verifikasi 2: Cek apakah sudah terdaftar
            // Menggunakan lockForUpdate() untuk mencegah race condition
            $existingEnrollment = Enrollment::where('user_id', $user->id)
                ->where('course_id', $course->id)
                ->lockForUpdate()  // Lock row untuk isolasi
                ->first();

            if ($existingEnrollment) {
                throw new Exception('Anda sudah terdaftar di kursus ini.');
            }

            // === OPERASI DATABASE ===

            // Buat enrollment
            $enrollment = Enrollment::create([
                'user_id' => $user->id,
                'course_id' => $course->id,
                'enrolled_at' => now(),
            ]);

            // (Opsional) Update counter kursus
            $course->increment('enrollment_count');

            // (Opsional) Catat log aktivitas
            // ActivityLog::create([...]);

            return $enrollment;
        });
    }
}
```

**Penjelasan Kode:**

1.  **`DB::transaction(function () { ... })`**:
    - Semua query di dalam closure dianggap satu atomic operation.
    - Jika ada exception, Laravel otomatis melakukan `ROLLBACK`.
    - Jika sukses, Laravel otomatis melakukan `COMMIT`.

2.  **`lockForUpdate()`**:
    - Mengunci row yang sedang dibaca agar transaction lain tidak bisa mengubahnya.
    - Mencegah **race condition** (dua request bersamaan membuat data duplikat).

3.  **`return $enrollment`**:
    - Nilai return dari closure menjadi return value `DB::transaction()`.

### Langkah 3: Memahami Retry Mechanism

Laravel transaction mendukung retry otomatis jika terjadi deadlock:

```php
// Retry 3 kali jika terjadi deadlock
DB::transaction(function () {
    Enrollment::create([...]);
}, 3); // Max 3 attempts
```

**Kapan retry diperlukan?**

- High-traffic system dengan banyak concurrent writes
- Complex queries dengan banyak table joins
- Sistem dengan locking yang ketat

### Langkah 4: Manual Transaction (When to Use)

Kadang kita perlu kontrol lebih granular:

```php
use Illuminate\Support\Facades\DB;
use Exception;

class EnrollUserInCourseService
{
    public function execute(User $user, Course $course): Enrollment
    {
        DB::beginTransaction();

        try {
            // Verifikasi
            if ($course->instructor_id === $user->id) {
                throw new Exception('Instruktur tidak boleh mendaftar.');
            }

            // Operasi database
            $enrollment = Enrollment::create([
                'user_id' => $user->id,
                'course_id' => $course->id,
            ]);

            // Operasi non-database (misal: kirim email)
            // Email::send(...); // Di luar transaction, setelah commit

            DB::commit();

            // Kirim email setelah commit berhasil
            // $this->sendConfirmationEmail($user, $course);

            return $enrollment;

        } catch (Exception $e) {
            DB::rollBack();

            // Log error
            \Log::error('Enrollment failed', [
                'user_id' => $user->id,
                'course_id' => $course->id,
                'error' => $e->getMessage(),
            ]);

            throw $e;
        }
    }
}
```

**Kapan menggunakan manual transaction?**

- Perlu logging error spesifik sebelum re-throw
- Perlu melakukan operasi non-database (email, API call) yang tidak boleh di-rollback
- Perlu kontrol alur commit/rollback berdasarkan kondisi kompleks

---

## Penjelasan Detail: Race Condition & Locking

### Masalah: Race Condition

Bayangkan dua pengguna mencoba mendaftar di kursus yang sama secara bersamaan:

```
Waktu    | User A              | User B
---------|---------------------|---------------------
T+0ms    | SELECT enrollment   |
T+10ms   | (belum ada data)    | SELECT enrollment
T+20ms   |                     | (belum ada data)
T+30ms   | INSERT enrollment   |
T+40ms   |                     | INSERT enrollment ← DUPLIKAT!
```

**Solusi:** `lockForUpdate()`

```php
$existingEnrollment = Enrollment::where('user_id', $user->id)
    ->where('course_id', $course->id)
    ->lockForUpdate()  // Kunci row untuk transaksi ini
    ->first();
```

Dengan locking:

```
Waktu    | User A              | User B
---------|---------------------|---------------------
T+0ms    | SELECT ... FOR UPDATE |
T+10ms   | (lock acquired)     | SELECT ... FOR UPDATE (wait)
T+20ms   | INSERT enrollment   | (masih waiting)
T+30ms   | COMMIT (lock released)
T+40ms   |                     | SELECT ... FOR UPDATE (now ada data!)
T+50ms   |                     | Return existing enrollment
```

### Jenis Lock di Laravel

1.  **`lockForUpdate()`**:
    - Write lock (pessimistic locking)
    - Transaction lain tidak bisa read/write row yang terkunci
    - Cocok untuk operasi INSERT/UPDATE

2.  **`sharedLock()`**:
    - Read lock (shared lock)
    - Transaction lain bisa read, tapi tidak write
    - Cocok untuk operasi yang hanya membaca

3.  **`lockForUpdate()` dengan skipLocked** (Laravel 10+):
    - Skip row yang sedang di-lock, proses row lain
    - Berguna untuk queue/job processing

```php
// Skip locked rows
Enrollment::where('status', 'pending')
    ->lockForUpdate()
    ->skipLocked()
    ->first();
```

---

## Tips Pro: Best Practice di Dunia Kerja

### 1. Transaction Scope Sekecil Mungkin

**❌ Jangan:**

```php
DB::transaction(function () {
    // Verifikasi (baca saja, tidak perlu transaction)
    $this->validateUser($user);

    // Query database
    $enrollment = Enrollment::create([...]);

    // Kirim email (operasi lambat, jangan di transaction!)
    Mail::send(...);

    // API call (bisa gagal, jangan di transaction!)
    PaymentGateway::charge(...);
});
```

**✅ Lakukan:**

```php
// Verifikasi di luar transaction (baca saja)
$this->validateUser($user);

// Transaction hanya untuk operasi database
$enrollment = DB::transaction(function () {
    return Enrollment::create([...]);
});

// Operasi non-database setelah transaction
Mail::send(...);
PaymentGateway::charge(...);
```

**Kenapa?** Transaction yang lama membuat database lock lebih lama, mengurangi concurrency.

### 2. Gunakan Transaction untuk Multi-Table Operation

```php
DB::transaction(function () {
    // Tabel 1: Enrollment
    $enrollment = Enrollment::create([...]);

    // Tabel 2: Course (update counter)
    $course->increment('enrollment_count');

    // Tabel 3: User (update statistik)
    $user->increment('courses_enrolled_count');

    // Tabel 4: Activity Log
    ActivityLog::create([
        'user_id' => $user->id,
        'action' => 'enrolled',
        'entity_type' => Course::class,
        'entity_id' => $course->id,
    ]);
});
```

### 3. Handle Deadlock dengan Graceful Error

```php
use Illuminate\Database\QueryException;

public function execute(User $user, Course $course): Enrollment
{
    try {
        return DB::transaction(function () use ($user, $course) {
            // ... logic ...
        }, 3); // Retry 3 kali
    } catch (QueryException $e) {
        if ($e->getCode() == 40001) { // Deadlock code
            \Log::error('Deadlock detected during enrollment', [
                'user_id' => $user->id,
                'course_id' => $course->id,
            ]);

            throw new Exception('Terjadi konflik saat memproses pendaftaran. Silakan coba lagi.');
        }

        throw $e;
    }
}
```

### 4. Transaction dalam Service Lain

Service boleh memanggil service lain dalam satu transaction:

```php
class EnrollAndPayService
{
    public function __construct(
        private EnrollUserInCourseService $enrollService,
        private PaymentService $paymentService
    ) {}

    public function execute(User $user, Course $course, PaymentMethod $method): Enrollment
    {
        return DB::transaction(function () use ($user, $course, $method) {
            // Enroll dulu
            $enrollment = $this->enrollService->execute($user, $course);

            // Kemudian bayar
            $this->paymentService->process($user, $course, $method);

            return $enrollment;
        });
    }
}
```

---
## 🔴 Fase RED: Test Enrollment

```bash
php artisan make:test --pest EnrollmentTest
```

Buka `tests/Feature/EnrollmentTest.php`:

```php
<?php

// Test enrollment mencakup:
// 1. Happy path: siswa berhasil enroll
// 2. Edge case: tidak boleh enroll dua kali
// 3. Edge case: instructor tidak boleh enroll di kursusnya sendiri
// 4. Edge case: guest harus login dulu

use App\Models\Course;
use App\Models\Enrollment;
use App\Models\User;

it('allows student to enroll in a published course', function () {
    // Arrange: buat siswa dan kursus
    $student = User::factory()->create(['role' => 'student']);
    $course = Course::factory()->published()->create();

    // Act: kirim POST request enrollment (sebagai siswa)
    $response = $this->actingAs($student)
        ->post("/courses/{$course->slug}/enroll");

    // Assert: redirect kembali + flash message sukses
    $response->assertRedirect();
    $response->assertSessionHas('success');

    // Assert: data enrollment tersimpan di database
    $this->assertDatabaseHas('enrollments', [
        'user_id' => $student->id,
        'course_id' => $course->id,
    ]);
});

it('prevents duplicate enrollment', function () {
    // Arrange: siswa sudah enrolled
    $student = User::factory()->create(['role' => 'student']);
    $course = Course::factory()->published()->create();
    Enrollment::factory()->create([
        'user_id' => $student->id,
        'course_id' => $course->id,
    ]);

    // Act: coba enroll lagi
    $response = $this->actingAs($student)
        ->post("/courses/{$course->slug}/enroll");

    // Assert: redirect + flash error (bukan exception)
    $response->assertRedirect();
    $response->assertSessionHas('error');

    // Assert: tetap hanya 1 enrollment di database
    expect(Enrollment::count())->toBe(1);
});

it('prevents instructor from enrolling in own course', function () {
    // Arrange: instructor + kursus miliknya
    $instructor = User::factory()->create(['role' => 'instructor']);
    $course = Course::factory()->published()->create([
        'instructor_id' => $instructor->id,
    ]);

    // Act: instructor coba enroll ke kursusnya sendiri
    $response = $this->actingAs($instructor)
        ->post("/courses/{$course->slug}/enroll");

    // Assert: ditolak
    $response->assertRedirect();
    $response->assertSessionHas('error');
    expect(Enrollment::count())->toBe(0);
});

it('requires authentication to enroll', function () {
    // Arrange: kursus published
    $course = Course::factory()->published()->create();

    // Act: guest (tidak login) coba enroll
    $response = $this->post("/courses/{$course->slug}/enroll");

    // Assert: redirect ke login
    $response->assertRedirect('/login');
});
```

🔴 **Gagal** — Route dan Controller method `enroll` belum ada.

---

## 🟢 Fase GREEN: Service Pattern + Controller

### Service Class (Business Logic)

Buat folder dan file: `app/Services/EnrollUserInCourseService.php`:

```php
<?php

namespace App\Services;

use App\Models\Course;
use App\Models\Enrollment;
use App\Models\User;
use Exception;
use Illuminate\Support\Facades\DB;

/**
 * Service Class: memisahkan business logic dari controller.
 *
 * Mengapa pakai Service?
 * - Controller hanya handle HTTP request/response (thin controller).
 * - Logic "boleh enroll atau tidak" ada di sini (bisa di-test terpisah).
 * - Bisa dipanggil dari controller, job, command, dll.
 */
class EnrollUserInCourseService
{
    /**
     * Enroll a user in a course using a database transaction.
     *
     * @param User $user
     * @param Course $course
     * @return Enrollment
     * @throws Exception
     */
    public function execute(User $user, Course $course): Enrollment
    {
        return DB::transaction(function () use ($user, $course) {
            // === VERIFIKASI 1: Instruktur tidak boleh mendaftar di kursusnya sendiri ===
            if ($course->instructor_id === $user->id) {
                throw new Exception('Anda adalah instruktur kursus ini. Tidak perlu mendaftar.');
            }

            // === VERIFIKASI 2: Cek apakah user sudah terdaftar sebelumnya ===
            $existingEnrollment = Enrollment::where('user_id', $user->id)
                ->where('course_id', $course->id)
                ->first();

            if ($existingEnrollment) {
                throw new Exception('Anda sudah terdaftar di kursus ini.');
            }

            // === OPERASI DATABASE ===

            // Buat enrollment
            $enrollment = Enrollment::create([
                'user_id' => $user->id,
                'course_id' => $course->id,
                'enrolled_at' => now(),
            ]);

            // (Opsional) Update counter kursus
            $course->increment('enrollment_count');

            // (Opsional) Catat log aktivitas
            // ActivityLog::create([...]);

            return $enrollment;
        });
    }
}
```

### EnrollmentController

```bash
php artisan make:controller EnrollmentController
```

**`app/Http/Controllers/EnrollmentController.php`:**

```php
<?php

namespace App\Http\Controllers;

use App\Models\Course;
use App\Services\EnrollUserInCourseService;
use Exception;

class EnrollmentController extends Controller
{
    public function enroll(Course $course, EnrollUserInCourseService $enrollUserInCourseService)
    {
        try {
            $enrollUserInCourseService->execute(auth()->user(), $course);
            return redirect()->route('courses.show', $course->slug)
                ->with('success', 'Selamat! Anda berhasil terdaftar di kursus ini.');
        } catch (Exception $e) {
            return redirect()->route('courses.show', $course->slug)
                ->with('error', $e->getMessage());
        }
    }
}

```

### Routes

Tambahkan di `routes/web.php`, di dalam group `auth` yang sudah ada:

```php
use App\Http\Controllers\EnrollmentController;

// Enrollment — harus login (middleware auth)
// {course:slug} = route model binding via kolom slug
Route::post('/courses/{course:slug}/enroll', [EnrollmentController::class, 'store'])
    ->middleware('auth')
    ->name('courses.enroll');
```

### Share Flash Data ke Frontend

Buka `app/Http/Middleware/HandleInertiaRequests.php`:

```php
public function share(Request $request): array
{
    return [
        ...parent::share($request),
        'auth' => [
            'user' => $request->user(),
        ],
        // ✨ TAMBAHKAN: share flash messages ke semua halaman
        // Flash message hanya ada untuk 1 request (setelah redirect)
        'flash' => [
            'success' => $request->session()->get('success'),
            'error' => $request->session()->get('error'),
        ],
    ];
}
```

**Jalankan test:**

```bash
# Generate Wayfinder agar route baru ter-detect
php artisan wayfinder:generate

php artisan test tests/Feature/EnrollmentTest.php
```

🟢 **SEMUA HIJAU!**

---

## 🔵 Fase REFACTOR: Flash Messages + Dashboard UI

### 1. Komponen Flash Message

Buat `resources/js/components/flash-message.tsx`:

```tsx
/**
 * FlashMessage — notifikasi sementara (success/error).
 *
 * Ditampilkan setelah aksi seperti enrollment berhasil/gagal.
 * Otomatis hilang setelah 5 detik.
 *
 * Cara kerja:
 * 1. Backend: return back()->with('success', 'Berhasil!')
 * 2. Middleware: share flash ke Inertia props
 * 3. Komponen ini: baca flash dari usePage() → tampilkan notifikasi
 */

import { usePage } from '@inertiajs/react';
import { CheckCircle2, XCircle } from 'lucide-react';
import { useEffect, useState } from 'react';

interface FlashData {
    success?: string;
    error?: string;
}

export default function FlashMessage() {
    const { flash } = usePage().props as { flash?: FlashData };

    // ✅ Hanya track apakah user/timer sudah dismiss
    const [dismissed, setDismissed] = useState(false);

    useEffect(() => {
        if (!(flash?.success || flash?.error)) return;

        // ✅ Reset dismissed saat flash baru masuk
        // Tidak ada setState sinkron — keduanya async via setTimeout
        const resetTimer = setTimeout(() => setDismissed(false), 0);
        const hideTimer  = setTimeout(() => setDismissed(true), 5000);

        return () => {
            clearTimeout(resetTimer);
            clearTimeout(hideTimer);
        };
    }, [flash?.success, flash?.error]);

    // ✅ `visible` diderive — bukan dari state terpisah
    const visible = !dismissed && (!!flash?.success || !!flash?.error);

    if (!visible) return null;

    const isSuccess = !!flash?.success;
    const message   = flash?.success ?? flash?.error;

    return (
        <div className="fixed top-4 right-4 z-50">
            <div
                className={`flex items-center gap-3 rounded-lg px-4 py-3 shadow-lg transition-all duration-300 ${
                    isSuccess
                        ? 'bg-green-50 text-green-800 dark:bg-green-950 dark:text-green-200'
                        : 'bg-red-50 text-red-800 dark:bg-red-950 dark:text-red-200'
                }`}
            >
                {isSuccess
                    ? <CheckCircle2 className="h-5 w-5 text-green-600" />
                    : <XCircle     className="h-5 w-5 text-red-600"   />
                }

                <p className="text-sm font-medium">{message}</p>

                <button
                    onClick={() => setDismissed(true)}
                    className="ml-2 opacity-60 hover:opacity-100"
                >
                    ×
                </button>
            </div>
        </div>
    );
}
```

### 2. Pasang FlashMessage di Layout

Buka `resources/js/layouts/app-layout.tsx` dan tambahkan FlashMessage:

```tsx
// Di bagian import, tambahkan:
import FlashMessage from '@/components/flash-message';

// Di dalam return JSX, tambahkan <FlashMessage /> di atas {children}:
// Contoh (sesuaikan dengan struktur layout kamu):
export default function AppLayout({ children }: { children: React.ReactNode }) {
    return (
        <AppLayoutTemplate>
            {/* ✨ TAMBAHKAN: Flash message di atas konten */}
            <FlashMessage />
            {children}
        </AppLayoutTemplate>
    );
}
```

> **Catatan**: Struktur `app-layout.tsx` bisa bervariasi antar versi starter kit. Prinsipnya: letakkan `<FlashMessage />` di dalam layout, **sebelum** `{children}`.

### 3. Dashboard Controller

```bash
php artisan make:controller DashboardController
```

**`app/Http/Controllers/DashboardController.php`:**

```php
<?php

namespace App\Http\Controllers;

use App\Http\Resources\CourseResource;
use Illuminate\Http\Request;
use Inertia\Inertia;
use Inertia\Response;

class DashboardController extends Controller
{
    /**
     * Dashboard: tampilkan kursus yang di-enroll dan yang diajar.
     */
    public function index(Request $request): Response
    {
        $user = $request->user();

        // Kursus yang di-enroll siswa (via relasi belongsToMany)
        $enrolledCourses = $user->enrolledCourses()
            ->with('instructor')
            ->withCount('lessons')
            ->get();

        // Kursus yang diajar (jika user adalah instructor)
        $teachingCourses = $user->courses()
            ->withCount(['lessons', 'enrollments'])
            ->get();

        return Inertia::render('dashboard', [
            'enrolledCourses' => CourseResource::collection($enrolledCourses),
            'teachingCourses' => CourseResource::collection($teachingCourses),
        ]);
    }
}
```

### Update Route Dashboard

Di `routes/web.php`, cari route `/dashboard` bawaan dan ganti:

```php
use App\Http\Controllers\DashboardController;

// ❌ HAPUS route dashboard closure bawaan:
// Route::inertia('dashboard', 'dashboard')->name('dashboard');

// ✅ GANTI dengan Controller:
Route::get('dashboard', [DashboardController::class, 'index'])
    ->middleware(['auth', 'verified'])
    ->name('dashboard');
```

### 4. Halaman Dashboard

Buka `resources/js/pages/dashboard.tsx` dan **ganti seluruh isinya**:

```tsx
/**
 * Dashboard — halaman utama setelah login.
 *
 * Menampilkan:
 * - Kursus yang di-enroll (sebagai siswa)
 * - Kursus yang diajar (sebagai instruktur)
 */

import AppLayout from '@/layouts/app-layout';
import { Head, Link, usePage } from '@inertiajs/react';
import { type Course, type PageProps } from '@/types/models';
import {
    Card, CardContent, CardFooter, CardHeader, CardTitle,
} from '@/components/ui/card';
import { Badge } from '@/components/ui/badge';
import { Button } from '@/components/ui/button';
import { BookOpen, GraduationCap, PlusCircle } from 'lucide-react';

// Wayfinder: import fungsi show dan index dari CourseController
import {
    index as coursesIndex,
    show as courseShow,
} from '@/actions/App/Http/Controllers/CourseController';

interface Props {
    enrolledCourses: { data: Course[] };
    teachingCourses: { data: Course[] };
}

export default function Dashboard({ enrolledCourses, teachingCourses }: Props) {
    // Ambil data auth user dari shared props
    const { auth } = usePage<PageProps>().props;

    return (
        <AppLayout>
            <Head title="Dashboard" />

            <div className="mx-auto max-w-7xl space-y-8 px-4 py-8 sm:px-6 lg:px-8">
                {/* === Sambutan === */}
                <div>
                    <h1 className="text-3xl font-bold tracking-tight">
                        Welcome back, {auth.user?.name}!
                    </h1>
                    <p className="mt-1 text-muted-foreground">
                        Here&apos;s your learning overview.
                    </p>
                </div>

                {/* === Kursus yang Di-enroll === */}
                <section>
                    <h2 className="mb-4 text-xl font-semibold">My Enrolled Courses</h2>
                    {enrolledCourses.data.length === 0 ? (
                        // Empty state: belum enroll kursus apapun
                        <Card>
                            <CardContent className="flex flex-col items-center py-12">
                                <GraduationCap className="mb-4 h-12 w-12 text-muted-foreground" />
                                <p className="mb-4 text-lg font-medium text-muted-foreground">
                                    No courses yet
                                </p>
                                <Link href={coursesIndex.url()}>
                                    <Button>
                                        <BookOpen className="mr-2 h-4 w-4" />
                                        Browse Courses
                                    </Button>
                                </Link>
                            </CardContent>
                        </Card>
                    ) : (
                        <div className="grid grid-cols-1 gap-4 md:grid-cols-2 lg:grid-cols-3">
                            {enrolledCourses.data.map((course) => (
                                <Link
                                    key={course.id}
                                    href={courseShow.url({ course: course.slug })}
                                >
                                    <Card className="h-full transition-colors hover:border-primary">
                                        <CardHeader className="pb-2">
                                            <CardTitle className="line-clamp-2 text-base">
                                                {course.title}
                                            </CardTitle>
                                        </CardHeader>
                                        <CardFooter className="flex justify-between text-sm text-muted-foreground">
                                            <span>{course.instructor?.name}</span>
                                            {course.lessons_count !== undefined && (
                                                <Badge variant="outline">
                                                    {course.lessons_count} lessons
                                                </Badge>
                                            )}
                                        </CardFooter>
                                    </Card>
                                </Link>
                            ))}
                        </div>
                    )}
                </section>

                {/* === Kursus yang Diajar (hanya untuk instructor) === */}
                {teachingCourses.data.length > 0 && (
                    <section>
                        <h2 className="mb-4 text-xl font-semibold">
                            <PlusCircle className="mr-2 inline h-5 w-5" />
                            Courses You Teach
                        </h2>
                        <div className="grid grid-cols-1 gap-4 md:grid-cols-2 lg:grid-cols-3">
                            {teachingCourses.data.map((course) => (
                                <Link
                                    key={course.id}
                                    href={courseShow.url({ course: course.slug })}
                                >
                                    <Card className="h-full border-blue-200 transition-colors hover:border-blue-400 dark:border-blue-800">
                                        <CardHeader className="pb-2">
                                            <CardTitle className="line-clamp-2 text-base">
                                                {course.title}
                                            </CardTitle>
                                        </CardHeader>
                                        <CardFooter className="text-sm text-muted-foreground">
                                            <Badge variant="secondary">
                                                Instructor
                                            </Badge>
                                        </CardFooter>
                                    </Card>
                                </Link>
                            ))}
                        </div>
                    </section>
                )}
            </div>
        </AppLayout>
    );
}
```

### 5. Tambahkan Dashboard Test Jika Belum Ada

```bash
php artisan make:test --pest DashboardTest
```

`tests/Feature/DashboardTest.php`:

```php
<?php

use App\Models\Course;
use App\Models\Enrollment;
use App\Models\User;
use Inertia\Testing\AssertableInertia;

// Tambahkan test berikut

it('shows enrolled courses on dashboard', function () {
    $student = User::factory()->create(['role' => 'student']);
    $course = Course::factory()->published()->create();
    Enrollment::factory()->create([
        'user_id' => $student->id,
        'course_id' => $course->id,
    ]);

    $this->actingAs($student)
        ->get('/dashboard')
        ->assertOk()
        ->assertInertia(
            fn (AssertableInertia $page) => $page
                ->component('dashboard')
                ->has('enrolledCourses.data', 1)
        );
});

it('shows teaching courses for instructor', function () {
    $instructor = User::factory()->create(['role' => 'instructor']);
    Course::factory()->count(2)->create([
        'instructor_id' => $instructor->id,
    ]);

    $this->actingAs($instructor)
        ->get('/dashboard')
        ->assertOk()
        ->assertInertia(
            fn (AssertableInertia $page) => $page
                ->component('dashboard')
                ->has('teachingCourses.data', 2)
        );
});

it('shows empty state when no enrollments', function () {
    $student = User::factory()->create(['role' => 'student']);

    $this->actingAs($student)
        ->get('/dashboard')
        ->assertOk()
        ->assertInertia(
            fn (AssertableInertia $page) => $page
                ->has('enrolledCourses.data', 0)
        );
});
```

---

## ✅ Checkpoint Modul 4

```bash
# 1. Generate Wayfinder (route baru: courses.enroll)
php artisan wayfinder:generate

# 2. Semua test harus hijau
php artisan test
# Expected: semua passed (SmokeTest + Models + Catalog + Enrollment + Dashboard)

# 3. Re-seed database (agar data konsisten)
php artisan migrate:fresh --seed

# 4. ESLint + TypeScript check
npx eslint resources/js/
npx tsc --noEmit
# Expected: tidak ada error
```

**Verifikasi visual (pastikan `npm run dev` berjalan):**

1. ✅ Login sebagai student (email dari seeder) → Dashboard menampilkan enrolled courses.
2. ✅ Buka `/courses` → klik kursus → klik **"Enroll Now"** → flash message hijau muncul.
3. ✅ Kembali ke Dashboard → kursus baru muncul di "My Enrolled Courses".
4. ✅ Coba enroll lagi → flash message merah "Already enrolled".
5. ✅ Login sebagai instructor (`admin@rumahcoding.co.id` / `password`) → Dashboard menampilkan "Courses You Teach".
