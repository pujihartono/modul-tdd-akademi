# Modul 5: Halaman Lesson, Progress & Sertifikat (TDD)

**Tujuan**: Halaman belajar lesson, tracking progress, dan sertifikat otomatis.
**Hasil Visual**: Halaman Lesson (video + konten + navigasi), progress bar, sertifikat.

---

## 🔴 Fase RED: Test Lesson & Completion

```bash
php artisan make:test --pest LessonTest
```

`tests/Feature/LessonTest.php`:

```php
<?php

use App\Models\Course;
use App\Models\Certificate;
use App\Models\Enrollment;
use App\Models\Lesson;
use App\Models\LessonCompletion;
use App\Models\User;
use Inertia\Testing\AssertableInertia;

it('shows lesson page to enrolled student', function () {
    // Arrange: student enrolled di kursus yang punya lesson
    $student = User::factory()->create(['role' => 'student']);
    $course = Course::factory()->published()->create();
    $lesson = Lesson::factory()->create(['course_id' => $course->id]);
    Enrollment::factory()->create([
        'user_id' => $student->id,
        'course_id' => $course->id,
    ]);

    // Act & Assert: bisa akses halaman lesson
    $this->actingAs($student)
        ->get("/courses/{$course->slug}/lessons/{$lesson->id}")
        ->assertOk()
        ->assertInertia(
            fn (AssertableInertia $page) => $page
                ->component('lesson/show')
                ->has('lesson.data')
                ->where('lesson.data.title', $lesson->title)
        );
});

it('denies access to non-enrolled student', function () {
    // Student TIDAK enrolled → harus ditolak
    $student = User::factory()->create(['role' => 'student']);
    $course = Course::factory()->published()->create();
    $lesson = Lesson::factory()->create(['course_id' => $course->id]);

    $this->actingAs($student)
        ->get("/courses/{$course->slug}/lessons/{$lesson->id}")
        ->assertRedirect()
        ->assertSessionHas('error');
});

it('allows marking lesson as complete', function () {
    // Arrange
    $student = User::factory()->create(['role' => 'student']);
    $course = Course::factory()->published()->create();
    $lesson = Lesson::factory()->create(['course_id' => $course->id]);
    Enrollment::factory()->create([
        'user_id' => $student->id,
        'course_id' => $course->id,
    ]);

    // Act: kirim POST mark complete
    $this->actingAs($student)
        ->post("/courses/{$course->slug}/lessons/{$lesson->id}/complete")
        ->assertRedirect();

    // Assert: data completion tersimpan
    $this->assertDatabaseHas('lesson_completions', [
        'user_id' => $student->id,
        'lesson_id' => $lesson->id,
    ]);
});

it('handles idempotent completion (marking twice)', function () {
    // Menandai lesson selesai 2x harus aman (tidak duplikat)
    $student = User::factory()->create(['role' => 'student']);
    $course = Course::factory()->published()->create();
    $lesson = Lesson::factory()->create(['course_id' => $course->id]);
    Enrollment::factory()->create([
        'user_id' => $student->id,
        'course_id' => $course->id,
    ]);

    // Mark complete dua kali
    $this->actingAs($student)
        ->post("/courses/{$course->slug}/lessons/{$lesson->id}/complete");
    $this->actingAs($student)
        ->post("/courses/{$course->slug}/lessons/{$lesson->id}/complete");

    // Tetap hanya 1 record
    expect(LessonCompletion::count())->toBe(1);
});

it('auto-generates certificate when all lessons completed', function () {
    // Arrange: kursus dengan 2 lesson
    $student = User::factory()->create(['role' => 'student']);
    $course = Course::factory()->published()->create();
    $lesson1 = Lesson::factory()->create(['course_id' => $course->id]);
    $lesson2 = Lesson::factory()->create(['course_id' => $course->id]);
    Enrollment::factory()->create([
        'user_id' => $student->id,
        'course_id' => $course->id,
    ]);

    // Act: selesaikan semua lesson
    $this->actingAs($student)
        ->post("/courses/{$course->slug}/lessons/{$lesson1->id}/complete");
    $this->actingAs($student)
        ->post("/courses/{$course->slug}/lessons/{$lesson2->id}/complete");

    // Assert: sertifikat otomatis dibuat
    $certificate = Certificate::where('user_id', $student->id)
        ->where('course_id', $course->id)
        ->first();

    expect($certificate)->not->toBeNull();
    expect($certificate->certificate_number)->toStartWith('ACAD-');
});
```

🔴 **Gagal** — Controller dan routes belum ada.

---

## 🟢 Fase GREEN: LessonController + Completion Logic

### Routes

Tambahkan di `routes/web.php`:

```php
use App\Http\Controllers\LessonController;

// Lesson routes — harus login
Route::middleware('auth')->group(function () {
    // Tampilkan halaman lesson
    Route::get(
        '/courses/{course:slug}/lessons/{lesson}',
        [LessonController::class, 'show']
    )->name('lessons.show');

    // Tandai lesson selesai
    Route::post(
        '/courses/{course:slug}/lessons/{lesson}/complete',
        [LessonController::class, 'complete']
    )->name('lessons.complete');
});
```

### LessonPolicy

```bash
php artisan make:policy LessonPolicy
```

```php
<?php

namespace App\Policies;

use App\Models\Lesson;
use App\Models\User;
use Illuminate\Auth\Access\HandlesAuthorization;

class LessonPolicy
{
    use HandlesAuthorization;

    /**
     * User boleh melihat lesson jika:
     * - Dia instructor kursus tersebut, ATAU
     * - Dia sudah enrolled di kursus tersebut
     */
    public function view(User $user, Lesson $lesson): bool
    {
        $course = $lesson->course;

        if ($course->instructor_id === $user->id) {
            return true;
        }

        return $user->enrollments()
            ->whereCourseId($course->id)
            ->exists();
    }

    /**
     * User boleh menandai lesson selesai hanya jika enrolled
     * (instructor tidak perlu menandai lesson sebagai selesai).
     */
    public function complete(User $user, Lesson $lesson): bool
    {
        return $user->enrollments()
            ->whereCourseId($lesson->course_id)
            ->exists();
    }
}
```


### LessonServices
buat file **`app/Services/LessonService.php`:**

```php
<?php

namespace App\Services;

use App\Models\Certificate;
use App\Models\Course;
use App\Models\Lesson;
use App\Models\LessonCompletion;
use App\Models\User;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Str;

class LessonService
{
    /**
     * Hitung persentase progress user di suatu kursus.
     * Logika kalkulasi ini adalah business rule, bukan sekedar query.
     */
    public function getProgressPercent(Course $course, array $completedLessonIds): int
    {
        $totalLessons = $course->lessons->count();

        if ($totalLessons === 0) {
            return 0;
        }

        return (int) round((count($completedLessonIds) / $totalLessons) * 100);
    }

    /**
     * Tandai lesson sebagai selesai (idempotent).
     * Jika semua lesson selesai, buat sertifikat secara otomatis.
     */
    public function markComplete(Course $course, Lesson $lesson, User $user): void
    {
        DB::transaction(function () use ($course, $lesson, $user) {
            LessonCompletion::firstOrCreate([
                'user_id' => $user->id,
                'lesson_id' => $lesson->id,
            ]);

            $this->issuesCertificateIfCompleted($course, $user);
        });
    }

    /**
     * Buat sertifikat jika semua lesson di kursus sudah diselesaikan.
     * Dipisah menjadi method sendiri agar mudah di-test secara terisolasi.
     */
    private function issuesCertificateIfCompleted(Course $course, User $user): void
    {
        $lessonIds = $course->lessons()->pluck('id');
        $totalLessons = $lessonIds->count();
        $completedCount = $user->lessonCompletions()
            ->whereIn('lesson_id', $lessonIds)
            ->count();

        if ($completedCount < $totalLessons) {
            return;
        }

        Certificate::firstOrCreate(
            [
                'user_id' => $user->id,
                'course_id' => $course->id,
            ],
            [
                // Format: ACAD-XXXXXXXX
                'certificate_number' => 'ACAD-'.Str::upper(Str::random(8)),
            ]
        );
    }
}
```

### LessonController

```bash
php artisan make:controller LessonController
```

**`app/Http/Controllers/LessonController.php`:**

```php
<?php

namespace App\Http\Controllers;

use App\Http\Resources\CourseResource;
use App\Http\Resources\LessonResource;
use App\Models\Course;
use App\Models\Lesson;
use App\Services\LessonService;
use Illuminate\Http\RedirectResponse;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Gate;
use Inertia\Inertia;
use Inertia\Response;

class LessonController extends Controller
{
    public function __construct(
        private readonly LessonService $lessonService,
    ) {}

    /**
     * Tampilkan halaman lesson.
     * Hanya bisa diakses oleh enrolled student atau instructor.
     *
     * Otorisasi → LessonPolicy::view()
     */
    public function show(Course $course, Lesson $lesson, Request $request): Response|RedirectResponse
    {
        $user = $request->user();

        // can() menggunakan LessonPolicy::view() di balik layar,
        if (! $user->can('view', $lesson)) {
            return redirect()
                ->route('courses.show', $course)
                ->with('error', 'You must enroll to access this lesson.');
        }

        //Load hanya jika belum diload
        $course->loadMissing('lessons');
        $lesson->loadComments();

        $completedLessonIds = $user->completedLessonIdsForCourse($course);

        return Inertia::render('lesson/show', [
            'course' => new CourseResource($course),
            'lesson' => new LessonResource($lesson),
            'lessons' => LessonResource::collection($course->lessons),
            'completedLessonIds' => $completedLessonIds,
            'progressPercent' => $this->lessonService->getProgressPercent($course, $completedLessonIds),
            'certificate' => $user->certificateFor($course),
            'prevLesson' => ($prev = $lesson->previousLesson()) ? new LessonResource($prev) : null,
            'nextLesson' => ($next = $lesson->nextLesson()) ? new LessonResource($next) : null,
        ]);
    }

    /**
     * Tandai lesson sebagai selesai.
     *
     * Otorisasi → LessonPolicy::complete()
     * Logika    → LessonService::markComplete()
     */
    public function complete(Course $course, Lesson $lesson, Request $request): RedirectResponse
    {
        Gate::authorize('complete', $lesson);

        $this->lessonService
            ->markComplete($course, $lesson, $request->user());

        return back()->with('success', 'Lesson marked as complete!');
    }
}

```


**Jalankan test:**

```bash
php artisan test tests/Feature/LessonTest.php
```

🟢 **SEMUA HIJAU!**

---

## 🔵 Fase REFACTOR: Halaman Lesson UI

Buat `resources/js/pages/lesson/show.tsx`:

```tsx
/**
 * Halaman Lesson — tempat belajar.
 *
 * Layout 2 kolom:
 * - Kiri (2/3): Video + Konten + Navigasi prev/next
 * - Kanan (1/3): Sidebar progress + daftar lesson
 */

import { Head, Link, router } from '@inertiajs/react';
import {
    Award,
    CheckCircle2,
    ChevronLeft,
    ChevronRight,
    Circle,
    PlayCircle,
} from 'lucide-react';
import { useState } from 'react';
// Wayfinder imports
import { show as courseShow } from '@/actions/App/Http/Controllers/CourseController';
import { show as lessonShow } from '@/actions/App/Http/Controllers/LessonController';

import { Badge } from '@/components/ui/badge';
import { Button } from '@/components/ui/button';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import AppLayout from '@/layouts/app-layout';
import type { Course, Lesson } from '@/types/models';

interface Props {
    course: { data: Course };
    lesson: { data: Lesson };
    lessons: { data: Lesson[] };
    completedLessonIds: number[];
    progressPercent: number;
    certificate: { id: number; certificate_number: string } | null;
    prevLesson: { data: Lesson } | null;
    nextLesson: { data: Lesson } | null;
}

// ─── Helper: Tombol navigasi prev/next ───────────────────────────────────────
interface NavButtonProps {
    href?: string;
    direction: 'prev' | 'next';
}

function NavButton({ href, direction }: NavButtonProps) {
    const isPrev = direction === 'prev';
    const label = isPrev ? 'Previous' : 'Next';
    const icon = isPrev ? (
        <ChevronLeft className="h-4 w-4" />
    ) : (
        <ChevronRight className="h-4 w-4" />
    );

    const content = (
        <Button variant="outline" size="sm" className="gap-1" disabled={!href}>
            {isPrev && icon}
            {label}
            {!isPrev && icon}
        </Button>
    );

    return href ? <Link href={href}>{content}</Link> : content;
}
// ─────────────────────────────────────────────────────────────────────────────

export default function LessonShow({
    course,
    lesson,
    lessons,
    completedLessonIds = [],
    progressPercent = 0,
    certificate,
    prevLesson,
    nextLesson,
}: Props) {
    const [processing, setProcessing] = useState(false);

    // Cek apakah lesson ini sudah diselesaikan
    const isCompleted = completedLessonIds.includes(lesson.data.id);

    /** Handle klik "Mark as Complete" */
    const handleComplete = () => {
        router.post(
            `/courses/${course.data.slug}/lessons/${lesson.data.id}/complete`,
            {},
            {
                preserveScroll: true,
                onStart: () => setProcessing(true),
                onFinish: () => setProcessing(false),
            },
        );
    };

    return (
        <AppLayout>
            <Head title={`${lesson.data.title} — ${course.data.title}`} />

            <div className="mx-auto max-w-7xl px-4 py-6 sm:px-6 lg:px-8">
                {/* === Breadcrumb === */}
                <nav className="mb-4 text-sm text-muted-foreground">
                    <Link
                        href={courseShow.url({ course: course.data.slug })}
                        className="hover:text-foreground"
                    >
                        {course.data.title}
                    </Link>
                    <span className="mx-2">/</span>
                    <span className="text-foreground">{lesson.data.title}</span>
                </nav>

                <div className="grid grid-cols-1 gap-8 lg:grid-cols-3">
                    {/* ======= KONTEN UTAMA (2/3) ======= */}
                    <div className="space-y-6 lg:col-span-2">
                        {/* Video Player */}
                        {lesson.data.video_url && (
                            <div className="aspect-video overflow-hidden rounded-lg bg-black">
                                <iframe
                                    src={lesson.data.video_url}
                                    title={lesson.data.title}
                                    className="h-full w-full"
                                    allowFullScreen
                                    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture"
                                />
                            </div>
                        )}

                        {/* Konten Teks */}
                        {lesson.data.content && (
                            <Card>
                                <CardHeader>
                                    <CardTitle>{lesson.data.title}</CardTitle>
                                </CardHeader>
                                <CardContent>
                                    <div className="prose prose-sm dark:prose-invert max-w-none">
                                        <p className="leading-relaxed whitespace-pre-wrap">
                                            {lesson.data.content}
                                        </p>
                                    </div>
                                </CardContent>
                            </Card>
                        )}

                        {/* Tombol Mark Complete + Navigasi */}
                        <div className="flex flex-wrap items-center justify-between gap-4">
                            {/* Mark as Complete / Completed badge */}
                            {isCompleted ? (
                                <Badge
                                    variant="secondary"
                                    className="gap-1.5 px-4 py-2 text-sm"
                                >
                                    <CheckCircle2 className="h-4 w-4 text-green-600" />
                                    Completed
                                </Badge>
                            ) : (
                                <Button
                                    onClick={handleComplete}
                                    disabled={processing}
                                    className="gap-2"
                                >
                                    <CheckCircle2 className="h-4 w-4" />
                                    {processing
                                        ? 'Saving...'
                                        : 'Mark as Complete'}
                                </Button>
                            )}

                            {/* Prev / Next navigation */}
                            <div className="flex gap-2">
                                <NavButton
                                    direction="prev"
                                    href={
                                        prevLesson
                                            ? lessonShow.url({
                                                  course: course.data.slug,
                                                  lesson: prevLesson.data.id,
                                              })
                                            : undefined
                                    }
                                />
                                <NavButton
                                    direction="next"
                                    href={
                                        nextLesson
                                            ? lessonShow.url({
                                                  course: course.data.slug,
                                                  lesson: nextLesson.data.id,
                                              })
                                            : undefined
                                    }
                                />
                            </div>
                        </div>
                    </div>

                    {/* ======= SIDEBAR (1/3) ======= */}
                    <div className="space-y-4 lg:col-span-1">
                        {/* Progress Bar */}
                        <Card>
                            <CardContent className="pt-6">
                                <div className="mb-2 flex items-center justify-between text-sm">
                                    <span className="font-medium">
                                        Progress
                                    </span>
                                    <span className="text-muted-foreground">
                                        {progressPercent}%
                                    </span>
                                </div>
                                <div className="h-2.5 w-full overflow-hidden rounded-full bg-secondary">
                                    <div
                                        className="h-full rounded-full bg-primary transition-all duration-500"
                                        style={{ width: `${progressPercent}%` }}
                                    />
                                </div>

                                {/* Sertifikat badge */}
                                {certificate && (
                                    <div className="mt-4 rounded-lg bg-amber-50 p-3 text-center dark:bg-amber-950">
                                        <Award className="mx-auto mb-1 h-6 w-6 text-amber-600" />
                                        <p className="text-sm font-semibold text-amber-800 dark:text-amber-200">
                                            Certificate Earned!
                                        </p>
                                        <p className="text-xs text-amber-600">
                                            #{certificate.certificate_number}
                                        </p>
                                    </div>
                                )}
                            </CardContent>
                        </Card>

                        {/* Daftar Lesson */}
                        <Card>
                            <CardHeader className="pb-3">
                                <CardTitle className="text-base">
                                    Lessons
                                </CardTitle>
                            </CardHeader>
                            <CardContent>
                                <div className="space-y-1">
                                    {lessons.data.map((l) => {
                                        const isCurrent =
                                            l.id === lesson.data.id;
                                        const isDone =
                                            completedLessonIds.includes(l.id);

                                        return (
                                            <Link
                                                key={l.id}
                                                href={lessonShow.url({
                                                    course: course.data.slug,
                                                    lesson: l.id,
                                                })}
                                                className={`flex items-center gap-3 rounded-md px-3 py-2 text-sm transition-colors ${
                                                    isCurrent
                                                        ? 'bg-primary/10 font-medium text-primary'
                                                        : 'hover:bg-accent'
                                                }`}
                                            >
                                                <span className="flex h-6 w-6 shrink-0 items-center justify-center">
                                                    {isDone ? (
                                                        <CheckCircle2 className="h-5 w-5 text-green-600" />
                                                    ) : isCurrent ? (
                                                        <PlayCircle className="h-5 w-5 text-primary" />
                                                    ) : (
                                                        <Circle className="h-4 w-4 text-muted-foreground" />
                                                    )}
                                                </span>
                                                <span className="line-clamp-1">
                                                    {l.title}
                                                </span>
                                            </Link>
                                        );
                                    })}
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

---

## ✅ Checkpoint Modul 5

```bash
# 1. Semua test harus hijau
php artisan test
# Expected: semua passed

# 2. Re-seed
php artisan migrate:fresh --seed
```

**Verifikasi visual:**

1. ✅ Login → enroll kursus → klik lesson → halaman lesson tampil (video + konten).
2. ✅ Klik **"Mark as Complete"** → badge hijau "Completed" muncul, progress bar bergerak.
3. ✅ Navigasi **Prev/Next** bekerja.
4. ✅ Sidebar menunjukkan ✅ checkmark untuk lesson yang sudah selesai.
5. ✅ Selesaikan SEMUA lesson → **Certificate Earned** badge muncul di sidebar.
6. ✅ Kembali ke detail kursus → badge sertifikat juga tampil di sidebar.
