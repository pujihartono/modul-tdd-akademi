# Modul: Membangun LMS "Akademi" dengan TDD — Laravel 12 + React Starter Kit

**Target Peserta**: Pemula yang sudah memahami dasar PHP, HTML/CSS, dan JavaScript.
**Prasyarat**: Laptop/PC modern, koneksi internet untuk instalasi awal.
**Durasi Estimasi**: 4 hari.

---

## Pengenalan Proyek "Akademi"

### Apa itu Akademi?

Akademi adalah **Learning Management System (LMS)** — platform belajar online seperti Udemy, Coursera, atau Ruangguru. Kita akan membangunnya dari nol menggunakan metodologi **Test-Driven Development (TDD)**, di mana setiap fitur _dibuktikan benar oleh test_ sebelum dibangun.

### Apa yang Akan Kita Bangun?

**Sebagai Siswa (Student), kamu bisa:**

- Melihat katalog kursus yang tersedia.
- Mendaftar (enroll) ke kursus.
- Menonton video dan membaca materi setiap lesson.
- Menandai lesson sebagai "selesai" dan melihat progress.
- Mendapat sertifikat otomatis setelah semua lesson selesai.
- Berdiskusi real-time dengan sesama siswa di setiap lesson.

**Sebagai Instruktur (Instructor), kamu bisa:**

- Membuat kursus dan menambah lesson.
- Melihat siapa saja yang mendaftar di kursusmu.

### Teknologi yang Digunakan

| Komponen | Teknologi | Fungsi |
|----------|-----------|--------|
| **Backend** | Laravel 12 (PHP 8.5+) | Framework utama, routing, database, autentikasi |
| **Frontend** | React 19 + TypeScript | User Interface (UI) interaktif |
| **Bridge** | Inertia.js 2 | Menghubungkan Laravel & React tanpa API terpisah |
| **Styling** | Tailwind CSS 4 | Utility-first CSS framework |
| **Components** | shadcn/ui + Radix UI | Komponen UI siap pakai (Button, Card, dll) |
| **Routing** | Laravel Wayfinder | Type-safe routing: URL di-generate otomatis dari controller |
| **Testing** | Pest PHP | Testing framework untuk PHP |
| **Real-time** | Laravel Reverb | WebSocket server untuk fitur real-time |
| **Database** | MySQL 8 | Penyimpanan data |

### Apa itu TDD?

TDD (Test-Driven Development) adalah cara kerja di mana kita menulis **test dulu** sebelum menulis kode:

1. 🔴 **RED** — Tulis test untuk fitur yang belum ada → test **pasti gagal**.
2. 🟢 **GREEN** — Tulis kode _minimal_ agar test **lulus**.
3. 🔵 **REFACTOR** — Perbaiki kualitas kode + bangun UI. Test menjadi **jaring pengaman**.

**Mengapa TDD?** Karena kamu mendapat _bukti_ bahwa kodemu benar, bukan hanya asumsi.

### Apa itu Wayfinder?

Wayfinder adalah package Laravel yang men-generate fungsi TypeScript dari route dan controller PHP kamu secara otomatis. Tanpa Wayfinder, kamu harus menulis URL secara manual di React:

```tsx
// ❌ TANPA Wayfinder — URL ditulis manual, rawan typo
<Link href="/courses/laravel-12-masterclass">Detail</Link>
```

Dengan Wayfinder, URL di-generate dari PHP:

```tsx
// ✅ DENGAN Wayfinder — type-safe, auto-update saat route berubah
import { show } from '@/actions/App/Http/Controllers/CourseController';

<Link href={show.url({ course: 'laravel-12-masterclass' })}>Detail</Link>
```

Jika kamu mengganti nama route di PHP, TypeScript akan **error saat compile** — bukan error di production!

---

## Analisis Kebutuhan & Desain Database

### Daftar Entitas

Berdasarkan fitur di atas, kita membutuhkan 6 entitas (tabel):

| No | Entitas | Deskripsi |
|----|---------|-----------|
| 1 | **User** | Pengguna sistem. Bisa berperan sebagai `student` atau `instructor`. |
| 2 | **Course** | Kursus/mata pelajaran. Dimiliki oleh satu Instructor. |
| 3 | **Lesson** | Pelajaran/materi di dalam Course. Berisi video dan konten teks. |
| 4 | **Enrollment** | Catatan pendaftaran siswa ke kursus (relasi many-to-many). |
| 5 | **LessonCompletion** | Catatan lesson mana saja yang sudah diselesaikan siswa. |
| 6 | **Certificate** | Sertifikat otomatis saat semua lesson di satu kursus selesai. |
| 7 | **Comment** | Komentar/diskusi di setiap lesson (real-time). |

### Entity Relationship Diagram (ERD) — Teks

```
┌──────────────────┐
│      USERS       │
├──────────────────┤       ┌──────────────────┐
│ id (PK)          │       │     COURSES      │
│ name             │       ├──────────────────┤
│ email (unique)   │──1──┐ │ id (PK)          │
│ password         │     │ │ instructor_id(FK)│──── Users.id
│ role             │     │ │ title            │
│ remember_token   │     │ │ slug (unique)    │
│ timestamps       │     │ │ description      │
└──────────────────┘     │ │ price            │
        │                │ │ published_at     │
        │                │ │ timestamps       │
        │                │ └──────────────────┘
        │                │          │
        │                │          │ 1
        │                │          │
        │                │          ▼ *
        │                │ ┌──────────────────┐
        │                │ │     LESSONS      │
        │                │ ├──────────────────┤
        │                │ │ id (PK)          │
        │                │ │ course_id (FK)   │──── Courses.id
        │                │ │ title            │
        │                │ │ video_url        │
        │                │ │ content          │
        │                │ │ order            │
        │                │ │ timestamps       │
        │                │ └──────────────────┘
        │                │          │
        │                │          │
        ▼ *              │          ▼ *
┌──────────────────┐     │ ┌──────────────────────┐
│   ENROLLMENTS    │     │ │  LESSON_COMPLETIONS  │
├──────────────────┤     │ ├──────────────────────┤
│ id (PK)          │     │ │ id (PK)              │
│ user_id (FK)     │─────┘ │ user_id (FK)         │──── Users.id
│ course_id (FK)   │───┐   │ lesson_id (FK)       │──── Lessons.id
│ enrolled_at      │   │   │ timestamps           │
│ completed_at     │   │   │ unique(user,lesson)  │
│ timestamps       │   │   └──────────────────────┘
│ unique(user,co.) │   │
└──────────────────┘   │   ┌──────────────────┐
                       │   │   CERTIFICATES   │
                       │   ├──────────────────┤
                       │   │ id (PK)          │
                       └──▶│ course_id (FK)   │──── Courses.id
                           │ user_id (FK)     │──── Users.id
                           │ certificate_no   │
                           │ issued_at        │
                           │ timestamps       │
                           │ unique(user,co.) │
                           └──────────────────┘
        │
        ▼ *
┌──────────────────┐
│    COMMENTS      │
├──────────────────┤
│ id (PK)          │
│ user_id (FK)     │──── Users.id
│ lesson_id (FK)   │──── Lessons.id
│ body             │
│ timestamps       │
└──────────────────┘
```

### Penjelasan Relasi

| Relasi | Tipe | Penjelasan |
|--------|------|------------|
| User → Course | **One-to-Many** | Satu instructor bisa membuat banyak kursus |
| Course → Lesson | **One-to-Many** | Satu kursus punya banyak lesson |
| User ↔ Course (via Enrollment) | **Many-to-Many** | Satu siswa bisa enroll ke banyak kursus, satu kursus bisa punya banyak siswa |
| User → LessonCompletion | **One-to-Many** | Satu siswa bisa menyelesaikan banyak lesson |
| Lesson → LessonCompletion | **One-to-Many** | Satu lesson bisa diselesaikan banyak siswa |
| User → Certificate | **One-to-Many** | Satu siswa bisa punya banyak sertifikat (satu per kursus) |
| Course → Certificate | **One-to-Many** | Satu kursus bisa punya banyak sertifikat (satu per siswa) |
| User → Comment | **One-to-Many** | Satu user bisa menulis banyak komentar |
| Lesson → Comment | **One-to-Many** | Satu lesson bisa punya banyak komentar |

### Constraint Penting

- **`enrollments`**: Kombinasi `user_id` + `course_id` harus **unique** — siswa tidak boleh enroll 2x ke kursus yang sama.
- **`lesson_completions`**: Kombinasi `user_id` + `lesson_id` harus **unique** — idempotent (menandai 2x tidak membuat duplikat).
- **`certificates`**: Kombinasi `user_id` + `course_id` harus **unique** — satu sertifikat per kursus per siswa.
- **`courses.slug`**: Harus **unique** — digunakan sebagai URL-friendly identifier.
- **`cascadeOnDelete`**: Jika Course dihapus, semua Lesson, Enrollment, dan Certificate terkait ikut terhapus.

### Alur Fitur Utama

```
[Guest]
  │
  ├── Lihat Katalog (/courses)
  │     └── Lihat Detail Kursus (/courses/{slug})
  │           └── "Login to Enroll" → Redirect ke /login
  │
[Student - setelah login]
  │
  ├── Dashboard (/dashboard)
  │     └── Daftar kursus yang di-enroll
  │
  ├── Enroll ke Kursus (POST /courses/{slug}/enroll)
  │     └── Redirect kembali + Flash Message "Berhasil!"
  │
  ├── Belajar Lesson (/courses/{slug}/lessons/{id})
  │     ├── Tonton video
  │     ├── Baca konten
  │     ├── "Mark as Complete" (POST)
  │     ├── Diskusi real-time (POST comment + WebSocket)
  │     └── Navigasi Prev/Next lesson
  │
  └── Sertifikat (otomatis saat semua lesson selesai)
```

---

## Daftar Isi Modul

| Modul | Topik | Hasil Visual |
|-------|-------|-------------|
| **1** | Setup Ekosistem & Test Framework | Sidebar menu + halaman placeholder |
| **2** | Data Layer TDD (Models, Migrations) | Verifikasi data di DBeaver |
| **3** | Katalog Kursus (Read & Frontend) | Halaman Katalog + Detail Kursus |
| **4** | Enrollment & Clean Architecture | Flash Messages, Dashboard, Tombol Enroll |
| **5** | Progress Belajar & Sertifikat | Halaman Lesson, Progress Bar, Certificate |
| **6** | Real-time WebSockets | Komponen komentar di halaman Lesson |
| **7** | CI/CD & Best Practices | GitHub Actions pipeline |

> **Penting**: Setiap modul diakhiri dengan **Checkpoint** — serangkaian perintah dan verifikasi untuk memastikan semua kode berjalan sebelum lanjut ke modul berikutnya.

---

# Modul 00: Dasar PHP & React untuk Proyek Laravel + React Starter Kit

> **Tujuan**: Memahami konsep PHP Dasar, PHP OOP, TypeScript, dan React sebagai fondasi sebelum masuk ke Laravel + React.
>
> **Untuk siapa**: Pemula yang baru belajar web development.
>
> **Catatan**: Modul ini adalah **pengantar ringkas** — fokus pada konsep yang paling sering dipakai, bukan semua fitur bahasa.

---

## Daftar Isi

1. [PHP Dasar](#1-php-dasar)
2. [PHP OOP (Object-Oriented Programming)](#2-php-oop)
3. [TypeScript Dasar](#3-typescript-dasar)
4. [React Dasar](#4-react-dasar)

---

## 1. PHP Dasar

PHP adalah bahasa pemrograman yang berjalan di **server**. Laravel dibangun di atas PHP, jadi memahami PHP dasar adalah wajib.

### 1.1 Variabel & Tipe Data

Variabel PHP selalu diawali dengan tanda `$`. PHP bersifat *dynamically typed* — tipe data ditentukan otomatis.

```php
<?php

$nama   = "Budi";           // string
$umur   = 25;               // integer
$tinggi = 170.5;            // float
$aktif  = true;             // boolean
$kosong = null;             // null

echo "Halo, nama saya $nama dan umur saya $umur tahun.";
// Output: Halo, nama saya Budi dan umur saya 25 tahun.
```

### 1.2 Array

Array di PHP bisa berupa **list** (indexed) atau **map** (associative).

```php
<?php

// Array indexed (seperti list)
$buah = ["apel", "mangga", "jeruk"];
echo $buah[0]; // apel

// Array associative (seperti kamus/object)
$user = [
    "nama"  => "Siti",
    "email" => "siti@email.com",
    "umur"  => 22,
];
echo $user["nama"]; // Siti

// Array of arrays (sering dipakai di Laravel untuk data koleksi)
$siswa = [
    ["nama" => "Ali",  "nilai" => 90],
    ["nama" => "Budi", "nilai" => 85],
];
echo $siswa[1]["nama"]; // Budi
```

### 1.3 Kondisi (if / match)

```php
<?php

$nilai = 75;

// if - else
if ($nilai >= 80) {
    echo "Lulus dengan baik";
} elseif ($nilai >= 60) {
    echo "Lulus";
} else {
    echo "Tidak lulus";
}

// match (PHP 8+) — lebih ringkas dari switch
$status = match(true) {
    $nilai >= 80 => "A",
    $nilai >= 60 => "B",
    default      => "C",
};
echo $status; // B
```

### 1.4 Perulangan (Loop)

```php
<?php

$angka = [1, 2, 3, 4, 5];

// foreach — paling sering dipakai di Laravel (looping koleksi/array)
foreach ($angka as $n) {
    echo $n . " ";
}
// Output: 1 2 3 4 5

// foreach dengan key (berguna untuk array associative)
$produk = ["nama" => "Buku", "harga" => 25000];
foreach ($produk as $key => $value) {
    echo "$key: $value\n";
}
// nama: Buku
// harga: 25000
```

### 1.5 Function

```php
<?php

// Function biasa
function sapa(string $nama): string
{
    return "Halo, $nama!";
}

echo sapa("Dewi"); // Halo, Dewi!

// Arrow function (closure ringkas, sering dipakai di Laravel collection)
$kali2 = fn($x) => $x * 2;
echo $kali2(5); // 10

// Default parameter
function greet(string $nama, string $sapaan = "Halo"): string
{
    return "$sapaan, $nama!";
}
echo greet("Andi");           // Halo, Andi!
echo greet("Andi", "Selamat datang"); // Selamat datang, Andi!
```

### 1.6 String Penting

```php
<?php

$nama  = "  Laravel  ";
$email = "user@example.com";

// Fungsi string yang sering dipakai
echo strtolower($nama);          // "  laravel  "
echo strtoupper($nama);          // "  LARAVEL  "
echo trim($nama);                // "Laravel"
echo strlen(trim($nama));        // 7
echo str_contains($email, "@");  // true (PHP 8+)
echo str_replace("@", " [at] ", $email); // user [at] example.com

// Heredoc — untuk string panjang/multiline
$pesan = <<<EOT
Halo $nama,
Selamat datang di sistem kami.
EOT;
```

---

## 2. PHP OOP

Laravel **sangat bergantung** pada OOP. Semua yang ada di Laravel — Controller, Model, Request, dll — adalah **class**.

### 2.1 Class & Object

```php
<?php

class Produk
{
    // Properties (variabel milik class)
    public string $nama;
    public int    $harga;

    // Constructor — dijalankan saat object dibuat
    public function __construct(string $nama, int $harga)
    {
        $this->nama  = $nama;
        $this->harga = $harga;
    }

    // Method (fungsi milik class)
    public function info(): string
    {
        return "{$this->nama} seharga Rp " . number_format($this->harga);
    }
}

// Membuat object dari class
$buku = new Produk("Buku Laravel", 85000);
echo $buku->info(); // Buku Laravel seharga Rp 85.000
```

### 2.2 Visibility (public, protected, private)

```php
<?php

class BankAccount
{
    public    string $pemilik;   // bisa diakses dari mana saja
    protected float  $saldo;     // hanya dari class ini & turunannya
    private   string $pin;       // hanya dari class ini saja

    public function __construct(string $pemilik, float $saldo, string $pin)
    {
        $this->pemilik = $pemilik;
        $this->saldo   = $saldo;
        $this->pin     = $pin;
    }

    public function getSaldo(): float
    {
        return $this->saldo; // method publik untuk akses saldo
    }
}

$akun = new BankAccount("Rini", 1000000, "1234");
echo $akun->pemilik;    // Rini ✅
echo $akun->getSaldo(); // 1000000 ✅
// echo $akun->pin;     // ❌ Error! private
```

### 2.3 Inheritance (Pewarisan)

Sebuah class bisa **mewarisi** properti dan method dari class lain. Di Laravel, Controller mewarisi `BaseController`, Model mewarisi `Eloquent\Model`, dll.

```php
<?php

// Class induk
class Hewan
{
    public string $nama;

    public function __construct(string $nama)
    {
        $this->nama = $nama;
    }

    public function bernafas(): string
    {
        return "{$this->nama} sedang bernafas.";
    }
}

// Class anak — mewarisi Hewan
class Kucing extends Hewan
{
    public function bersuara(): string
    {
        return "{$this->nama} berkata: Meong!";
    }
}

$kucingku = new Kucing("Mimi");
echo $kucingku->bernafas(); // Mimi sedang bernafas. (dari Hewan)
echo $kucingku->bersuara(); // Mimi berkata: Meong!  (milik Kucing)
```

### 2.4 Interface & Abstract Class

**Interface** adalah kontrak: "class yang implement interface ini *wajib* punya method berikut."

```php
<?php

// Interface — mendefinisikan "kontrak"
interface Pembayaran
{
    public function bayar(int $jumlah): string;
    public function refund(int $jumlah): string;
}

// Class yang mengimplementasi interface
class Midtrans implements Pembayaran
{
    public function bayar(int $jumlah): string
    {
        return "Bayar Rp $jumlah via Midtrans berhasil.";
    }

    public function refund(int $jumlah): string
    {
        return "Refund Rp $jumlah via Midtrans berhasil.";
    }
}

class Stripe implements Pembayaran
{
    public function bayar(int $jumlah): string
    {
        return "Pay \$$jumlah via Stripe success.";
    }

    public function refund(int $jumlah): string
    {
        return "Refund \$$jumlah via Stripe success.";
    }
}

// Keduanya bisa dipakai secara seragam
function prosesCheckout(Pembayaran $gateway, int $nominal): void
{
    echo $gateway->bayar($nominal) . "\n";
}

prosesCheckout(new Midtrans(), 150000);
prosesCheckout(new Stripe(), 10);
```

### 2.5 Trait

**Trait** adalah kumpulan method yang bisa "ditempelkan" ke banyak class tanpa inheritance. Laravel banyak menggunakan trait (contoh: `HasFactory`, `Notifiable`).

```php
<?php

trait Timestampable
{
    public function createdAt(): string
    {
        return date("Y-m-d H:i:s");
    }
}

trait SoftDeletable
{
    private bool $deleted = false;

    public function delete(): void
    {
        $this->deleted = true;
    }

    public function isDeleted(): bool
    {
        return $this->deleted;
    }
}

class Post
{
    use Timestampable, SoftDeletable; // pakai dua trait sekaligus

    public string $judul;

    public function __construct(string $judul)
    {
        $this->judul = $judul;
    }
}

$post = new Post("Belajar Laravel");
echo $post->createdAt();         // 2025-01-01 10:00:00
echo $post->isDeleted() ? "terhapus" : "aktif"; // aktif
$post->delete();
echo $post->isDeleted() ? "terhapus" : "aktif"; // terhapus
```

### 2.6 Static Method & Property

```php
<?php

class MathHelper
{
    // Konstanta — tidak berubah
    const PI = 3.14159;

    // Static property — dibagi semua instance
    private static int $hitungPanggilan = 0;

    // Static method — bisa dipanggil tanpa membuat object
    public static function luasLingkaran(float $r): float
    {
        self::$hitungPanggilan++;
        return self::PI * $r * $r;
    }

    public static function totalPanggilan(): int
    {
        return self::$hitungPanggilan;
    }
}

echo MathHelper::luasLingkaran(7);  // 153.938...
echo MathHelper::luasLingkaran(3);  // 28.274...
echo MathHelper::totalPanggilan();  // 2
```

---

## 3. TypeScript Dasar

TypeScript adalah JavaScript dengan **tipe data**. Di proyek Laravel + React Starter Kit, semua kode frontend ditulis dalam TypeScript (`.ts` / `.tsx`).

### 3.1 Tipe Data Dasar

```typescript
// Deklarasi variabel dengan tipe
let nama: string  = "Budi";
let umur: number  = 25;
let aktif: boolean = true;

// TypeScript akan error jika tipe tidak cocok
// nama = 123; // ❌ Error: Type 'number' is not assignable to type 'string'

// Union type — bisa lebih dari satu tipe
let id: string | number = "abc123";
id = 42; // ✅ boleh

// Optional (bisa ada, bisa tidak)
let nickname?: string;
```

### 3.2 Interface & Type

Di TypeScript, `interface` dan `type` dipakai untuk mendefinisikan **bentuk sebuah object**.

```typescript
// Interface — untuk mendefinisikan struktur object
interface User {
    id: number;
    name: string;
    email: string;
    role?: "admin" | "student" | "teacher"; // optional, hanya 3 nilai ini
}

// Pakai interface sebagai tipe
const user: User = {
    id: 1,
    name: "Rina",
    email: "rina@example.com",
    role: "student",
};

// Type alias — mirip interface, lebih fleksibel untuk union
type Status = "active" | "inactive" | "banned";
type ID = string | number;

// Type untuk object (sama seperti interface)
type Kursus = {
    id: ID;
    judul: string;
    status: Status;
};
```

### 3.3 Function dengan TypeScript

```typescript
// Tipe parameter dan return value
function tambah(a: number, b: number): number {
    return a + b;
}

// Arrow function
const sapa = (nama: string): string => `Halo, ${nama}!`;

// Generic function — bekerja dengan berbagai tipe
function getFirst<T>(arr: T[]): T | undefined {
    return arr[0];
}

const angkaPertama = getFirst([10, 20, 30]); // type: number
const namaPertama  = getFirst(["Ali", "Budi"]); // type: string
```

### 3.4 Array & Object Typing

```typescript
interface Produk {
    nama: string;
    harga: number;
}

// Array of interface
const produk: Produk[] = [
    { nama: "Buku",  harga: 50000 },
    { nama: "Pensil", harga: 5000 },
];

// Map / Record
const hargaPerItem: Record<string, number> = {
    buku:   50000,
    pensil: 5000,
    penghapus: 3000,
};

// Destructuring dengan tipe (sering di React props)
const { nama, harga }: Produk = produk[0];
console.log(`${nama}: Rp ${harga}`); // Buku: Rp 50000
```

### 3.5 Tipe yang Sering Dipakai di Laravel + React

```typescript
// Tipe untuk Inertia.js Page Props (data dari Laravel Controller)
interface PageProps {
    auth: {
        user: {
            id: number;
            name: string;
            email: string;
        };
    };
    flash?: {
        message?: string;
        error?: string;
    };
}

// Tipe untuk form
interface LoginForm {
    email: string;
    password: string;
    remember: boolean;
}

// Nullable — bisa null
interface Lesson {
    id: number;
    judul: string;
    videoUrl: string | null; // bisa ada, bisa null
    publishedAt: string | null;
}
```

---

## 4. React Dasar

React adalah library JavaScript untuk membangun **antarmuka pengguna (UI)**. Di Laravel Starter Kit, React dipakai bersama **Inertia.js** sebagai layer frontend.

### 4.1 Komponen

Komponen adalah "blok bangunan" UI. Di React modern, komponen berupa **function** yang mengembalikan JSX.

```tsx
// Komponen paling sederhana
function Halo() {
    return <h1>Halo, Dunia!</h1>;
}

// Komponen dengan props (parameter dari luar)
interface GreetingProps {
    nama: string;
    pesan?: string; // optional
}

function Greeting({ nama, pesan = "Selamat datang" }: GreetingProps) {
    return (
        <div>
            <h2>{pesan}, {nama}!</h2>
        </div>
    );
}

// Pemakaian
function App() {
    return (
        <div>
            <Halo />
            <Greeting nama="Budi" />
            <Greeting nama="Siti" pesan="Halo" />
        </div>
    );
}
```

### 4.2 useState — State / Data Lokal

`useState` adalah Hook untuk menyimpan data yang bisa **berubah** dan memicu re-render UI.

```tsx
import { useState } from "react";

function Counter() {
    // [nilai saat ini, fungsi untuk ubah nilai]
    const [count, setCount] = useState(0);

    return (
        <div>
            <p>Hitungan: {count}</p>
            <button onClick={() => setCount(count + 1)}>Tambah</button>
            <button onClick={() => setCount(count - 1)}>Kurang</button>
            <button onClick={() => setCount(0)}>Reset</button>
        </div>
    );
}
```

```tsx
// useState dengan object (contoh: form login)
import { useState } from "react";

interface LoginForm {
    email: string;
    password: string;
}

function LoginPage() {
    const [form, setForm] = useState<LoginForm>({
        email: "",
        password: "",
    });

    const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
        setForm({ ...form, [e.target.name]: e.target.value });
    };

    return (
        <form>
            <input
                name="email"
                type="email"
                value={form.email}
                onChange={handleChange}
                placeholder="Email"
            />
            <input
                name="password"
                type="password"
                value={form.password}
                onChange={handleChange}
                placeholder="Password"
            />
            <p>Email yang diketik: {form.email}</p>
        </form>
    );
}
```

### 4.3 useEffect — Efek Samping

`useEffect` dijalankan **setelah render**. Berguna untuk fetch data, subscribe event, dsb.

```tsx
import { useState, useEffect } from "react";

interface Post {
    id: number;
    title: string;
}

function PostList() {
    const [posts, setPosts] = useState<Post[]>([]);
    const [loading, setLoading] = useState(true);

    // [] artinya hanya jalan sekali saat komponen pertama kali muncul
    useEffect(() => {
        fetch("https://jsonplaceholder.typicode.com/posts?_limit=5")
            .then((res) => res.json())
            .then((data: Post[]) => {
                setPosts(data);
                setLoading(false);
            });
    }, []);

    if (loading) return <p>Memuat...</p>;

    return (
        <ul>
            {posts.map((post) => (
                <li key={post.id}>{post.title}</li>
            ))}
        </ul>
    );
}
```

### 4.4 Props & Komponen yang Dapat Digunakan Ulang

```tsx
// Komponen Button yang reusable
interface ButtonProps {
    label: string;
    onClick: () => void;
    variant?: "primary" | "danger" | "ghost";
    disabled?: boolean;
}

function Button({ label, onClick, variant = "primary", disabled = false }: ButtonProps) {
    const styles = {
        primary: "bg-blue-600 text-white hover:bg-blue-700",
        danger:  "bg-red-600 text-white hover:bg-red-700",
        ghost:   "bg-transparent text-gray-700 hover:bg-gray-100",
    };

    return (
        <button
            onClick={onClick}
            disabled={disabled}
            className={`px-4 py-2 rounded ${styles[variant]} disabled:opacity-50`}
        >
            {label}
        </button>
    );
}

// Pemakaian
function FormPage() {
    const handleSimpan = () => alert("Data disimpan!");
    const handleHapus  = () => alert("Data dihapus!");

    return (
        <div className="flex gap-2">
            <Button label="Simpan" onClick={handleSimpan} />
            <Button label="Hapus"  onClick={handleHapus}  variant="danger" />
            <Button label="Batal"  onClick={() => {}}     variant="ghost" />
        </div>
    );
}
```

### 4.5 Conditional Rendering & List

```tsx
interface Kursus {
    id: number;
    judul: string;
    status: "aktif" | "draft";
}

function KursusList({ kursus }: { kursus: Kursus[] }) {
    // Jika data kosong
    if (kursus.length === 0) {
        return <p className="text-gray-500">Belum ada kursus.</p>;
    }

    return (
        <ul className="space-y-2">
            {kursus.map((k) => (
                <li key={k.id} className="flex items-center gap-2">
                    <span>{k.judul}</span>

                    {/* Conditional rendering dengan && */}
                    {k.status === "draft" && (
                        <span className="text-xs bg-yellow-100 text-yellow-800 px-2 py-0.5 rounded">
                            Draft
                        </span>
                    )}

                    {/* Ternary operator */}
                    <span className={k.status === "aktif" ? "text-green-600" : "text-gray-400"}>
                        ●
                    </span>
                </li>
            ))}
        </ul>
    );
}
```

### 4.6 useForm dari Inertia.js (Khusus Laravel)

Di proyek Laravel + React, form biasanya menggunakan **`useForm` dari Inertia.js**, bukan fetch biasa.

```tsx
import { useForm } from "@inertiajs/react";

interface RegisterForm {
    name: string;
    email: string;
    password: string;
    password_confirmation: string;
}

function RegisterPage() {
    const { data, setData, post, processing, errors } = useForm<RegisterForm>({
        name: "",
        email: "",
        password: "",
        password_confirmation: "",
    });

    const handleSubmit = (e: React.FormEvent) => {
        e.preventDefault();
        post("/register"); // kirim ke Laravel route /register
    };

    return (
        <form onSubmit={handleSubmit} className="space-y-4">
            <div>
                <input
                    type="text"
                    value={data.name}
                    onChange={(e) => setData("name", e.target.value)}
                    placeholder="Nama lengkap"
                />
                {/* Tampilkan error dari Laravel validation */}
                {errors.name && <p className="text-red-500 text-sm">{errors.name}</p>}
            </div>

            <div>
                <input
                    type="email"
                    value={data.email}
                    onChange={(e) => setData("email", e.target.value)}
                    placeholder="Email"
                />
                {errors.email && <p className="text-red-500 text-sm">{errors.email}</p>}
            </div>

            <button type="submit" disabled={processing}>
                {processing ? "Mendaftar..." : "Daftar"}
            </button>
        </form>
    );
}
```

---

## Ringkasan

| Topik | Konsep Kunci | Relevansinya di Laravel + React |
|---|---|---|
| **PHP Dasar** | Variabel, Array, Loop, Function | Dipakai di semua logika Controller, Model, Seeder |
| **PHP OOP** | Class, Interface, Trait, Static | Dasar dari semua komponen Laravel |
| **TypeScript** | Interface, Type, Generic | Typing props, data dari API, form |
| **React** | Komponen, Props, useState, useEffect | Semua halaman & komponen UI |
| **Inertia.js** | `useForm`, `usePage` | Penghubung antara Laravel & React |

---

> **Selanjutnya → Modul 01**: Persiapan Ekosistem, Fondasi & Test Framework.

---

# Modul 1: Persiapan Ekosistem, Fondasi & Test Framework

**Tujuan**: Menyiapkan tools, membuat project, memahami struktur, dan menulis test pertama.
**Hasil Visual**: Menu "Courses" di sidebar + halaman placeholder.

---

## 1.1 Instalasi Tools Pengembangan

### Laravel Herd (PHP Environment)

Herd menyediakan PHP, Nginx, Composer, Node.js secara otomatis — tanpa konfigurasi manual.

1. Download dari [herd.laravel.com](https://herd.laravel.com) → instal.
2. Buka **Herd Settings → General** → catat _Sites Path_ (`~/Herd` di macOS, `C:\Users\<nama>\Herd` di Windows).
3. Verifikasi di terminal:

```bash
# Pastikan semua tool tersedia dan versinya sesuai
php -v        # Harus PHP 8.5+
composer -V   # Composer terbaru
node -v       # Node.js 20+
npm -v        # npm terbaru
```

### DBngin (Database Server)

1. Download dari [dbngin.com](https://dbngin.com) → instal.
2. Buka → klik **"+"** → pilih **MySQL** → nama: `MySQL-8` → **Create** → **Start**.

| Parameter | Nilai |
|-----------|-------|
| Host | `127.0.0.1` |
| Port | `3306` |
| Username | `root` |
| Password | _(kosong — tidak perlu diisi)_ |

### DBeaver (Database Client)

1. Download dari [dbeaver.io/download](https://dbeaver.io/download/) → instal.
2. **Database → New Connection → MySQL** → isi sesuai tabel di atas → **Test Connection** → pastikan sukses.
3. Klik kanan koneksi → **Create New Database** → nama: `akademi`.

### PHPStorm (IDE)

1. Download dari [jetbrains.com/phpstorm](https://www.jetbrains.com/phpstorm/) → instal.
2. **Settings → Plugins → Marketplace** → instal:
   - **Laravel Idea** — autocomplete route, model, view.
   - **Pest** — syntax highlighting + test runner.
3. **Settings → PHP → CLI Interpreter** → arahkan ke PHP Herd.
4. **Settings → PHP → Test Frameworks** → pilih **Pest** → arahkan ke `vendor/autoload.php`.
5. **Settings → Languages & Frameworks → JavaScript → Code Quality Tools → ESLint** → centang **Automatic ESLint configuration**. PHPStorm akan otomatis membaca konfigurasi ESLint dari starter kit.

---

## 1.2 Inisialisasi Project Laravel 12

```bash
# Pastikan Laravel installer terbaru
composer global require laravel/installer

# Masuk ke folder Sites Path Herd
cd ~/Herd

# Buat project baru
laravel new akademi
```

**Pilih opsi berikut:**

```
┌ Which starter kit would you like to install? ─────────────────┐
│ › React                                                        │
└────────────────────────────────────────────────────────────────┘

┌ Which authentication provider would you like to use? ──────────┐
│ › Laravel's built-in authentication                            │
└────────────────────────────────────────────────────────────────┘

┌ Which testing framework do you prefer? ────────────────────────┐
│ › Pest                    ← PENTING! Bukan PHPUnit             │
└────────────────────────────────────────────────────────────────┘

┌ Which database will your application use? ─────────────────────┐
│ › MySQL                                                        │
└────────────────────────────────────────────────────────────────┘

┌ Would you like to run the default database migrations? ────────┐
│ › No    ← Kita jalankan manual setelah .env dikonfigurasi      │
└────────────────────────────────────────────────────────────────┘
```

```bash
# Masuk ke folder project
cd akademi

# Instal dependency frontend
npm install
```

---

## 1.3 Konfigurasi Database

Buka file `.env` di root project, cari bagian `DB_*` dan sesuaikan:

```env
# Koneksi ke MySQL di DBngin
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=akademi     # Nama database yang dibuat di DBeaver
DB_USERNAME=root
DB_PASSWORD=             # Kosong — DBngin default tanpa password
```

```bash
# Jalankan migrasi bawaan starter kit (tabel users, sessions, dll)
php artisan migrate
```

> Jika database `akademi` belum ada, Laravel akan menawarkan untuk membuatnya — pilih **Yes**.

---

## 1.4 Strict Mode Eloquent

Buka `app/Providers/AppServiceProvider.php`:

```php
<?php

namespace App\Providers;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        //
    }

    public function boot(): void
    {
        $this->configureDefaults();
        // Aktifkan Strict Mode untuk menangkap bug lebih awal:
        // 1. Lazy loading → exception (mencegah N+1 query problem)
        // 2. Mass assignment pada kolom yang tidak di-$fillable → exception
        // 3. Akses atribut yang tidak ada → exception
        Model::shouldBeStrict();
    }
}
```

---

## 1.5 Memahami Struktur Starter Kit

Sebelum lanjut, pahami di mana file-file penting berada:

```
akademi/
├── app/
│   ├── Actions/Fortify/        ← Logic autentikasi (register, password)
│   ├── Http/Controllers/       ← Controller backend (logic bisnis)
│   ├── Models/                 ← Eloquent models (representasi tabel)
│   └── Providers/              ← Service providers (konfigurasi app)
│
├── database/
│   ├── factories/              ← Factory (pabrik data palsu untuk testing)
│   ├── migrations/             ← Migration (skema database, versi-controlled)
│   └── seeders/                ← Seeder (data dummy untuk development)
│
├── resources/js/               ← ⭐ SEMUA kode React ada di sini
│   ├── components/             ← Komponen reusable
│   │   └── ui/                 ← Komponen shadcn/ui (Button, Card, dll)
│   ├── layouts/                ← Layout system (sidebar vs header)
│   │   ├── app/                ← Layout untuk user yang sudah login
│   │   └── auth/               ← Layout untuk halaman login/register
│   ├── pages/                  ← ⭐ Halaman (1 file .tsx = 1 halaman)
│   ├── types/                  ← TypeScript type definitions
│   └── actions/                ← ⭐ Auto-generated oleh Wayfinder
│
├── routes/
│   └── web.php                 ← Semua route didefinisikan di sini
│
├── tests/
│   ├── Feature/                ← Feature tests (HTTP request + database)
│   └── Unit/                   ← Unit tests (logic murni tanpa DB)
│
├── .env                        ← Konfigurasi environment (DB, dll)
├── phpunit.xml                 ← Konfigurasi testing
├── vite.config.ts              ← Konfigurasi Vite (bundler frontend)
└── eslint.config.js            ← Konfigurasi ESLint (linter JS/TS)
```

**Tentang Wayfinder**: Perhatikan folder `resources/js/actions/` — ini di-generate otomatis oleh Wayfinder saat `npm run dev` berjalan. Folder ini sudah ada di `.gitignore`. Setiap kali kamu menambah route atau controller method baru, Wayfinder akan membuat ulang file TypeScript di sini.

**Tentang ESLint**: Starter kit sudah menyertakan `eslint.config.js` yang dikonfigurasi untuk React + TypeScript. PHPStorm akan otomatis membaca konfigurasi ini jika kamu mengaktifkan **Automatic ESLint configuration** di Settings.

---

## 1.6 Menjalankan Aplikasi

Karena project ada di Sites Path Herd, aplikasi otomatis tersedia di `http://akademi.test`.

```bash
# Jalankan Vite dev server (hot-reload untuk React)
npm run dev
```

Buka browser → `http://akademi.test`:

1. Kamu melihat **halaman Welcome Laravel 12**.
2. Klik **Register** → buat akun → masuk ke **Dashboard** dengan sidebar shadcn/ui!

---

## 1.7 Setup Pest PHP & Test Pertama

### Konfigurasi `tests/Pest.php`

Buka file `tests/Pest.php` dan tambahkan helper functions:

```php
<?php

/*
|--------------------------------------------------------------------------
| Test Case
|--------------------------------------------------------------------------
|
| The closure you provide to your test functions is always bound to a specific PHPUnit test
| case class. By default, that class is "PHPUnit\Framework\TestCase". Of course, you may
| need to change it using the "pest()" function to bind a different classes or traits.
|
*/

use App\Models\User;

pest()->extend(Tests\TestCase::class)
    ->use(Illuminate\Foundation\Testing\RefreshDatabase::class)
    ->in('Feature');


/**
 * Helper: Login sebagai user dengan role tertentu.
 * Membuat user baru di database, lalu "bertindak sebagai" user tersebut.
 *
 * Contoh penggunaan dalam test:
 *   $user = loginAs('student');
 *   $this->get('/dashboard')->assertOk();
 */
function loginAs(string $role = 'student'): User
{
    // Buat user baru menggunakan Factory
    $user = User::factory()->create(['role' => $role]);

    // Jadikan user ini sebagai "pengguna aktif" dalam test
    test()->actingAs($user);

    return $user;
}

/**
 * Helper: Buat user dengan role instructor.
 * Shortcut agar test lebih ringkas.
 */
function createInstructor(array $attributes = []): User
{
    return User::factory()->create(
        array_merge(['role' => 'instructor'], $attributes)
    );
}

/**
 * Helper: Buat user dengan role student.
 */
function createStudent(array $attributes = []): User
{
    return User::factory()->create(
        array_merge(['role' => 'student'], $attributes)
    );
}

```

### Konfigurasi `phpunit.xml`

Buka `phpunit.xml`, pastikan ada di bagian `<env>`:

```xml
<!-- Gunakan SQLite in-memory untuk testing agar cepat -->
<!-- Database di-reset setiap test (RefreshDatabase trait) -->
<env name="APP_ENV" value="testing"/>
<env name="DB_CONNECTION" value="sqlite"/>
<env name="DB_DATABASE" value=":memory:"/>
```

### Smoke Test (Test Pertama!)

```bash
# Buat file test baru menggunakan Pest
php artisan make:test --pest SmokeTest
```

Buka `tests/Feature/SmokeTest.php`:

```php
<?php

// Smoke Test = test paling dasar untuk memastikan
// aplikasi bisa berjalan tanpa error fatal.
// Jalankan ini setiap kali ada perubahan besar.

it('can load the landing page', function () {
    // Kirim GET request ke halaman utama
    // assertStatus(200) = pastikan halaman tampil sukses
    $this->get('/')->assertStatus(200);
});

it('can load the login page', function () {
    $this->get('/login')->assertStatus(200);
});

it('can load the register page', function () {
    $this->get('/register')->assertStatus(200);
});

it('redirects guest from dashboard to login', function () {
    // Guest (belum login) tidak boleh akses dashboard
    // Harus di-redirect ke halaman login
    $this->get('/dashboard')->assertRedirect('/login');
});
```

```bash
# Jalankan test!
php artisan test tests/Feature/SmokeTest.php
```

✅ Semua hijau!

> **💡 PHPStorm Tip**: Klik ikon ▶ hijau di samping `it(...)` untuk menjalankan satu test. Gunakan **Ctrl+Shift+F10** (Windows) / **⌃⇧R** (Mac) untuk run file.

---

## 1.8 Hasil Visual: Menambah Halaman & Menu Sidebar

Agar kamu langsung merasakan _workflow_ membuat halaman baru, kita tambahkan menu **"Courses"** di sidebar.

### Buat Route

Buka `routes/web.php` dan tambahkan:

```php
// Route sementara (placeholder) untuk halaman Courses.
// Nanti di Modul 3, kita akan mengganti ini dengan Controller.
Route::get('/courses', function () {
    return \Inertia\Inertia::render('course/index');
})->name('courses.index');
```

### Buat Halaman Placeholder

Buat file baru `resources/js/pages/course/index.tsx`:

```tsx
/**
 * Halaman Katalog Kursus (placeholder).
 * Di Modul 3, halaman ini akan diisi dengan data kursus dari database.
 *
 * Setiap file di folder pages/ = satu halaman di aplikasi.
 * Nama folder dan file harus sesuai dengan parameter di Inertia::render().
 * Contoh: Inertia::render('course/index') → pages/course/index.tsx
 */

// AppLayout = layout utama dengan sidebar (sudah ada dari starter kit)
// Head = komponen Inertia untuk mengatur <title> halaman
import { Head } from '@inertiajs/react';
import AppLayout from '@/layouts/app-layout';


export default function CourseIndex() {
    return (
        <AppLayout>
            {/* Set judul tab browser */}
            <Head title="Course Catalog" />

            <div className="mx-auto max-w-7xl px-4 py-8 sm:px-6 lg:px-8">
                <h1 className="text-3xl font-bold tracking-tight">
                    Course Catalog
                </h1>
                <p className="mt-2 text-muted-foreground">
                    Halaman ini akan menampilkan daftar kursus. Kita akan
                    membangunnya di Modul 3!
                </p>
            </div>
        </AppLayout>
    );
}
```

### Ganti Layout

Buka file `resources/js/layouts/app-layout.tsx`. Ganti app-sidebar-layout menjadi app-header-layout:

```tsx
import AppLayoutTemplate from '@/layouts/app/app-header-layout';
import type { AppLayoutProps } from '@/types';

export default ({ children, breadcrumbs, ...props }: AppLayoutProps) => (
    <AppLayoutTemplate breadcrumbs={breadcrumbs} {...props}>
        {children}
    </AppLayoutTemplate>
);

```


### Tambahkan Menu di Sidebar

Buka file `resources/js/components/app-header.tsx`. Cari array nav items dan tambahkan:

```tsx
// Import icon yang dibutuhkan dari lucide-react
import { BookOpen, LayoutGrid } from 'lucide-react';

// Cari array navMain atau mainNavItems (nama bisa bervariasi),
// lalu tambahkan item Courses:
const mainNavItems: NavItem[] = [
    {
        title: 'Dashboard',
        href: '/dashboard',
        icon: LayoutGrid,
    },
    // ✨ Item baru: Courses
    {
        title: 'Courses',
        href: '/courses',
        icon: BookOpen,    // Icon buku dari lucide-react
    },
];
```

> **Catatan**: Struktur persis `app-header.tsx` bisa berbeda antar versi starter kit. Prinsipnya: cari array nav items yang sudah ada, perhatikan pattern-nya, dan tambahkan entry baru mengikuti format yang sama.

### Lihat Hasilnya!

Pastikan `npm run dev` berjalan, buka `http://akademi.test/dashboard`:

- ✅ Sidebar sekarang punya menu **"Courses"** dengan icon buku.
- ✅ Klik → navigasi ke halaman placeholder **"Course Catalog"**.

---

## ✅ Checkpoint Modul 1

Jalankan semua verifikasi ini sebelum lanjut ke Modul 2:

```bash
# 1. Semua test harus hijau
php artisan test
# Expected: 4 passed (SmokeTest)

# 2. ESLint harus bersih (tidak ada error)
npx eslint resources/js/pages/course/index.tsx
# Expected: tidak ada output error

# 3. TypeScript harus valid
npx tsc --noEmit
# Expected: tidak ada error
```

**Verifikasi visual:**

1. ✅ `http://akademi.test` → halaman Welcome tampil.
2. ✅ Register akun → masuk Dashboard dengan sidebar.
3. ✅ Menu **Courses** ada di sidebar → klik menuju halaman placeholder.
4. ✅ DBeaver → database `akademi` terhubung, tabel `users` ada.
