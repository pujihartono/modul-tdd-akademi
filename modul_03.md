
---

# Modul 3: Membangun Katalog Kursus (Read & Frontend — TDD)

**Tujuan**: Menampilkan data kursus ke browser, diverifikasi oleh test.
**Hasil Visual**: Halaman Katalog + Detail Kursus.

---

## 🔴 Fase RED: Feature Test Katalog

```bash
php artisan make:test --pest CourseCatalogTest
```

Buka `tests/Feature/CourseCatalogTest.php`:

```php
<?php

use App\Models\Course;
use App\Models\User;
use App\Models\Enrollment;
use Inertia\Testing\AssertableInertia;

it('displays published courses only', function () {
    // Buat 3 published + 2 draft → hanya 3 yang tampil
    Course::factory()->published()->count(3)->create();
    Course::factory()->unpublished()->count(2)->create();

    $this->get('/courses')
        ->assertOk()           // HTTP 200
        ->assertInertia(       // Verifikasi response Inertia
            fn (AssertableInertia $page) => $page
                ->component('course/index')       // Render halaman ini
                ->has('courses.data', 3)          // Hanya 3 kursus (yang published)
        );
});

it('includes instructor data without sensitive fields', function () {
    // KEAMANAN: pastikan email dan password tidak bocor ke frontend
    $instructor = User::factory()->create([
        'role' => 'instructor',
        'name' => 'Budi Santoso',
    ]);
    Course::factory()->published()->create(['instructor_id' => $instructor->id]);

    $this->get('/courses')
        ->assertOk()
        ->assertInertia(
            fn (AssertableInertia $page) => $page
                ->component('course/index')
                ->where('courses.data.0.instructor.name', 'Budi Santoso')
                ->missing('courses.data.0.instructor.password')  // ❌ Harus TIDAK ADA
                ->missing('courses.data.0.instructor.email')     // ❌ Harus TIDAK ADA
        );
});

it('shows course detail with lessons', function () {
    $course = Course::factory()->published()->create();
    \App\Models\Lesson::factory()->count(4)->create(['course_id' => $course->id]);

    // Akses detail kursus via slug
    $this->get("/courses/{$course->slug}")
        ->assertOk()
        ->assertInertia(
            fn (AssertableInertia $page) => $page
                ->component('course/show')          // Render halaman show
                ->has('course.data')                 // Ada data kursus
                ->where('course.data.title', $course->title)
                ->has('lessons.data', 4)             // Ada 4 lesson
        );
});

it('shows enrolled status for authenticated student', function () {
    $student = User::factory()->create(['role' => 'student']);
    $course = Course::factory()->published()->create();
    Enrollment::factory()->create([
        'user_id' => $student->id,
        'course_id' => $course->id,
    ]);

    $this->actingAs($student)
        ->get("/courses/{$course->slug}")
        ->assertOk()
        ->assertInertia(
            fn (AssertableInertia $page) => $page
                ->where('isEnrolled', true)  // Student sudah enrolled
        );
});

it('shows not enrolled for guest', function () {
    $course = Course::factory()->published()->create();

    $this->get("/courses/{$course->slug}")
        ->assertOk()
        ->assertInertia(
            fn (AssertableInertia $page) => $page
                ->where('isEnrolled', false)  // Guest belum enrolled
        );
});
```

🔴 **Gagal** — Controller dan Routes belum ada.

---

## 🟢 Fase GREEN: Controller, Routes, API Resources

### API Resources (Proteksi Data)

API Resource mengontrol **field apa saja** yang dikirim ke frontend.

```bash
php artisan make:resource UserResource
php artisan make:resource CourseResource
php artisan make:resource LessonResource
```

**`app/Http/Resources/UserResource.php`:**

```php
<?php

namespace App\Http\Resources;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

class UserResource extends JsonResource
{
    /**
     * Hanya kirim data yang AMAN untuk ditampilkan di frontend.
     * ❌ TIDAK menyertakan: email, password, remember_token
     */
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'name' => $this->name,
            'role' => $this->role,
        ];
    }
}
```

**`app/Http/Resources/CourseResource.php`:**

```php
<?php

namespace App\Http\Resources;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

class CourseResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'title' => $this->title,
            'slug' => $this->slug,
            'description' => $this->description,
            'price' => $this->price,
            'published_at' => $this->published_at?->format('Y-m-d'),
            // whenLoaded = hanya sertakan jika relasi sudah di-load
            'instructor' => new UserResource($this->whenLoaded('instructor')),
            // whenCounted = hanya sertakan jika sudah di-withCount
            'lessons_count' => $this->whenCounted('lessons'),
        ];
    }
}
```

**`app/Http/Resources/LessonResource.php`:**

```php
<?php

namespace App\Http\Resources;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

class LessonResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'title' => $this->title,
            'video_url' => $this->video_url,
            'content' => $this->content,
        ];
    }
}
```

### CourseController

```bash
php artisan make:controller CourseController
```

**`app/Http/Controllers/CourseController.php`:**

```php
<?php

namespace App\Http\Controllers;

use App\Http\Resources\CourseResource;
use App\Http\Resources\LessonResource;
use App\Models\Certificate;
use App\Models\Course;
use Illuminate\Http\Request;
use Inertia\Inertia;
use Inertia\Response;

class CourseController extends Controller
{
    /**
     * Katalog: daftar kursus yang sudah dipublish.
     * Bisa diakses siapa saja (guest maupun logged-in user).
     */
    public function index(): Response
    {
        $courses = Course::query()
            ->with('instructor')          // Eager load relasi instructor
            ->withCount('lessons')         // Hitung jumlah lesson per kursus
            ->whereNotNull('published_at') // Hanya yang sudah published
            ->latest()                     // Terbaru dulu
            ->paginate(9);                 // 9 per halaman (grid 3x3)

        return Inertia::render('course/index', [
            // CourseResource::collection() memfilter data via Resource
            'courses' => CourseResource::collection($courses),
        ]);
    }

    /**
     * Detail kursus: informasi lengkap + daftar lesson.
     * Route binding via slug: /courses/{course:slug}
     */
    public function show(Course $course, Request $request): Response
    {
        // Load relasi yang dibutuhkan (eager loading)
        $course->load('instructor', 'lessons');

        $user = $request->user(); // null jika guest

        // Cek apakah user saat ini sudah enrolled
        $isEnrolled = $user
            ? ($course->instructor_id === $user->id  // Instructor = otomatis akses
                || $user->enrollments()->where('course_id', $course->id)->exists())
            : false;

        // Progress: lesson mana saja yang sudah selesai
        $completedLessonIds = ($user && $isEnrolled)
            ? $user->lessonCompletions()
                ->whereIn('lesson_id', $course->lessons->pluck('id'))
                ->pluck('lesson_id')
                ->toArray()
            : [];

        // Sertifikat (jika ada)
        $certificate = $user
            ? Certificate::where('user_id', $user->id)
                ->where('course_id', $course->id)
                ->first()
            : null;

        return Inertia::render('course/show', [
            'course' => new CourseResource($course),
            'lessons' => LessonResource::collection($course->lessons),
            'isEnrolled' => $isEnrolled,
            'completedLessonIds' => $completedLessonIds,
            'certificate' => $certificate,
        ]);
    }
}
```

### Routes

Buka `routes/web.php`. **Hapus route placeholder** dari Modul 1 dan ganti:

```php
use App\Http\Controllers\CourseController;

// ❌ HAPUS route closure placeholder dari Modul 1:
// Route::get('/courses', function () { ... })->name('courses.index');

// ✅ GANTI dengan Controller routes:
// Public = bisa diakses tanpa login
Route::get('/courses', [CourseController::class, 'index'])
    ->name('courses.index');

// {course:slug} = route model binding via kolom 'slug' (bukan id)
Route::get('/courses/{course:slug}', [CourseController::class, 'show'])
    ->name('courses.show');
```

**Jalankan Wayfinder + test:**

```bash
# Generate ulang TypeScript definitions dari routes & controllers
php artisan wayfinder:generate

# Jalankan test
php artisan test tests/Feature/CourseCatalogTest.php
```

🟢 **SEMUA HIJAU!**

---

## 🔵 Fase REFACTOR: UI Lengkap

### 1. TypeScript Type Definitions

Buat file `resources/js/types/models.d.ts`:

```typescript
/**
 * Type definitions untuk aplikasi Akademi.
 *
 * File ini adalah "kontrak" antara backend (Laravel) dan frontend (React).
 * Setiap kali backend mengirim data via Inertia, frontend tahu
 * persis bentuk data yang diterima berkat types ini.
 */

/** Data auth user dari Inertia shared props. */
export interface AuthUser {
    id: number;
    name: string;
    email: string;
}

/**
 * Props global yang tersedia di setiap halaman via Inertia.
 * Digunakan dengan: usePage<PageProps>().props
 */
export interface PageProps {
    auth: {
        user: AuthUser | null; // null jika belum login (guest)
    };
    flash?: {
        success?: string;
        error?: string;
    };
    [key: string]: unknown; // Izinkan props tambahan
}

/** Data instruktur (subset dari User, tanpa data sensitif). */
export interface Instructor {
    id: number;
    name: string;
    role: 'student' | 'instructor';
}

/** Data kursus dari CourseResource. */
export interface Course {
    id: number;
    title: string;
    slug: string;
    description: string;
    price: string;          // String karena cast 'decimal:2'
    published_at: string | null;
    instructor?: Instructor;
    lessons_count?: number;
}

/** Data lesson dari LessonResource. */
export interface Lesson {
    id: number;
    title: string;
    video_url: string | null;
    content: string | null;
}

/** Wrapper untuk data yang dipaginasi oleh Laravel. */
export interface PaginatedData<T> {
    data: T[];
    links: {
        first: string;
        last: string;
        prev: string | null;
        next: string | null;
    };
    meta: {
        current_page: number;
        last_page: number;
        per_page: number;
        total: number;
    };
}
```

### 2. Instal Komponen shadcn/ui

```bash
# Komponen yang belum ada di starter kit
npx shadcn@latest add card badge separator
```

### 3. Halaman Katalog

Buka dan **ganti isi** `resources/js/pages/course/index.tsx`:

```tsx
/**
 * Halaman Katalog Kursus — menampilkan semua kursus published.
 * Data diterima dari CourseController::index() via Inertia.
 */

import AppLayout from '@/layouts/app-layout';
import { Head, Link } from '@inertiajs/react';
import { type Course, type PaginatedData } from '@/types/models';
import {
    Card, CardContent, CardFooter, CardHeader, CardTitle,
} from '@/components/ui/card';
import { Badge } from '@/components/ui/badge';
import { BookOpen, GraduationCap, User } from 'lucide-react';

// Import Wayfinder: fungsi show() dari CourseController
// Auto-generated oleh Wayfinder saat npm run dev berjalan
import { show } from '@/actions/App/Http/Controllers/CourseController';

/** Props yang diterima dari Controller (harus cocok dengan Inertia::render). */
interface Props {
    courses: PaginatedData<Course>;
}

export default function CourseIndex({ courses }: Props) {
    return (
        <AppLayout>
            <Head title="Course Catalog" />

            <div className="mx-auto max-w-7xl px-4 py-8 sm:px-6 lg:px-8">
                {/* === Header Halaman === */}
                <div className="mb-8">
                    <h1 className="text-3xl font-bold tracking-tight">
                        Course Catalog
                    </h1>
                    <p className="mt-2 text-muted-foreground">
                        Explore our courses and start learning today.
                    </p>
                </div>

                {/* === Grid Kartu Kursus === */}
                {courses.data.length === 0 ? (
                    // State kosong: belum ada kursus
                    <Card>
                        <CardContent className="flex flex-col items-center py-12">
                            <GraduationCap className="mb-4 h-12 w-12 text-muted-foreground" />
                            <p className="text-lg font-medium text-muted-foreground">
                                No courses available yet.
                            </p>
                        </CardContent>
                    </Card>
                ) : (
                    <div className="grid grid-cols-1 gap-6 md:grid-cols-2 lg:grid-cols-3">
                        {courses.data.map((course) => (
                            <Link
                                key={course.id}
                                // Wayfinder: generate URL type-safe dari controller
                                href={show.url({ course: course.slug })}
                                className="group"
                            >
                                <Card className="h-full transition-all duration-200 group-hover:-translate-y-1 group-hover:shadow-lg">
                                    <CardHeader className="pb-3">
                                        <div className="flex items-start justify-between gap-2">
                                            <CardTitle className="line-clamp-2 text-lg leading-snug group-hover:text-primary">
                                                {course.title}
                                            </CardTitle>
                                            <Badge variant="secondary" className="shrink-0">
                                                ${course.price}
                                            </Badge>
                                        </div>
                                    </CardHeader>

                                    <CardContent className="pb-3">
                                        {/* line-clamp-3 = maksimal 3 baris, sisanya di-truncate */}
                                        <p className="line-clamp-3 text-sm text-muted-foreground">
                                            {course.description}
                                        </p>
                                    </CardContent>

                                    <CardFooter className="flex items-center justify-between border-t pt-4 text-sm text-muted-foreground">
                                        <div className="flex items-center gap-1.5">
                                            <User className="h-4 w-4" />
                                            <span>{course.instructor?.name}</span>
                                        </div>
                                        {course.lessons_count !== undefined && (
                                            <div className="flex items-center gap-1.5">
                                                <BookOpen className="h-4 w-4" />
                                                <span>{course.lessons_count} lessons</span>
                                            </div>
                                        )}
                                    </CardFooter>
                                </Card>
                            </Link>
                        ))}
                    </div>
                )}

                {/* === Pagination === */}
                {courses.meta.last_page > 1 && (
                    <div className="mt-8 flex justify-center gap-2">
                        {courses.links.prev && (
                            <Link
                                href={courses.links.prev}
                                className="rounded-md border px-4 py-2 text-sm hover:bg-accent"
                            >
                                ← Previous
                            </Link>
                        )}
                        <span className="rounded-md bg-primary px-4 py-2 text-sm text-primary-foreground">
                            Page {courses.meta.current_page} of {courses.meta.last_page}
                        </span>
                        {courses.links.next && (
                            <Link
                                href={courses.links.next}
                                className="rounded-md border px-4 py-2 text-sm hover:bg-accent"
                            >
                                Next →
                            </Link>
                        )}
                    </div>
                )}
            </div>
        </AppLayout>
    );
}
```

### 4. Halaman Detail Kursus

Buat file `resources/js/pages/course/show.tsx`:

```tsx
/**
 * Halaman Detail Kursus — info lengkap, kurikulum, dan sidebar enrollment.
 * Layout 2 kolom: konten (2/3) + sidebar (1/3).
 */

import AppLayout from '@/layouts/app-layout';
import { Head, Link, useForm, usePage } from '@inertiajs/react';
import { type Course, type Lesson, type PageProps } from '@/types/models';
import {
    Card, CardContent, CardHeader, CardTitle,
} from '@/components/ui/card';
import { Badge } from '@/components/ui/badge';
import { Button } from '@/components/ui/button';
import { Separator } from '@/components/ui/separator';
import {
    Award, BookOpen, CheckCircle2, Clock, Lock, PlayCircle, User,
} from 'lucide-react';

// Wayfinder imports: fungsi dari Controller yang relevan
import { index as coursesIndex } from '@/actions/App/Http/Controllers/CourseController';

interface Props {
    course: { data: Course };
    lessons: { data: Lesson[] };
    isEnrolled: boolean;
    completedLessonIds: number[];
    certificate: { id: number; certificate_number: string } | null;
}

export default function CourseShow({
    course,
    lessons,
    isEnrolled,
    completedLessonIds = [],
    certificate,
}: Props) {
    // Akses shared props (auth user, flash messages)
    const { auth } = usePage<PageProps>().props;

    // useForm dari Inertia — untuk POST request enrollment
    const { post, processing } = useForm({});

    // Cek apakah user ini adalah instructor dari kursus ini
    const isInstructor = auth?.user
        ? course.data.instructor?.id === auth.user.id
        : false;

    /** Handle klik tombol Enroll. */
    const handleEnroll = () => {
        // POST ke route enroll (akan dibuat di Modul 4)
        post(`/courses/${course.data.slug}/enroll`);
    };

    return (
        <AppLayout>
            <Head title={course.data.title} />

            <div className="mx-auto max-w-7xl px-4 py-8 sm:px-6 lg:px-8">
                {/* === Breadcrumb Navigation === */}
                <nav className="mb-6 text-sm text-muted-foreground">
                    <Link
                        href={coursesIndex.url()}
                        className="hover:text-foreground"
                    >
                        Courses
                    </Link>
                    <span className="mx-2">/</span>
                    <span className="text-foreground">{course.data.title}</span>
                </nav>

                <div className="grid grid-cols-1 gap-8 lg:grid-cols-3">
                    {/* ======= KOLOM UTAMA (2/3) ======= */}
                    <div className="space-y-6 lg:col-span-2">
                        {/* Header */}
                        <div>
                            <h1 className="text-3xl font-bold tracking-tight">
                                {course.data.title}
                            </h1>
                            <div className="mt-3 flex items-center gap-4 text-sm text-muted-foreground">
                                <div className="flex items-center gap-1.5">
                                    <User className="h-4 w-4" />
                                    {course.data.instructor?.name}
                                </div>
                                <div className="flex items-center gap-1.5">
                                    <BookOpen className="h-4 w-4" />
                                    {lessons.data.length} lessons
                                </div>
                                {course.data.published_at && (
                                    <div className="flex items-center gap-1.5">
                                        <Clock className="h-4 w-4" />
                                        Published {course.data.published_at}
                                    </div>
                                )}
                            </div>
                        </div>

                        {/* About */}
                        <Card>
                            <CardHeader>
                                <CardTitle>About this Course</CardTitle>
                            </CardHeader>
                            <CardContent>
                                <p className="whitespace-pre-wrap leading-relaxed text-muted-foreground">
                                    {course.data.description}
                                </p>
                            </CardContent>
                        </Card>

                        {/* Curriculum */}
                        <Card>
                            <CardHeader>
                                <CardTitle className="flex items-center justify-between">
                                    Curriculum
                                    <Badge variant="outline">
                                        {lessons.data.length} lessons
                                    </Badge>
                                </CardTitle>
                            </CardHeader>
                            <CardContent>
                                <div className="divide-y">
                                    {lessons.data.map((lesson, index) => (
                                        <div
                                            key={lesson.id}
                                            className="flex items-center justify-between py-3"
                                        >
                                            <div className="flex items-center gap-3">
                                                {/* Nomor atau checkmark */}
                                                <span className="flex h-8 w-8 shrink-0 items-center justify-center rounded-full bg-primary/10 text-sm font-semibold text-primary">
                                                    {completedLessonIds.includes(lesson.id) ? (
                                                        <CheckCircle2 className="h-5 w-5 text-green-600" />
                                                    ) : (
                                                        index + 1
                                                    )}
                                                </span>
                                                <span className="font-medium">
                                                    {lesson.title}
                                                </span>
                                            </div>

                                            {/* Link Watch atau Lock icon */}
                                            {isEnrolled || isInstructor ? (
                                                <Link
                                                    href={`/courses/${course.data.slug}/lessons/${lesson.id}`}
                                                    className="flex items-center gap-1.5 text-sm font-medium text-primary hover:underline"
                                                >
                                                    <PlayCircle className="h-4 w-4" />
                                                    Watch
                                                </Link>
                                            ) : (
                                                <div className="flex items-center gap-1.5 text-sm text-muted-foreground">
                                                    <Lock className="h-3.5 w-3.5" />
                                                    Locked
                                                </div>
                                            )}
                                        </div>
                                    ))}
                                </div>
                            </CardContent>
                        </Card>
                    </div>

                    {/* ======= SIDEBAR (1/3) ======= */}
                    <div className="lg:col-span-1">
                        <Card className="sticky top-6">
                            <CardContent className="space-y-4 pt-6">
                                {/* Harga */}
                                <div className="text-center">
                                    <p className="text-4xl font-bold">${course.data.price}</p>
                                </div>

                                <Separator />

                                {/* Status / Action */}
                                {isInstructor ? (
                                    <div className="rounded-lg bg-blue-50 p-4 text-center dark:bg-blue-950">
                                        <CheckCircle2 className="mx-auto mb-2 h-8 w-8 text-blue-600" />
                                        <p className="font-semibold text-blue-800 dark:text-blue-200">
                                            You are teaching this course
                                        </p>
                                    </div>
                                ) : isEnrolled ? (
                                    <>
                                        <div className="rounded-lg bg-green-50 p-4 text-center dark:bg-green-950">
                                            <CheckCircle2 className="mx-auto mb-2 h-8 w-8 text-green-600" />
                                            <p className="font-semibold text-green-800 dark:text-green-200">
                                                You are enrolled!
                                            </p>
                                            <p className="mt-1 text-sm text-green-600 dark:text-green-400">
                                                Click any lesson to start learning.
                                            </p>
                                        </div>
                                        {/* Badge sertifikat jika sudah selesai */}
                                        {certificate && (
                                            <div className="rounded-lg bg-amber-50 p-4 text-center dark:bg-amber-950">
                                                <Award className="mx-auto mb-2 h-8 w-8 text-amber-600" />
                                                <p className="font-semibold text-amber-800 dark:text-amber-200">
                                                    Certificate Earned!
                                                </p>
                                                <p className="mt-1 text-xs text-amber-600">
                                                    #{certificate.certificate_number}
                                                </p>
                                            </div>
                                        )}
                                    </>
                                ) : (
                                    <div className="space-y-3">
                                        {auth?.user ? (
                                            <Button
                                                className="w-full"
                                                size="lg"
                                                onClick={handleEnroll}
                                                disabled={processing}
                                            >
                                                {processing ? 'Enrolling...' : 'Enroll Now'}
                                            </Button>
                                        ) : (
                                            <Link href="/login" className="block">
                                                <Button className="w-full" size="lg" variant="default">
                                                    Login to Enroll
                                                </Button>
                                            </Link>
                                        )}
                                        <p className="text-center text-xs text-muted-foreground">
                                            Full access to all lessons
                                        </p>
                                    </div>
                                )}

                                <Separator />

                                {/* Course Info */}
                                <div className="space-y-3 text-sm">
                                    <div className="flex justify-between">
                                        <span className="text-muted-foreground">Instructor</span>
                                        <span className="font-medium">{course.data.instructor?.name}</span>
                                    </div>
                                    <div className="flex justify-between">
                                        <span className="text-muted-foreground">Lessons</span>
                                        <span className="font-medium">{lessons.data.length}</span>
                                    </div>
                                </div>
                            </CardContent>
                        </Card>
                    </div>
                </div>
            </div>
        </AppLayout>
    );
}
```

> **Catatan tentang Wayfinder**: Jika import dari `@/actions/...` menampilkan error di IDE, pastikan `npm run dev` sedang berjalan — Wayfinder men-generate file-file di `resources/js/actions/` secara otomatis saat Vite dev server aktif.

### Perbaikan app-header.tsx

```tsx
<DropdownMenu>
    <DropdownMenuTrigger asChild>
        <Button
            variant="ghost"
            className="size-10 rounded-full p-1"
        >
            <Avatar className="size-8 overflow-hidden rounded-full">
                <AvatarImage
                    src={auth.user.avatar}
                    alt={auth.user.name}
                />
                <AvatarFallback className="rounded-lg bg-neutral-200 text-black dark:bg-neutral-700 dark:text-white">
                    {getInitials(auth.user.name)}
                </AvatarFallback>
            </Avatar>
        </Button>
    </DropdownMenuTrigger>
    <DropdownMenuContent className="w-56" align="end">
        <UserMenuContent user={auth.user} />
    </DropdownMenuContent>
</DropdownMenu>
```

ubah menjadi

```tsx
{/* Penambahan Conditional Rendering di sini */}
{auth.user ? (
    <DropdownMenu>
        <DropdownMenuTrigger asChild>
            <Button
                variant="ghost"
                className="size-10 rounded-full p-1"
            >
                <Avatar className="size-8 overflow-hidden rounded-full">
                    <AvatarImage
                        src={auth.user.avatar}
                        alt={auth.user.name}
                    />
                    <AvatarFallback className="rounded-lg bg-neutral-200 text-black dark:bg-neutral-700 dark:text-white">
                        {getInitials(auth.user.name)}
                    </AvatarFallback>
                </Avatar>
            </Button>
        </DropdownMenuTrigger>
        <DropdownMenuContent className="w-56" align="end">
            <UserMenuContent user={auth.user} />
        </DropdownMenuContent>
    </DropdownMenu>
) : (
    <div className="flex items-center space-x-4 pl-4">
        <Link
            href="/login"
            className="text-sm font-medium text-neutral-600 hover:text-black dark:text-neutral-400 dark:hover:text-white"
        >
            Log in
        </Link>
        <Link
            href="/register"
            className="text-sm font-medium text-neutral-600 hover:text-black dark:text-neutral-400 dark:hover:text-white"
        >
            Register
        </Link>
    </div>
)}
```

---

## ✅ Checkpoint Modul 3

```bash
# 1. Test harus hijau
php artisan test
# Expected: semua test passed

# 2. Generate Wayfinder
php artisan wayfinder:generate
# Expected: file-file di resources/js/actions/ ter-generate

# 3. ESLint + TypeScript check
npx eslint resources/js/pages/course/
npx tsc --noEmit
# Expected: tidak ada error
```

**Verifikasi visual (pastikan `npm run dev` berjalan):**

1. ✅ `http://akademi.test/courses` → grid kartu kursus tampil (2 kursus dari seeder).
2. ✅ Klik salah satu kursus → halaman detail dengan kurikulum dan sidebar.
3. ✅ Sebagai Guest → lesson terkunci 🔒, sidebar "Login to Enroll".
4. ✅ Login → tombol "Enroll Now" tampil (belum berfungsi, akan aktif di Modul 4).
5. ✅ Buka **DevTools → Network** → cek response JSON: **tidak ada** email/password di data instructor.
