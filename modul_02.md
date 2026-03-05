
---

# Modul 2: Desain Data & Relasi (The Data Layer — TDD)

**Tujuan**: Membangun seluruh data layer (tabel, model, relasi) dengan TDD.
**Hasil Visual**: Data terverifikasi di DBeaver.

---

## 🔴 Fase RED: Test Relasi Eloquent

Kita tulis test _sebelum_ model dan migration ada. Test ini mendefinisikan **perilaku yang diharapkan**.

### Test User Model

```bash
php artisan make:test --pest Models/UserModelTest
```

Buka `tests/Feature/Models/UserModelTest.php`:

```php
<?php

// Test ini memastikan model User memiliki relasi yang benar.
// Saat dijalankan sekarang, semua GAGAL karena model Course,
// Lesson, Enrollment, Comment belum dibuat.

use App\Models\User;
use App\Models\Course;
use App\Models\Enrollment;
use App\Models\Comment;

it('has a role attribute', function () {
    // Buat user dengan role instructor menggunakan Factory
    $user = User::factory()->create(['role' => 'instructor']);

    // Pastikan atribut role tersimpan dengan benar
    expect($user->role)->toBe('instructor');
});

it('can have courses as instructor', function () {
    // Skenario: instructor memiliki kursus
    $instructor = User::factory()->create(['role' => 'instructor']);
    $course = Course::factory()->create(['instructor_id' => $instructor->id]);

    // Relasi "courses" harus mengembalikan kursus milik instructor ini
    expect($instructor->courses)->toHaveCount(1);
    expect($instructor->courses->first()->title)->toBe($course->title);
});

it('can have enrollments as student', function () {
    // Skenario: student mendaftar ke kursus
    $student = User::factory()->create(['role' => 'student']);
    $course = Course::factory()->create();

    Enrollment::factory()->create([
        'user_id' => $student->id,
        'course_id' => $course->id,
    ]);

    // Relasi "enrollments" harus mengembalikan data pendaftaran
    expect($student->enrollments)->toHaveCount(1);
});

it('can access enrolled courses via belongsToMany', function () {
    // Skenario: akses kursus via shortcut enrolledCourses
    $student = User::factory()->create(['role' => 'student']);
    $course = Course::factory()->create();

    Enrollment::factory()->create([
        'user_id' => $student->id,
        'course_id' => $course->id,
    ]);

    // Relasi belongsToMany via tabel enrollments
    expect($student->enrolledCourses)->toHaveCount(1);
    expect($student->enrolledCourses->first()->id)->toBe($course->id);
});

it('can have comments', function () {
    $user = User::factory()->create();
    $ = \App\Models\Lesson::factory()->create();

    Comment::factory()->create([
        'user_id' => $user->id,
        'lesson_id' => $lesson->id,
    ]);

    expect($user->comments)->toHaveCount(1);
});
```

### Test Course Model

```bash
php artisan make:test --pest Models/CourseModelTest
```

Buka `tests/Feature/Models/CourseModelTest.php`:

```php
<?php

use App\Models\Course;
use App\Models\User;
use App\Models\Lesson;
use App\Models\Enrollment;

it('belongs to an instructor', function () {
    // Setiap Course HARUS dimiliki oleh satu User (instructor)
    $instructor = User::factory()->create(['role' => 'instructor']);
    $course = Course::factory()->create(['instructor_id' => $instructor->id]);

    // Relasi "instructor" mengembalikan User yang membuat kursus
    expect($course->instructor->id)->toBe($instructor->id);
    expect($course->instructor->name)->toBe($instructor->name);
});

it('has many lessons', function () {
    // Satu Course bisa punya banyak Lesson
    $course = Course::factory()->create();
    Lesson::factory()->count(3)->create(['course_id' => $course->id]);

    expect($course->lessons)->toHaveCount(3);
});

it('has many enrollments', function () {
    // Satu Course bisa punya banyak pendaftar
    $course = Course::factory()->create();
    $students = User::factory()->count(2)->create(['role' => 'student']);

    foreach ($students as $student) {
        Enrollment::factory()->create([
            'user_id' => $student->id,
            'course_id' => $course->id,
        ]);
    }

    expect($course->enrollments)->toHaveCount(2);
});

it('casts price to decimal and published_at to datetime', function () {
    // Cast memastikan tipe data konsisten di PHP
    $course = Course::factory()->create([
        'price' => 49.99,
        'published_at' => '2025-06-15 10:00:00',
    ]);

    // Refresh dari database untuk memastikan cast bekerja
    $course->refresh();

    // decimal:2 → disimpan sebagai string "49.99"
    expect($course->price)->toBeString();

    // datetime → dikonversi ke Carbon (library tanggal Laravel)
    expect($course->published_at)->toBeInstanceOf(
        \Carbon\CarbonImmutable::class
    );
});
```

**Jalankan test sekarang:**

```bash
php artisan test
```

🔴 **GAGAL** — Class `App\Models\Course` not found! Ini yang kita harapkan dalam TDD.

---

## 🟢 Fase GREEN: Implementasi Minimal

### 1. Modifikasi User (tambah kolom role)

Buka `database/migrations/0001_01_01_000000_create_users_table.php`:

```php
Schema::create('users', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('email')->unique();
    $table->timestamp('email_verified_at')->nullable();
    $table->string('password');

    // ✨ TAMBAHKAN: Kolom role untuk membedakan student dan instructor
    // Default 'student' — setiap user baru otomatis jadi student
    $table->string('role')->default('student');

    $table->rememberToken();
    $table->timestamps();
});
```

Buka `app/Models/User.php`, tambahkan `role` ke `$fillable` dan relasi:

```php
// Di dalam class User, tambahkan 'role' ke array $fillable:
protected $fillable = [
    'name',
    'email',
    'password',
    'role',  // ✨ TAMBAHKAN
];

// ✨ TAMBAHKAN relasi-relasi berikut di bawah method yang sudah ada:

/**
 * Kursus yang diajar oleh instructor ini.
 * Relasi: User (instructor) hasMany Course.
 */
public function courses(): \Illuminate\Database\Eloquent\Relations\HasMany
{
    // 'instructor_id' = nama kolom foreign key di tabel courses
    return $this->hasMany(\App\Models\Course::class, 'instructor_id');
}

/**
 * Data enrollment (pendaftaran) milik user ini.
 * Relasi: User hasMany Enrollment.
 */
public function enrollments(): \Illuminate\Database\Eloquent\Relations\HasMany
{
    return $this->hasMany(\App\Models\Enrollment::class);
}

/**
 * Kursus yang di-enroll via tabel pivot enrollments.
 * Shortcut: $user->enrolledCourses (langsung dapat Course, bukan Enrollment).
 */
public function enrolledCourses(): \Illuminate\Database\Eloquent\Relations\BelongsToMany
{
    return $this->belongsToMany(\App\Models\Course::class, 'enrollments')
        ->withPivot('enrolled_at');  // Sertakan kolom tambahan dari pivot
}

/**
 * Komentar yang ditulis user ini.
 */
public function comments(): \Illuminate\Database\Eloquent\Relations\HasMany
{
    return $this->hasMany(\App\Models\Comment::class);
}

/**
 * Lesson yang sudah diselesaikan user ini.
 */
public function lessonCompletions(): \Illuminate\Database\Eloquent\Relations\HasMany
{
    return $this->hasMany(\App\Models\LessonCompletion::class);
}

/**
 * Sertifikat yang dimiliki user ini.
 */
public function certificates(): \Illuminate\Database\Eloquent\Relations\HasMany
{
    return $this->hasMany(\App\Models\Certificate::class);
}

/**
 * Instruktur di kelas terkait
 */
public function instructedCourses(): HasMany
{
    return $this->hasMany(Course::class, 'instructor_id');
}

// -------------------------------------------------------------------------
// Domain Methods
// -------------------------------------------------------------------------

/**
 * Ambil kursus yang di-enroll beserta relasi yang dibutuhkan di dashboard.
 * Query ini milik User, bukan tanggung jawab controller.
 *
 * @return Collection<int, Course>
 */
public function getEnrolledCourses(): Collection
{
    return $this->enrolledCourses()
        ->with('instructor')
        ->withCount('lessons')
        ->get();
}

/**
 * Ambil kursus yang diajar beserta statistik yang dibutuhkan di dashboard.
 *
 * @return Collection<int, Course>
 */
public function getTeachingCourses(): Collection
{
    return $this->instructedCourses()
        ->withCount(['lessons', 'enrollments'])
        ->get();
}

/**
 * Ambil ID lesson yang sudah diselesaikan user di suatu kursus.
 *
 * @return array<int>
 */
public function completedLessonIdsForCourse(Course $course): array
{
    return $this->lessonCompletions()
        ->whereIn('lesson_id', $course->lessons->pluck('id'))
        ->pluck('lesson_id')
        ->all();
}

/**
 * Ambil sertifikat user untuk kursus tertentu.
 * Null jika belum mendapat sertifikat.
 */
public function certificateFor(Course $course): ?Certificate
{
    return $this->certificates()
        ->where('course_id', $course->id)
        ->first();
}
```

Update `database/factories/UserFactory.php` — tambahkan `role`:

```php
public function definition(): array
{
    return [
        'name' => fake()->name(),
        'email' => fake()->unique()->safeEmail(),
        'email_verified_at' => now(),
        'password' => static::$password ??= Hash::make('password'),
        'role' => 'student',  // ✨ Default role untuk Factory
        'remember_token' => Str::random(10),
    ];
}
```

### 2. Course (Model + Migration + Factory)

```bash
# -mf = buat Migration dan Factory sekaligus
php artisan make:model Course -mf
```

**Migration** `database/migrations/xxxx_create_courses_table.php`:

```php
public function up(): void
{
    Schema::create('courses', function (Blueprint $table) {
        $table->id();

        // Foreign key ke tabel users (instructor yang membuat kursus)
        // cascadeOnDelete = jika user dihapus, kursusnya ikut terhapus
        $table->foreignId('instructor_id')
            ->constrained('users')
            ->cascadeOnDelete();

        $table->string('title');                      // Judul kursus
        $table->string('slug')->unique();             // URL-friendly: "laravel-12-masterclass"
        $table->text('description')->nullable();       // Deskripsi (opsional)
        $table->decimal('price', 10, 2)->default(0);  // Harga (10 digit, 2 desimal)
        $table->timestamp('published_at')->nullable(); // null = draft, ada tanggal = published
        $table->timestamps();                          // created_at & updated_at
    });
}
```

**Model** `app/Models/Course.php`:

```php
<?php

namespace App\Models;

use Database\Factories\CourseFactory;
use Illuminate\Contracts\Pagination\LengthAwarePaginator;
use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;

class Course extends Model
{
    /** @use HasFactory<CourseFactory> */
    use HasFactory;

    // Kolom yang boleh diisi via mass assignment (create/update)
    protected $fillable = [
        'instructor_id',
        'title',
        'slug',
        'description',
        'price',
        'published_at',
    ];

    /**
     * Cast: konversi tipe data otomatis saat baca/tulis dari DB.
     * - price → selalu string format "49.99" (bukan float yang bisa rounding error)
     * - published_at → Carbon object (bisa diformat tanggal dengan mudah)
     */
    protected function casts(): array
    {
        return [
            'price' => 'decimal:2',
            'published_at' => 'datetime',
        ];
    }

    /** Instructor yang membuat kursus ini. */
    public function instructor(): BelongsTo
    {
        return $this->belongsTo(User::class, 'instructor_id');
    }

    /** Daftar lesson di kursus ini. */
    public function lessons(): HasMany
    {
        return $this->hasMany(Lesson::class);
    }

    /** Daftar enrollment (pendaftaran) di kursus ini. */
    public function enrollments(): HasMany
    {
        return $this->hasMany(Enrollment::class);
    }

    public function certificates(): HasMany
    {
        return $this->hasMany(Certificate::class);
    }

    // -------------------------------------------------------------------------
    // Local Scopes
    // -------------------------------------------------------------------------

    /**
     * Hanya kursus yang sudah dipublish.
     */
    public function scopeWherePublished(Builder $query): void
    {
        $query->whereNotNull('published_at');
    }

    // -------------------------------------------------------------------------
    // Static Query Methods
    // -------------------------------------------------------------------------

    /**
     * Ambil daftar kursus published untuk halaman katalog.
     * Query ini tidak mengandung business rule, hanya presentasi data.
     */
    public static function getPublished(int $perPage = 9): LengthAwarePaginator
    {
        return static::query()
            ->with('instructor')
            ->withCount('lessons')
            ->wherePublished()
            ->latest('published_at')
            ->paginate($perPage);
    }

    // -------------------------------------------------------------------------
    // Helpers
    // -------------------------------------------------------------------------

    public function isPublished(): bool
    {
        return $this->published_at !== null;
    }
}
```

**Factory** `database/factories/CourseFactory.php`:

```php
<?php

namespace Database\Factories;

use App\Models\User;
use Illuminate\Database\Eloquent\Factories\Factory;
use Illuminate\Support\Str;

class CourseFactory extends Factory
{
    public function definition(): array
    {
        $title = fake()->sentence();

        return [
            // Jika instructor_id tidak diberikan, buat user instructor baru
            'instructor_id' => User::factory()->state(['role' => 'instructor']),
            'title' => $title,
            // Slug = judul yang di-lowercase dan spasi diganti strip
            'slug' => Str::slug($title).'-'.fake()->unique()->randomNumber(5),
            'description' => fake()->paragraph(),
            'price' => fake()->randomFloat(2, 10, 100),
            'published_at' => null,  // Default: belum dipublish (draft)
        ];
    }

    /**
     * Factory State: kursus yang sudah dipublish.
     * Penggunaan: Course::factory()->published()->create()
     */
    public function published(): static
    {
        return $this->state(fn () => ['published_at' => now()]);
    }

    /** Factory State: kursus yang belum dipublish (draft). */
    public function unpublished(): static
    {
        return $this->state(fn () => ['published_at' => null]);
    }

    /** Factory State: kursus gratis (harga 0). */
    public function free(): static
    {
        return $this->state(fn () => ['price' => 0]);
    }
}
```

### 3. Lesson

```bash
php artisan make:model Lesson -mf
```

**Migration:**

```php
public function up(): void
{
    Schema::create('lessons', function (Blueprint $table) {
        $table->id();
        $table->foreignId('course_id')->constrained()->cascadeOnDelete();
        $table->string('title');                   // Judul lesson
        $table->string('video_url')->nullable();   // URL video (YouTube, Vimeo, dll)
        $table->longText('content')->nullable();   // Konten teks/artikel
        $table->unsignedTinyInteger('order')->default(1);
        $table->timestamps();
    });
}
```

**Model** `app/Models/Lesson.php`:

```php
<?php

namespace App\Models;

use Database\Factories\LessonFactory;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;

class Lesson extends Model
{
    /** @use HasFactory<LessonFactory> */
    use HasFactory;

    protected $fillable = ['course_id', 'title', 'video_url', 'content', 'order'];

    /** Kursus yang memiliki lesson ini. */
    public function course(): BelongsTo
    {
        return $this->belongsTo(Course::class);
    }

    /** Komentar di lesson ini. */
    public function comments(): HasMany
    {
        return $this->hasMany(Comment::class);
    }

    /** Lessom yang yang sudah selesai */
    public function completions(): HasMany
    {
        return $this->hasMany(LessonCompletion::class);
    }

    // -------------------------------------------------------------------------
    // Domain Methods
    // -------------------------------------------------------------------------

    /**
     * Ambil lesson sebelumnya dalam kursus (berdasarkan order).
     * Query ini berkaitan langsung dengan Lesson, tepat berada di model.
     */
    public function previousLesson(): ?self
    {
        return $this->course->lessons
            ->sortBy('order')
            ->values()
            ->filter(fn (self $l) => $l->order < $this->order)
            ->last();
    }

    /**
     * Ambil lesson berikutnya dalam kursus (berdasarkan order).
     */
    public function nextLesson(): ?self
    {
        return $this->course->lessons
            ->sortBy('order')
            ->values()
            ->filter(fn (self $l) => $l->order > $this->order)
            ->first();
    }

    /**
     * Load komentar lesson beserta relasi user-nya (untuk ditampilkan di halaman lesson).
     */
    public function loadComments(): self
    {
        return $this->load(['comments' => fn ($query) => $query->oldest()->with('user:id,name,role')]);
    }
}
```

**Factory** `database/factories/LessonFactory.php`:

```php
<?php

namespace Database\Factories;

use App\Models\Course;
use App\Models\Lesson;
use Illuminate\Database\Eloquent\Factories\Factory;

/**
 * @extends Factory<Lesson>
 */
class LessonFactory extends Factory
{
    /**
     * Define the model's default state.
     *
     * @return array<string, mixed>
     */
    public function definition(): array
    {
        return [
            'course_id' => Course::factory(), // Auto-buat Course jika tidak disediakan
            'title' => fake()->sentence(),
            'video_url' => 'https://example.com/video/'.fake()->uuid(),
            'content' => fake()->paragraphs(3, true), // 3 paragraf digabung jadi string
            'order' => $this->faker->numberBetween(1, 100),
        ];
    }
}
```

### 4. Enrollment

```bash
php artisan make:model Enrollment -mf
```

**Migration:**

```php
public function up(): void
{
    Schema::create('enrollments', function (Blueprint $table) {
        $table->id();
        $table->foreignId('user_id')->constrained()->cascadeOnDelete();
        $table->foreignId('course_id')->constrained()->cascadeOnDelete();
        $table->timestamp('enrolled_at')->useCurrent(); // Otomatis isi waktu saat insert
        $table->timestamp('completed_at')->nullable();   // Diisi saat semua lesson selesai
        $table->timestamps();

        // CONSTRAINT: satu siswa hanya bisa enroll sekali ke satu kursus
        $table->unique(['user_id', 'course_id']);
    });
}
```

**Model** `app/Models/Enrollment.php`:

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class Enrollment extends Model
{
    use HasFactory;

    protected $fillable = ['user_id', 'course_id', 'enrolled_at', 'completed_at'];

    protected function casts(): array
    {
        return [
            'enrolled_at' => 'datetime',
            'completed_at' => 'datetime',
        ];
    }

    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }

    public function course(): BelongsTo
    {
        return $this->belongsTo(Course::class);
    }
}
```

**Factory** `database/factories/EnrollmentFactory.php`:

```php
<?php

namespace Database\Factories;

use App\Models\Course;
use App\Models\User;
use Illuminate\Database\Eloquent\Factories\Factory;

class EnrollmentFactory extends Factory
{
    public function definition(): array
    {
        return [
            'user_id' => User::factory(),
            'course_id' => Course::factory(),
        ];
    }
}
```

### 5. Comment

```bash
php artisan make:model Comment -mf
```

**Migration:**

```php
public function up(): void
{
    Schema::create('comments', function (Blueprint $table) {
        $table->id();
        $table->foreignId('user_id')->constrained()->cascadeOnDelete();
        $table->foreignId('lesson_id')->constrained()->cascadeOnDelete();
        $table->text('body');  // Isi komentar
        $table->timestamps();
    });
}
```

**Model** `app/Models/Comment.php`:

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class Comment extends Model
{
    use HasFactory;

    protected $fillable = ['user_id', 'lesson_id', 'body'];

    /** User yang menulis komentar ini. */
    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }

    /** Lesson tempat komentar ini berada. */
    public function lesson(): BelongsTo
    {
        return $this->belongsTo(Lesson::class);
    }
}
```

**Factory** `database/factories/CommentFactory.php`:

```php
<?php

namespace Database\Factories;

use App\Models\Lesson;
use App\Models\User;
use Illuminate\Database\Eloquent\Factories\Factory;

class CommentFactory extends Factory
{
    public function definition(): array
    {
        return [
            'user_id' => User::factory(),
            'lesson_id' => Lesson::factory(),
            'body' => fake()->paragraph(),
        ];
    }
}
```

### 6. LessonCompletion & Certificate

Kita buat sekarang agar tidak perlu menambah migration di modul selanjutnya:

```bash
php artisan make:model LessonCompletion -mf
php artisan make:model Certificate -mf
```

**Migration `lesson_completions`:**

```php
public function up(): void
{
    Schema::create('lesson_completions', function (Blueprint $table) {
        $table->id();
        $table->foreignId('user_id')->constrained()->cascadeOnDelete();
        $table->foreignId('lesson_id')->constrained()->cascadeOnDelete();
        $table->timestamps();
        // CONSTRAINT: satu user hanya bisa menyelesaikan satu lesson sekali
        $table->unique(['user_id', 'lesson_id']);
    });
}
```

**Model** `app/Models/LessonCompletion.php`:

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class LessonCompletion extends Model
{
    use HasFactory;

    protected $fillable = ['user_id', 'lesson_id'];

    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }

    public function lesson(): BelongsTo
    {
        return $this->belongsTo(Lesson::class);
    }
}
```

**Factory** `database/factories/LessonCompletionFactory.php`:

```php
<?php

namespace Database\Factories;

use Illuminate\Database\Eloquent\Factories\Factory;

class LessonCompletionFactory extends Factory
{
    public function definition(): array
    {
        return [
            'user_id' => \App\Models\User::factory(),
            'lesson_id' => \App\Models\Lesson::factory(),
        ];
    }
}
```

**Migration `certificates`:**

```php
public function up(): void
{
    Schema::create('certificates', function (Blueprint $table) {
        $table->id();
        $table->foreignId('user_id')->constrained()->cascadeOnDelete();
        $table->foreignId('course_id')->constrained()->cascadeOnDelete();
        // Nomor sertifikat unik, contoh: "ACAD-AB12CD34"
        $table->string('certificate_number')->unique();
        $table->timestamp('issued_at')->default(now());
        $table->timestamps();
        // CONSTRAINT: satu sertifikat per user per course
        $table->unique(['user_id', 'course_id']);
    });
}
```

**Model** `app/Models/Certificate.php`:

```php
<?php

namespace App\Models;

use Database\Factories\CertificateFactory;
use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class Certificate extends Model
{
    /** @use HasFactory<CertificateFactory> */
    use HasFactory;

    protected $fillable = [
        'user_id',
        'course_id',
        'certificate_number',
        'issued_at',
    ];

    public function casts(): array
    {
        return [
            'issued_at' => 'datetime',
        ];
    }

    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }

    public function course(): BelongsTo
    {
        return $this->belongsTo(Course::class);
    }

    /**
     * Filter sertifikat berdasarkan user.
     */
    public function scopeForUser(Builder $query, int $userId): void
    {
        $query->where('user_id', $userId);
    }

    /**
     * Filter sertifikat berdasarkan kursus.
     */
    public function scopeForCourse(Builder $query, int $courseId): void
    {
        $query->where('course_id', $courseId);
    }
}
```

**Factory** `database/factories/CertificateFactory.php`:

```php
<?php

namespace Database\Factories;

use App\Models\Certificate;
use App\Models\Course;
use App\Models\User;
use Illuminate\Database\Eloquent\Factories\Factory;
use Illuminate\Support\Str;

/**
 * @extends Factory<Certificate>
 */
class CertificateFactory extends Factory
{
    /**
     * Define the model's default state.
     *
     * @return array<string, mixed>
     */
    public function definition(): array
    {
        return [
            'user_id' => User::factory(),
            'course_id' => Course::factory(),
            'certificate_number' => 'ACAD-'.Str::upper(Str::random(8)),
            'issued_at' => now(),
        ];
    }
}
```

**Jalankan test:**

```bash
php artisan test
```

🟢 **SEMUA HIJAU!**

---

## 🔵 Fase REFACTOR: Seeder + Verifikasi Visual

Buka `database/seeders/DatabaseSeeder.php`:

```php
<?php

namespace Database\Seeders;

use App\Models\Course;
use App\Models\Enrollment;
use App\Models\Lesson;
use App\Models\User;
use Illuminate\Database\Console\Seeds\WithoutModelEvents;
use Illuminate\Database\Seeder;

class DatabaseSeeder extends Seeder
{
    /**
     * Seed the application's database.
     */
    use WithoutModelEvents; // Nonaktifkan events saat seeding (lebih cepat)

    public function run(): void
    {
        // === 1. Buat Instruktur ===
        $instructor = User::create([
            'name' => 'Rumah Coding',
            'email' => 'admin@rumahcoding.co.id',
            'password' => bcrypt('password'),
            'role' => 'instructor',
        ]);

        // === 2. Buat 5 Siswa ===
        $students = User::factory()->count(5)->create([
            'role' => 'student',
            'password' => bcrypt('password'), // Semua pakai password "password"
        ]);

        // === 3. Buat 2 Kursus (published) ===
        $course1 = Course::create([
            'instructor_id' => $instructor->id,
            'title' => 'Laravel 12 Masterclass',
            'slug' => 'laravel-12-masterclass',
            'description' => 'Learn Laravel 12 from scratch to advanced level.',
            'price' => 49.99,
            'published_at' => now(), // Sudah dipublish
        ]);

        $course2 = Course::create([
            'instructor_id' => $instructor->id,
            'title' => 'React & Inertia JS',
            'slug' => 'react-inertia-js',
            'description' => 'Build modern SPAs with React, Inertia and Laravel.',
            'price' => 59.99,
            'published_at' => now(),
        ]);

        // === 4. Buat Lessons ===
        Lesson::create([
            'course_id' => $course1->id,
            'title' => 'Introduction to Laravel 12',
            'video_url' => 'https://www.youtube.com/embed/dQw4w9WgXcQ',
            'content' => 'Overview of Laravel 12 features and what we will build.',
            'order' => 1,
        ]);
        Lesson::create([
            'course_id' => $course1->id,
            'title' => 'Routing and Controllers',
            'video_url' => 'https://www.youtube.com/embed/dQw4w9WgXcQ',
            'content' => 'Deep dive into routing, named routes, and resource controllers.',
            'order' => 2,
        ]);
        Lesson::create([
            'course_id' => $course2->id,
            'title' => 'React Fundamentals',
            'video_url' => 'https://www.youtube.com/embed/dQw4w9WgXcQ',
            'content' => 'JSX, components, props, and hooks.',
            'order' => 1,
        ]);
        Lesson::create([
            'course_id' => $course2->id,
            'title' => 'Inertia.js Deep Dive',
            'video_url' => 'https://www.youtube.com/embed/dQw4w9WgXcQ',
            'content' => 'How Inertia bridges Laravel and React.',
            'order' => 2,
        ]);

        // === 5. Enroll semua siswa ke kursus pertama ===
        foreach ($students as $student) {
            Enrollment::create([
                'user_id' => $student->id,
                'course_id' => $course1->id,
            ]);
        }
    }
}
```

```bash
# Reset database + jalankan seeder
php artisan migrate:fresh --seed
```

### Verifikasi di DBeaver

Buka DBeaver → klik kanan database `akademi` → **Refresh** → buka setiap tabel:

| Tabel | Jumlah Record | Verifikasi |
|-------|:------------:|------------|
| `users` | 6 | 1 instructor + 5 students |
| `courses` | 2 | Laravel 12 Masterclass + React & Inertia |
| `lessons` | 4 | 2 per kursus |
| `enrollments` | 5 | Semua student enrolled di kursus pertama |
| `lesson_completions` | 0 | Kosong (belum ada yang selesai) |
| `certificates` | 0 | Kosong (belum ada sertifikat) |
| `comments` | 0 | Kosong (belum ada komentar) |

---

## ✅ Checkpoint Modul 2

```bash
# 1. Semua test harus hijau
php artisan test
# Expected: 9+ passed (SmokeTest + UserModelTest + CourseModelTest)

# 2. Database harus bisa di-seed tanpa error
php artisan migrate:fresh --seed
# Expected: "Seeding database" tanpa error

# 3. Aplikasi masih berjalan normal
# Buka http://akademi.test → login masih bekerja
```
