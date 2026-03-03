# Modul 7: CI/CD, Code Quality & Best Practices

**Tujuan**: Menjaga kualitas kode dengan automated testing, linting, dan GitHub Actions.

---

## 7.1 Test Coverage

### Setup Test Requirement
#### For Mac
Langkah 1: Install PCOV via Homebrew

```bash
brew install shivammathur/extensions/pcov@8.5
```
8.x disesuaikan dengan versi php

Edit php.ini milik Herd

```bash
extension=/opt/homebrew/opt/pcov@8.5/pcov.so
pcov.enabled=1
pcov.directory=.
```

Restart Herd
```bash
herd restart
```

#### For Windows
Download PCOV
https://pecl.php.net/package/pcov/1.0.11/windows
Pilih versi yang sesuai dengan versi PHP.
Gunakan "Thread Safe (TS)" jika ragu (biasanya Herd menggunakan versi ini).
Pilih x64 untuk Windows modern.


Pasang di Folder Herd Windows
Ekstrak file .zip yang diunduh, cari file bernama php_pcov.dll.

Pindahkan file tersebut ke folder ekstensi PHP milik Herd. Biasanya di:
%APPDATA%\Herd\config\php\<versi-php>\ext\
(Atau cari di mana folder instalasi PHP Herd berada).

**Aktifkan ekstensi di Herd**
extension=php_pcov.dll
pcov.enabled=1
pcov.directory=.

**Restart Herd**

```bash
# Jalankan semua test dengan coverage report
# --min=80 = gagal jika coverage di bawah 80%
php artisan test --coverage --min=80
```

> Atau kalau Herd Pro **Prasyarat**: Instal Xdebug atau PCOV di PHP. Jika pakai Herd: **Herd → PHP → Extensions → Xdebug → Enable**.

---

## 7.2 Parallel Testing

```bash
# Jalankan test secara paralel (lebih cepat di mesin multi-core)
php artisan test --parallel --processes=4
```

---

## 7.3 Inertia-specific Testing Patterns

```php
<?php
// Contoh-contoh assertion Inertia yang berguna:

it('passes correct props to Inertia page', function () {
    $course = Course::factory()->published()->create();

    $this->get('/courses')
        ->assertOk()
        ->assertInertia(
            fn (AssertableInertia $page) => $page
                // Pastikan component yang dirender benar
                ->component('course/index')
                // Cek jumlah item di array
                ->has('courses.data', 1)
                // Cek nilai spesifik
                ->where('courses.data.0.title', $course->title)
                // Pastikan field sensitif TIDAK ada
                ->missing('courses.data.0.instructor.email')
                ->missing('courses.data.0.instructor.password')
        );
});

it('handles Inertia validation errors', function () {
    $student = User::factory()->create(['role' => 'student']);
    $course = Course::factory()->published()->create();
    $lesson = Lesson::factory()->create(['course_id' => $course->id]);
    Enrollment::factory()->create([
        'user_id' => $student->id,
        'course_id' => $course->id,
    ]);

    // POST tanpa body → error validasi
    $this->actingAs($student)
        ->post("/courses/{$course->slug}/lessons/{$lesson->id}/comments", [
            'body' => '',
        ])
        // assertSessionHasErrors verifikasi validasi Laravel
        ->assertSessionHasErrors('body');
});
```

---

## 7.4 GitHub Actions

Starter kit sudah menyertakan workflow di `.github/workflows/`. Pastikan ada 2 file:

### `tests.yml` — Jalankan test otomatis

Buka `.github/workflows/tests.yml` (edit jika perlu):

```yaml
# Workflow ini berjalan otomatis setiap push/PR ke branch main/develop.
# Menjalankan semua test PHP untuk memastikan kode tidak rusak.

name: tests

on:
  push:
    branches:
      - develop
      - main
      - master
      - workos
  pull_request:
    branches:
      - develop
      - main
      - master
      - workos

jobs:
  ci:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        php-version: ['8.4', '8.5']

    steps:
      - name: Checkout code
        uses: actions/checkout@v6

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: ${{ matrix.php-version }}
          tools: composer:v2
          coverage: xdebug

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: '22'

      - name: Install Node Dependencies
        run: npm i

      - name: Install Dependencies
        run: composer install --no-interaction --prefer-dist --optimize-autoloader

      - name: Copy Environment File
        run: cp .env.example .env

      - name: Generate Application Key
        run: php artisan key:generate

      - name: Build Assets
        run: npm run build

      - name: Tests
        run: ./vendor/bin/pest


```

### `lint.yml` — Code quality check

```yaml
name: linter

on:
  push:
    branches:
      - develop
      - main
      - master
      - workos
  pull_request:
    branches:
      - develop
      - main
      - master
      - workos

permissions:
  contents: write

jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.4'

      - name: Install Dependencies
        run: |
          composer install -q --no-ansi --no-interaction --no-scripts --no-progress --prefer-dist
          npm install

      - name: Run Pint
        run: composer lint

      - name: Format Frontend
        run: npm run format

      - name: Lint Frontend
        run: npm run lint

      # - name: Commit Changes
      #   uses: stefanzweifel/git-auto-commit-action@v7
      #   with:
      #     commit_message: fix code style
      #     commit_options: '--no-verify'

```

---

## 7.5 Pre-commit Hooks (Opsional)

Jalankan linting otomatis sebelum setiap commit:

```bash
# Instal Husky (git hooks manager)
npm install -D husky lint-staged
npx husky init
```

Edit `.husky/pre-commit`:

```bash
npx lint-staged
```

Tambahkan di `package.json`:

```json
{
    "lint-staged": {
        "*.php": ["./vendor/bin/pint"],
        "*.{ts,tsx}": ["eslint --fix", "prettier --write"],
        "*.{css,md,json}": ["prettier --write"]
    }
}
```

---

## 7.6 PHPStorm File Watcher (Pint)

Otomatiskan format PHP setiap kali save:

1. **Settings → Tools → File Watchers** → klik **+** → pilih `<custom>`.
2. Isi:
   - **Name**: Laravel Pint
   - **File type**: PHP
   - **Program**: `$ProjectFileDir$/vendor/bin/pint`
   - **Arguments**: `$FilePath$`
   - **Output paths to refresh**: `$FilePath$`
3. Centang **Auto-save edited files to trigger the watcher**.

---

## ✅ Checkpoint Final

```bash
# === SEMUA COMMAND INI HARUS SUKSES TANPA ERROR ===

# 1. Database fresh + seed
php artisan migrate:fresh --seed

# 2. Generate Wayfinder
php artisan wayfinder:generate

# 3. PHP test (semua hijau)
php artisan test

# 4. PHP code style (optional)
./vendor/bin/pint --test

# 5. TypeScript type check (optional)
npx tsc --noEmit

# 6. ESLint (JS/TS linting)
npx eslint resources/js/

# 7. Build frontend (production build tanpa error)
npm run build
```

**Verifikasi visual end-to-end:**

1. ✅ Landing page tampil.
2. ✅ Register → login → Dashboard.
3. ✅ Katalog kursus → kartu grid.
4. ✅ Detail kursus → curriculum + sidebar.
5. ✅ Enroll → flash message hijau.
6. ✅ Lesson → video + konten + mark complete.
7. ✅ Progress bar bergerak.
8. ✅ Semua lesson selesai → sertifikat muncul.
9. ✅ Komentar bisa diposting.
10. ✅ Dashboard menampilkan enrolled courses.

---

## Rangkuman Arsitektur

```
┌──────────────────────────────────────────────────────────────┐
│                     BROWSER (React 19)                        │
│                                                              │
│  pages/              components/           types/            │
│  ├── dashboard.tsx   ├── flash-message.tsx  ├── models.d.ts  │
│  ├── course/         ├── lesson-comments.tsx                  │
│  │   ├── index.tsx   └── ui/ (shadcn)                        │
│  │   └── show.tsx                                            │
│  └── lesson/         actions/ (Wayfinder auto-generated)     │
│      └── show.tsx    └── App/Http/Controllers/*.ts           │
│                                                              │
│  ← Inertia.js (no API, just props) →                        │
│                                                              │
│                     LARAVEL 12 (PHP)                          │
│                                                              │
│  Controllers/            Services/           Resources/       │
│  ├── CourseController    EnrollUserIn         ├── UserResource│
│  ├── LessonController    CourseService        ├── Course...   │
│  ├── EnrollmentController                     └── Lesson...  │
│  ├── CommentController                                       │
│  └── DashboardController                                     │
│                                                              │
│  Models/                 Events/                              │
│  ├── User                CommentPosted (broadcast)           │
│  ├── Course                                                  │
│  ├── Lesson              Factories/                          │
│  ├── Enrollment          └── *Factory.php (testing data)     │
│  ├── LessonCompletion                                        │
│  ├── Certificate         Tests/Feature/                      │
│  └── Comment             ├── SmokeTest                       │
│                          ├── CourseModelTest                  │
│                          ├── CourseCatalogTest                │
│                          ├── EnrollmentTest                   │
│                          ├── DashboardTest                    │
│                          ├── LessonTest                       │
│                          └── CommentTest                      │
│                                                              │
│                     MySQL 8 (Database)                        │
│  ┌──────┬────────┬────────┬───────────┬──────────┬─────────┐ │
│  │users │courses │lessons │enrollments│lesson_   │certifi- │ │
│  │      │        │        │           │completions│cates   │ │
│  └──────┴────────┴────────┴───────────┴──────────┴─────────┘ │
└──────────────────────────────────────────────────────────────┘
```

---

## Daftar Semua Routes

Jalankan `php artisan route:list --compact` untuk memverifikasi:

| Method | URI | Controller | Name |
|--------|-----|------------|------|
| GET | `/` | (Welcome) | — |
| GET | `/courses` | CourseController@index | courses.index |
| GET | `/courses/{course:slug}` | CourseController@show | courses.show |
| POST | `/courses/{course:slug}/enroll` | EnrollmentController@store | courses.enroll |
| GET | `/courses/{course:slug}/lessons/{lesson}` | LessonController@show | lessons.show |
| POST | `/courses/{course:slug}/lessons/{lesson}/complete` | LessonController@complete | lessons.complete |
| POST | `/courses/{course:slug}/lessons/{lesson}/comments` | CommentController@store | comments.store |
| GET | `/dashboard` | DashboardController@index | dashboard |

---

## Selamat! 🎉

Kamu telah membangun LMS "Akademi" lengkap dengan:

- **TDD**: Setiap fitur dimulai dari test, memastikan kode terbukti benar.
- **Wayfinder**: Type-safe routing antara Laravel dan React.
- **Clean Architecture**: Service Pattern, API Resources, proper separation of concerns.
- **Real-time**: WebSocket broadcasting untuk fitur komentar.
- **Modern Stack**: Laravel 12, React 19, TypeScript, Tailwind CSS 4, shadcn/ui.
- **CI/CD**: GitHub Actions untuk automated testing dan linting.

**Langkah selanjutnya yang bisa kamu coba:**

1. **CRUD Kursus**: Halaman create/edit kursus untuk instructor.
2. **Midtrans Payment**: Integrasi pembayaran untuk kursus berbayar.
3. **File Upload**: Upload video lesson menggunakan Laravel File Storage.
4. **Search & Filter**: Pencarian kursus dengan full-text search.
5. **Admin Panel**: Dashboard admin menggunakan Filament.
