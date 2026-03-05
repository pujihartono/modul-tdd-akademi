![Cover Modul TDD LMS](Cover%20Modul%20TDD%20LMS%20Laravel%2012%20%2B%20React%20Starter%20Kit.png)

# 🧪 Modul Test-Driven Development (TDD)
**IHT Rumah Coding: Akademi (LMS Laravel 12 + React Starter Kit)**

Repositori ini berisi panduan komprehensif untuk menerapkan *Test-Driven Development* (TDD) dalam ekosistem Laravel 12 tingkat lanjut. Modul ini dirancang khusus untuk membangun fondasi arsitektur sistem *Learning Management System* (LMS) yang *scalable*, dipadukan dengan *frontend* React. Fokus utamanya adalah mengubah paradigma pengembangan ke siklus **Red - Green - Refactor** untuk menghasilkan kode yang bersih, minim *bug*, dan mudah di-*maintain*.

## 🛠️ Tech Stack & Environment
Modul dan struktur kode ini dioptimalkan untuk standar pengembangan modern. Untuk pengalaman *development* terbaik, pastikan *environment* Anda selaras dengan konfigurasi berikut:

* **Backend:** Laravel 12
* **Frontend:** React (Inertia.js / API-driven)
* **Testing Framework:** Pest PHP (Direkomendasikan) / PHPUnit
* **Environment Rekomendasi:** macOS + [Laravel Herd](https://herd.laravel.com/) + MySQL

---

## 📚 Alur Pembelajaran (Course Modules)

Repositori ini dibagi menjadi 7 modul progresif. Silakan klik pada masing-masing tautan untuk mempelajari detail materi dan implementasi kodenya:

### [Modul 01: Pengenalan Proyek "Akademi" dan Persiapan Proyek](modul_01.md)
Mengenal proyek yang akan dibuat dan langkah awal mengonfigurasi *environment testing* pada Laravel 12.

### [Modul 2: Desain Data & Relasi (The Data Layer — TDD)](modul_02.md)
Memahami database dan relasi yang dibutuhkan

### [Modul 3: Membangun Katalog Kursus (Read & Frontend — TDD)](modul_03.md)
Menampilkan data kursus ke browser, diverifikasi oleh test. Hasil Visual: Halaman Katalog + Detail Kursus.

### [Modul 4: Enrollment, Flash Messages & Dashboard (TDD)](modul_04.md)
Membuat fitur enrollment kursus dengan Service Pattern, flash messages, dan dashboard. Hasil Visual: Tombol Enroll berfungsi, Flash Messages, Dashboard menampilkan kursus enrolled.

### [Modul 5: Halaman Lesson, Progress & Sertifikat (TDD)](modul_05.md)
Membuat halaman belajar lesson, tracking progress, dan sertifikat otomatis. Hasil Visual: Halaman Lesson (video + konten + navigasi), progress bar, sertifikat.

### [Modul 6: Real-time Comments (Broadcasting — TDD)](modul_06.md)
Fitur komentar di setiap lesson dengan real-time updates via WebSocket. Hasil Visual: Kolom komentar di bawah konten lesson, update tanpa refresh.

### [Modul 7: CI/CD, Code Quality & Best Practices](modul_07.md)
Tujuan: Menjaga kualitas kode dengan automated testing, linting, dan GitHub Actions.

---

## 🚀 Cara Penggunaan Repositori

1. **Clone repositori ini:**
   ```bash
   git clone [https://github.com/pujihartono/tdd-akademi.git](https://github.com/pujihartono/tdd-akademi.git)
