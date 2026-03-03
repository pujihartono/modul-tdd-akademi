# Modul 6: Real-time Comments (Broadcasting — TDD)

**Tujuan**: Fitur komentar di setiap lesson dengan real-time updates via WebSocket.
**Hasil Visual**: Kolom komentar di bawah konten lesson, update tanpa refresh.

---

## 🔴 Fase RED: Test Comment

```bash
php artisan make:test --pest CommentTest
```

`tests/Feature/CommentTest.php`:

```php
<?php

use App\Models\Comment;
use App\Models\Course;
use App\Models\Enrollment;
use App\Models\Lesson;
use App\Models\User;
use Illuminate\Support\Facades\Event;

it('allows enrolled student to post comment', function () {
    // Arrange: student enrolled + lesson
    $student = User::factory()->create(['role' => 'student']);
    $course = Course::factory()->published()->create();
    $lesson = Lesson::factory()->create(['course_id' => $course->id]);
    Enrollment::factory()->create([
        'user_id' => $student->id,
        'course_id' => $course->id,
    ]);

    // Fake events agar broadcasting tidak benar-benar kirim
    Event::fake();

    // Act: POST komentar
    $this->actingAs($student)
        ->post("/courses/{$course->slug}/lessons/{$lesson->id}/comments", [
            'body' => 'This is a great lesson!',
        ])
        ->assertRedirect();

    // Assert: komentar tersimpan di database
    $this->assertDatabaseHas('comments', [
        'user_id' => $student->id,
        'lesson_id' => $lesson->id,
        'body' => 'This is a great lesson!',
    ]);
});

it('validates comment body is required', function () {
    $student = User::factory()->create(['role' => 'student']);
    $course = Course::factory()->published()->create();
    $lesson = Lesson::factory()->create(['course_id' => $course->id]);
    Enrollment::factory()->create([
        'user_id' => $student->id,
        'course_id' => $course->id,
    ]);

    // Act: POST tanpa body → harus validasi error
    $this->actingAs($student)
        ->post("/courses/{$course->slug}/lessons/{$lesson->id}/comments", [
            'body' => '', // Kosong
        ])
        ->assertSessionHasErrors('body');
});

it('prevents non-enrolled user from commenting', function () {
    $student = User::factory()->create(['role' => 'student']);
    $course = Course::factory()->published()->create();
    $lesson = Lesson::factory()->create(['course_id' => $course->id]);
    // Sengaja TIDAK enroll

    $this->actingAs($student)
        ->post("/courses/{$course->slug}/lessons/{$lesson->id}/comments", [
            'body' => 'Should not work',
        ])
        ->assertForbidden();

    expect(Comment::count())->toBe(0);
});
```

🔴 **Gagal** — Controller dan routes belum ada.

---

## 🟢 Fase GREEN: CommentController + Event

### CommentController

```bash
php artisan make:controller CommentController
```

**`app/Http/Controllers/CommentController.php`:**

```php
<?php

namespace App\Http\Controllers;

use App\Events\CommentPosted;
use App\Models\Course;
use App\Models\Lesson;
use Illuminate\Http\Request;

class CommentController extends Controller
{
    public function store(Request $request, Course $course, Lesson $lesson)
    {
        $validated = $request->validate([
            'body' => 'required|string|max:1000',
        ]);

        // Cek otorisasi (pastikan user berhak komentar)
        // Idealnya gunakan Policy, tapi ini logika inline untuk pragmatisme
        $isInstructor = $course->instructor_id === $request->user()->id;
        $isEnrolled = $request->user()->enrollments()->where('course_id', $course->id)->exists();

        abort_if(! $isInstructor && ! $isEnrolled, 403, 'Unauthorized action.');

        $comment = $lesson->comments()->create([
            'user_id' => $request->user()->id,
            'body' => $validated['body'],
        ]);

        // Trigger event ke Reverb
        broadcast(new CommentPosted($comment));

        // Return back jika lewat Inertia normal
        return back();
    }
}
```

### Event CommentPosted

```bash
php artisan make:event CommentPosted
```

**`app/Events/CommentPosted.php`:**

```php
<?php

namespace App\Events;

use App\Models\Comment;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Broadcasting\PrivateChannel;
use Illuminate\Contracts\Broadcasting\ShouldBroadcastNow;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

// Menggunakan ShouldBroadcastNow agar real-time instan tanpa antrean worker tambahan,
// sangat efisien untuk arsitektur server tunggal sederhana.
class CommentPosted implements ShouldBroadcastNow
{
    use Dispatchable, InteractsWithSockets, SerializesModels;

    /**
     * Data yang akan dikirim ke frontend.
     */
    public array $commentData;

    public function __construct(public Comment $comment)
    {
        // Eager load user agar frontend bisa menampilkan nama/avatar
        $this->comment->load('user');

        // Format data yang dikirim agar aman dan bersih
        $this->commentData = [
            'id' => $this->comment->id,
            'body' => $this->comment->body,
            'lesson_id' => $this->comment->lesson_id,
            'user' => [
                'id' => $this->comment->user->id,
                'name' => $this->comment->user->name,
            ],
            'created_at' => $this->comment->created_at->toIso8601String(),
        ];
    }

    /**
     * Tentukan channel broadcast.
     * Gunakan PrivateChannel agar hanya user yang berhak (enrolled/instructor) yang bisa listen.
     */
    public function broadcastOn(): array
    {
        return [
            new PrivateChannel('lesson.'.$this->comment->lesson_id),
        ];
    }

    /**
     * Nama event yang didengarkan oleh frontend.
     */
    public function broadcastAs(): string
    {
        return 'comment.posted';
    }
}
```

### Routes

Tambahkan di `routes/web.php` (di dalam group `auth`):

```php
use App\Http\Controllers\CommentController;

// Di dalam Route::middleware('auth')->group(function () { ... })
Route::post('/courses/{course:slug}/lessons/{lesson}/comments',
        [CommentController::class, 'store']
    )->name('comments.store');
```

Tambahkan di `routes/channels.php`:

```php
Broadcast::channel('lesson.{lessonId}', function (User $user, int $lessonId) {
    $lesson = Lesson::findOrFail($lessonId);

    // Logika otorisasi: Izinkan jika instruktur atau murid yang terdaftar
    $isInstructor = $lesson->course->instructor_id === $user->id;
    $isEnrolled = $user->enrollments()->where('course_id', $lesson->course_id)->exists();

    return $isInstructor || $isEnrolled;
});
```
ini perlu dilakukan karena kita menggunakan private channel, jika menggunakan public channel maka tidak perlu ditambahkan


### Update LessonController — Sertakan Comments

Di `app/Http/Controllers/LessonController.php`, method `show()`, load comments agar data dapat terkirim ke frontend:

```php
// Di akhir method show(), sebelum return Inertia::render, tambahkan:

$lesson->load(['comments' => function ($query) {
    $query->oldest()->with('user:id,name');
}]);

```

### Update LessonResource — Sertakan Comments

Di `app/Http/Resources/LessonResource.php`

```php
'comments' => $this->whenLoaded('comments'),
```

### Setup Laravel Reverb (WebSocket Server)

```bash
# Instal Reverb
php artisan install:broadcasting
```

Pilih **Reverb** saat diminta. Ini akan:
- Menambahkan konfigurasi di `.env`
- Menginstal `laravel-echo` dan `pusher-js` di frontend

Update `.env`:

```env
BROADCAST_CONNECTION=reverb

REVERB_APP_ID=akademi
REVERB_APP_KEY=akademi-key
REVERB_HOST="0.0.0.0"
REVERB_PORT=8080
REVERB_SCHEME=https

// Disesuaikan dengan lokasi masing-masing
REVERB_TLS_CERT="/Users/hartono/Library/Application Support/Herd/config/valet/Certificates/akademi.test.crt"
REVERB_TLS_KEY="/Users/hartono/Library/Application Support/Herd/config/valet/Certificates/akademi.test.key"

VITE_REVERB_APP_KEY="${REVERB_APP_KEY}"
VITE_REVERB_HOST="akademiuji.test"
VITE_REVERB_PORT=8080
VITE_REVERB_SCHEME=https
```

### Ubah konfig broadcasting `config/broadcasting.php`

```php
'client_options' => [
    // Agar tidak error ketika menggunakan Herd
    'verify' => false,
],
```

### ubah app.tsx

```tsx
import { createInertiaApp } from '@inertiajs/react';
// 1. Impor Pusher dan Echo
import Echo from 'laravel-echo';
import { resolvePageComponent } from 'laravel-vite-plugin/inertia-helpers';
import Pusher from 'pusher-js';
import { StrictMode } from 'react';
import { createRoot } from 'react-dom/client';
import '../css/app.css';
import { initializeTheme } from '@/hooks/use-appearance';

// 2. Lekatkan Pusher ke window agar dikenali secara internal oleh Echo
window.Pusher = Pusher;

// 3. Inisialisasi dan lekatkan Echo ke window
window.Echo = new Echo({
    broadcaster: 'reverb',
    key: import.meta.env.VITE_REVERB_APP_KEY,
    wsHost: import.meta.env.VITE_REVERB_HOST,
    wsPort: import.meta.env.VITE_REVERB_PORT ?? 80,
    wssPort: import.meta.env.VITE_REVERB_PORT ?? 443,
    forceTLS: (import.meta.env.VITE_REVERB_SCHEME ?? 'https') === 'https',
    enabledTransports: ['ws', 'wss'],
});

const appName = import.meta.env.VITE_APP_NAME || 'Laravel';

createInertiaApp({
    title: (title) => (title ? `${title} - ${appName}` : appName),
    resolve: (name) =>
        resolvePageComponent(
            `./pages/${name}.tsx`,
            import.meta.glob('./pages/**/*.tsx'),
        ),
    setup({ el, App, props }) {
        const root = createRoot(el);

        root.render(
            <StrictMode>
                <App {...props} />
            </StrictMode>,
        );
    },
    progress: {
        color: '#4B5563',
    },
});

// This will set light / dark mode on load...
initializeTheme();

```

### edit file vite-env.d.ts menjadi

```ts
/// <reference types="vite/client" />
import type { AxiosInstance } from 'axios';
import type Echo from 'laravel-echo';
import type Pusher from 'pusher-js';

declare global {
    interface Window {
        axios: AxiosInstance;
        Pusher: typeof Pusher;
        Echo: Echo;
    }
}
```

**Pastikan jalankan Reverb dan NPM saat uji coba**

```bash
php artisan reverb:start --host="0.0.0.0" --port=8080 --hostname="akademi.test" --debug
```

```bash
npm run dev
```


**Jalankan test:**

```bash
php artisan wayfinder:generate
php artisan test tests/Feature/CommentTest.php
```


🟢 **SEMUA HIJAU!**

---

## 🔵 Fase REFACTOR: Komponen Komentar + Real-time

### Komponen LessonComments

Buat `resources/js/components/lesson-comments.tsx`:

```tsx
/**
 * LessonComments — kolom diskusi real-time di halaman lesson.
 */

import { useForm } from '@inertiajs/react';
import { MessageCircle, Send } from 'lucide-react';
import { useEffect, useState } from 'react';
import { Button } from '@/components/ui/button';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';

// ─── Deklarasi global agar TypeScript tidak error "Echo not defined" ──────────
declare global {
    interface Window {
        Echo?: {
            channel: (name: string) => {
                listen: (event: string, callback: (data: unknown) => void) => void;
            };
            leaveChannel: (name: string) => void;
        };
    }
}
// ─────────────────────────────────────────────────────────────────────────────

interface CommentData {
    id: number;
    body: string;
    user: { id: number; name: string };
    created_at: string;
}

interface Props {
    lessonId: number;
    courseSlug: string;
    initialComments: CommentData[];
}

export default function LessonComments({
    lessonId,
    courseSlug,
    initialComments,
}: Props) {
    const [comments, setComments] = useState<CommentData[]>(initialComments);

    const { data, setData, post, processing, reset, errors } = useForm({
        body: '',
    });

    useEffect(() => {
        // ✅ Guard: Echo belum tentu di-load (mis. user belum login / Pusher belum init)
        if (!window.Echo) {
            console.warn('Laravel Echo belum tersedia. Real-time dinonaktifkan.');
            return;
        }

        const channelName = `lesson.${lessonId}`;
        const channel = window.Echo.channel(channelName);

        channel.listen('CommentPosted', (event: unknown) => {
            // ✅ Cast setelah validasi tipe
            const comment = event as CommentData;
            setComments((prev) => [comment, ...prev]);
        });

        return () => {
            window.Echo?.leaveChannel(channelName);
        };
    }, [lessonId]);

    const handleSubmit = (e: React.FormEvent) => {
        e.preventDefault();

        post(`/courses/${courseSlug}/lessons/${lessonId}/comments`, {
            preserveScroll: true,
            onSuccess: () => reset(),
        });
    };

    return (
        <Card className="w-full">
            <CardHeader>
                <CardTitle className="flex items-center gap-2 text-lg">
                    <MessageCircle className="h-5 w-5" />
                    Discussion ({comments.length})
                </CardTitle>
            </CardHeader>
            <CardContent className="space-y-4">
                {/* === Form Komentar === */}
                <form onSubmit={handleSubmit} className="space-y-3">
                    <textarea
                        value={data.body}
                        onChange={(e) => setData('body', e.target.value)}
                        placeholder="Write a comment..."
                        className="w-full rounded-md border bg-background px-3 py-2 text-sm ring-offset-background placeholder:text-muted-foreground focus-visible:ring-2 focus-visible:ring-ring focus-visible:outline-none"
                        rows={3}
                    />
                    {errors.body && (
                        <p className="text-sm text-red-500">{errors.body}</p>
                    )}
                    <div className="flex justify-end">
                        <Button
                            type="submit"
                            size="sm"
                            disabled={processing || !data.body.trim()}
                            className="gap-1.5"
                        >
                            <Send className="h-3.5 w-3.5" />
                            {processing ? 'Posting...' : 'Post Comment'}
                        </Button>
                    </div>
                </form>

                {/* === Daftar Komentar === */}
                {comments.length === 0 ? (
                    <p className="py-4 text-center text-sm text-muted-foreground">
                        No comments yet. Be the first to share your thoughts!
                    </p>
                ) : (
                    <div className="max-h-96 space-y-3 overflow-y-auto">
                        {comments.map((comment) => (
                            <div
                                key={comment.id}
                                className="rounded-lg bg-accent/50 p-3"
                            >
                                <div className="mb-1 flex items-center justify-between">
                                    <span className="text-sm font-semibold">
                                        {comment.user.name}
                                    </span>
                                    <span className="text-xs text-muted-foreground">
                                        {comment.created_at}
                                    </span>
                                </div>
                                <p className="text-sm">{comment.body}</p>
                            </div>
                        ))}
                    </div>
                )}
            </CardContent>
        </Card>
    );
}
```

### Perbaiki Halaman Lesson

Buka `resources/js/pages/lesson/show.tsx` dan ganti dengan :

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
    MessageSquare,
    PlayCircle,
} from 'lucide-react';
import { useEffect, useRef, useState } from 'react';
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
    const [commentBody, setCommentBody] = useState('');
    const [realtimeComments, setRealtimeComments] = useState<any[]>([]);
    const [currentLessonId, setCurrentLessonId] = useState(lesson.data.id);

    if (lesson.data.id !== currentLessonId) {
        setRealtimeComments([]); // Kosongkan komentar realtime lama
        setCurrentLessonId(lesson.data.id); // Update ID yang sedang dilacak
    }

    const commentsEndRef = useRef<HTMLDivElement>(null);
    const mergedComments = [
        ...(lesson.data.comments || []),
        ...realtimeComments,
    ];

    const displayComments = Array.from(
        new Map(mergedComments.map((c) => [c.id, c])).values(),
    );

    useEffect(() => {
        commentsEndRef.current?.scrollIntoView({ behavior: 'smooth' });
    }, [displayComments]);

    useEffect(() => {
        if (!lesson.data.id || !window.Echo) return;

        const channelName = `lesson.${lesson.data.id}`;

        const channel = window.Echo.private(channelName).listen(
            '.comment.posted',
            (e: any) => {
                setRealtimeComments((prev) => {
                    const isExist = prev.some((c) => c.id === e.commentData.id);
                    if (isExist) return prev;
                    return [...prev, e.commentData];
                });
            },
        );

        return () => {
            channel.stopListening('.comment.posted');
            window.Echo.leave(channelName);
        };
    }, [lesson.data.id]);

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

    const handlePostComment = (e: React.FormEvent) => {
        e.preventDefault();
        if (!commentBody.trim()) return;

        router.post(
            `/courses/${course.data.slug}/lessons/${lesson.data.id}/comments`,
            { body: commentBody },
            {
                preserveScroll: true,
                onSuccess: () => {
                    setCommentBody('');
                    // Reload data tidak sepenuhnya wajib karena Echo akan menyuntikkan data ke user lain,
                    // Tapi Inertia otomatis me-refresh props saat request POST berhasil,
                    // state di atas akan tersinkronisasi.
                },
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

                        {/* ========================================= */}
                        {/* FITUR KOMENTAR REAL-TIME                  */}
                        {/* ========================================= */}
                        <Card className="mt-8">
                            <CardHeader>
                                <CardTitle className="flex items-center gap-2">
                                    <MessageSquare className="h-5 w-5" />
                                    Diskusi Lesson ({displayComments.length})
                                </CardTitle>
                            </CardHeader>
                            <CardContent className="space-y-6">
                                {/* Form Komentar */}
                                <form
                                    onSubmit={handlePostComment}
                                    className="flex gap-3"
                                >
                                    <div className="flex-1">
                                        <textarea
                                            value={commentBody}
                                            onChange={(e) =>
                                                setCommentBody(e.target.value)
                                            }
                                            placeholder="Ada pertanyaan atau diskusi?"
                                            className="w-full rounded-md border border-input bg-transparent px-3 py-2 text-sm shadow-sm placeholder:text-muted-foreground focus-visible:ring-1 focus-visible:ring-ring focus-visible:outline-none disabled:cursor-not-allowed disabled:opacity-50"
                                            rows={2}
                                        />
                                    </div>
                                    <Button
                                        type="submit"
                                        disabled={!commentBody.trim()}
                                    >
                                        Kirim
                                    </Button>
                                </form>

                                {/* Daftar Komentar */}
                                <div className="max-h-[500px] space-y-4 overflow-y-auto pr-2">
                                    {displayComments.map((comment) => (
                                        <div
                                            key={comment.id}
                                            className="flex gap-3 text-sm"
                                        >
                                            <div className="flex h-8 w-8 shrink-0 items-center justify-center rounded-full bg-primary/10 font-bold text-primary">
                                                {comment.user.name.charAt(0)}
                                            </div>
                                            <div className="flex-1 rounded-lg bg-muted p-3">
                                                <div className="mb-1 flex items-center justify-between">
                                                    <span className="font-semibold text-foreground">
                                                        {comment.user.name}
                                                    </span>
                                                    <span className="text-xs text-muted-foreground">
                                                        {new Date(
                                                            comment.created_at,
                                                        ).toLocaleTimeString(
                                                            [],
                                                            {
                                                                hour: '2-digit',
                                                                minute: '2-digit',
                                                            },
                                                        )}
                                                    </span>
                                                </div>
                                                <p className="whitespace-pre-wrap text-muted-foreground">
                                                    {comment.body}
                                                </p>
                                            </div>
                                        </div>
                                    ))}
                                    {/* Ref untuk auto-scroll ke bawah saat ada komentar baru */}
                                    <div ref={commentsEndRef} />
                                </div>
                            </CardContent>
                        </Card>
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

> **Catatan**: Fitur komentar tetap berfungsi TANPA Reverb — hanya saja komentar baru tidak muncul real-time (perlu refresh halaman). Reverb membuat komentar muncul instan di semua browser yang membuka lesson yang sama.

---

## ✅ Checkpoint Modul 6

```bash
# 1. Generate Wayfinder
php artisan wayfinder:generate

# 2. Semua test harus hijau
php artisan test
# Expected: SEMUA passed

# 3. Re-seed
php artisan migrate:fresh --seed

# 4. ESLint + TypeScript
npx eslint resources/js/
npx tsc --noEmit
```

**Verifikasi visual:**

1. ✅ Buka halaman lesson → kolom **Discussion** tampil di bawah konten.
2. ✅ Ketik komentar → klik **Post Comment** → komentar muncul.
3. ✅ Form kosong setelah submit.
4. ✅ Flash message "Comment posted!" muncul.
5. ✅ (Opsional, jika Reverb jalan) Buka di 2 browser → komentar dari satu browser muncul otomatis di browser lain.


---
