# SPESIFIKASI.md — Haland Petcare
## Sumber Kebenaran Mutlak (Single Source of Truth)

| | |
|---|---|
| **Versi** | 2.0 (Super Lengkap) |
| **Status** | MVP Development |
| **Tipe Sistem** | Single Clinic (bukan SaaS, bukan multi-tenant) |
| **Target Eksekutor** | GitHub Copilot Chat (Codespace) |

> Dokumen ini adalah rujukan **tunggal dan mutlak**. Copilot/AI Assistant WAJIB membaca seluruh dokumen ini sebelum menulis kode apapun, dan tidak boleh mengambil keputusan arsitektur di luar yang tertulis di sini. Jika ada kebutuhan yang tidak tercakup, tanyakan dulu — jangan berasumsi.

---

## DAFTAR ISI

1. Gambaran Umum
2. Tech Stack
3. Environment Variables
4. Struktur Folder Project
5. Role & Hak Akses (Matrix Lengkap)
6. Authentication & Session
7. Database Schema (Prisma Lengkap)
8. Kontrak Server Actions per Modul
9. Skema Validasi Zod
10. Modul: User Management
11. Modul: Customer & Pet
12. Modul: Appointment
13. Modul: Medical Record
14. Modul: Pet Hotel
15. Modul: Services
16. Modul: Billing & Invoice
17. Modul: Dashboard
18. Modul: Report
19. Modul: Setting
20. Format Response & Error Handling
21. Seed Data Awal
22. Routing & Halaman
23. UI/UX Guidelines
24. Testing Checklist per Role
25. Rule Absolut
26. Target MVP
27. Out of Scope
28. Catatan Deployment
29. Roadmap Implementasi (Referensi Fase)

---

## 1. GAMBARAN UMUM

Haland Petcare adalah sistem manajemen klinik hewan berbasis web untuk **satu klinik hewan tunggal**. Sistem mencakup pendaftaran pelanggan, manajemen data hewan, appointment, pemeriksaan dokter, pet hotel, rekam medis, billing/invoice, laporan, dan setting.

### Prinsip Utama

- Simpel, cepat dikembangkan, tidak ada kompleksitas enterprise.
- Tidak ada multi-tenant / multi-clinic. Tidak ada `clinic_id` di manapun.
- Tidak ada self-registration. Semua akun dibuat oleh internal klinik.
- Login menggunakan **Username + PIN**, bukan email/password.
- Semua mutasi data lewat **Server Actions**, bukan REST API custom.
- Semua input divalidasi dengan **Zod** sebelum menyentuh database.
- Setiap query WAJIB di-scope sesuai role yang login (tidak ada "lupa filter").

---

## 2. TECH STACK (FIXED — TIDAK BOLEH DIUBAH TANPA PERSETUJUAN)

| Layer | Teknologi | Catatan |
|---|---|---|
| Framework | Next.js 14+ (App Router) | Gunakan Server Components sebisa mungkin, Client Component hanya untuk interaktivitas |
| Bahasa | TypeScript | Strict mode aktif |
| Styling | Tailwind CSS | |
| UI Components | shadcn/ui | Install per-komponen sesuai kebutuhan, jangan install semua sekaligus |
| ORM | Prisma | |
| Database | SQLite (dev) → PostgreSQL (production) | Gunakan `DATABASE_URL` di .env agar migrasi mudah |
| Auth | NextAuth (Auth.js) v5 — Credentials Provider | JWT strategy |
| Validasi | Zod | Definisikan schema di `lib/validations/` |
| Chart | Recharts | Untuk dashboard Owner |
| Hashing PIN | bcrypt | Salt rounds minimal 10 |
| Form Handling | react-hook-form + @hookform/resolvers/zod | Opsional tapi disarankan |
| State Notifikasi UI | shadcn `toast`/`sonner` | Untuk feedback sukses/gagal Server Action |

**Tidak digunakan:**
- REST API custom (`app/api/*` hanya untuk kebutuhan NextAuth callback, bukan untuk CRUD)
- Library multi-tenancy apapun
- Library RBAC generik (role cukup 1 kolom enum di tabel `users`)
- ORM lain selain Prisma
- CSS framework lain selain Tailwind

---

## 3. ENVIRONMENT VARIABLES

Buat file `.env` (dan `.env.example` sebagai template, tanpa nilai rahasia):

```env
# Database
DATABASE_URL="file:./dev.db"

# NextAuth
NEXTAUTH_SECRET="generate-dengan-openssl-rand-base64-32"
NEXTAUTH_URL="http://localhost:3000"

# App
NODE_ENV="development"
```

Catatan:
- `NEXTAUTH_SECRET` wajib di-generate unik, jangan dikosongkan.
- Saat migrasi ke PostgreSQL, cukup ganti `DATABASE_URL` dan `provider` di `schema.prisma`, tidak ada perubahan skema logika.

---

## 4. STRUKTUR FOLDER PROJECT

```
haland-petcare/
├── app/
│   ├── (auth)/
│   │   └── login/
│   │       └── page.tsx
│   ├── (dashboard)/
│   │   ├── dashboard/
│   │   │   └── page.tsx
│   │   ├── users/                # Owner only
│   │   ├── customers/            # Owner, Admin
│   │   ├── pets/                 # Owner, Admin, Customer(own)
│   │   ├── appointments/         # Semua role (scoped)
│   │   ├── medical-records/      # Dokter, Owner(RO), Admin(RO), Customer(RO,own)
│   │   ├── pet-hotel/            # Owner, Admin, Dokter(RO)
│   │   ├── billing/              # Owner, Admin, Customer(RO,own)
│   │   ├── services/             # Owner, Admin
│   │   ├── reports/              # Owner
│   │   └── settings/             # Owner
│   ├── layout.tsx
│   └── globals.css
├── components/
│   ├── ui/                       # shadcn components
│   ├── layout/                   # Sidebar, Navbar, per-role nav
│   └── forms/                    # Form-form per modul
├── lib/
│   ├── auth.ts                   # NextAuth config
│   ├── prisma.ts                 # Prisma client singleton
│   ├── validations/              # Zod schemas per modul
│   │   ├── user.ts
│   │   ├── customer.ts
│   │   ├── pet.ts
│   │   ├── appointment.ts
│   │   ├── medical-record.ts
│   │   ├── pet-hotel.ts
│   │   ├── service.ts
│   │   └── invoice.ts
│   ├── actions/                  # Server Actions per modul
│   │   ├── user.actions.ts
│   │   ├── customer.actions.ts
│   │   ├── pet.actions.ts
│   │   ├── appointment.actions.ts
│   │   ├── medical-record.actions.ts
│   │   ├── pet-hotel.actions.ts
│   │   ├── service.actions.ts
│   │   └── invoice.actions.ts
│   ├── auth-guard.ts             # Helper cek role di Server Action
│   └── utils.ts
├── prisma/
│   ├── schema.prisma
│   └── seed.ts
├── middleware.ts                 # Proteksi route berdasarkan role
├── .env.example
└── spesifikasi.md
```

---

## 5. ROLE & HAK AKSES (MATRIX LENGKAP)

| Fitur / Aksi | OWNER | ADMIN | DOKTER | CUSTOMER |
|---|:---:|:---:|:---:|:---:|
| Login | ✅ | ✅ | ✅ | ✅ |
| Buat akun Admin/Dokter | ✅ | ❌ | ❌ | ❌ |
| Buat akun Customer | ✅ | ✅ | ❌ | ❌ |
| Reset PIN user lain | ✅ | ❌ | ❌ | ❌ |
| Aktif/nonaktifkan akun | ✅ | ❌ | ❌ | ❌ |
| Registrasi Customer baru | ✅ | ✅ | ❌ | ❌ |
| Registrasi Pet | ✅ | ✅ | ❌ | ❌ (lihat saja) |
| Lihat semua data pet | ✅ | ✅ | ✅ (pet appointment-nya) | ❌ (hanya milik sendiri) |
| Booking Appointment | ✅ | ✅ | ❌ | ❌ (read-only) |
| Ubah status appointment → IN_PROGRESS/COMPLETED | ✅ (override) | ❌ | ✅ (hanya appointment miliknya) | ❌ |
| Isi/edit Medical Record | ❌ (RO) | ❌ (RO) | ✅ | ❌ (RO, milik sendiri) |
| Booking/Check-in/Check-out Pet Hotel | ✅ | ✅ | ❌ (RO) | ❌ |
| Kelola Services (harga obat/layanan) | ✅ | ✅ | ❌ | ❌ |
| Generate Invoice | ✅ | ✅ | ❌ | ❌ |
| Tandai Invoice PAID | ✅ | ✅ | ❌ | ❌ |
| Lihat Invoice | ✅ (semua) | ✅ (semua) | ❌ | ✅ (milik sendiri) |
| Lihat Report | ✅ | ❌ | ❌ | ❌ |
| Akses Setting | ✅ | ❌ | ❌ | ❌ |

> Catatan: sesuai spesifikasi awal, Admin yang membuat appointment atas nama customer (walk-in/telepon), Customer bersifat read-only untuk appointment. Jika ke depan Customer diberi hak self-booking, ini HARUS diupdate dulu di dokumen ini sebelum diimplementasikan (lihat Rule Absolut #15).

### 5.1 Detail per Role

**OWNER** — Akses penuh ke seluruh sistem.
Menu: Dashboard, User Management, Customer, Pet, Appointment, Medical Record, Pet Hotel, Billing, Services, Report, Setting.

**ADMIN OPERASIONAL** — Mengelola operasional harian klinik.
Menu: Dashboard, Customer, Pet, Appointment, Pet Hotel, Billing, Services.
Larangan: Tidak dapat membuat akun Admin maupun Dokter.

**DOKTER HEWAN**
Menu: Dashboard, Appointment (miliknya), Medical Record, Pet Hotel (View only).
Larangan: Tidak dapat mengubah pembayaran/billing dalam kondisi apapun.

**CUSTOMER**
Menu: Dashboard, My Pets, Appointment, Pet Hotel, Medical Record (RO), Invoice (RO).
Larangan: Hanya dapat melihat data miliknya sendiri (scoped by `customer_id` yang terhubung ke `user_id` miliknya). Tidak dapat membuat akun sendiri.

---

## 6. AUTHENTICATION & SESSION

### 6.1 Flow Login

1. User membuka `/login`, input Username + PIN (4-6 digit numerik, ditentukan saat pembuatan akun).
2. Server Action / NextAuth `authorize()` mencari user berdasarkan `username`.
3. Jika ditemukan dan `is_active = true`, bandingkan PIN dengan `pin_hash` menggunakan `bcrypt.compare`.
4. Jika cocok, buat session JWT berisi: `id`, `username`, `role`, `name`.
5. Redirect ke `/dashboard` — halaman dashboard membaca `session.user.role` untuk menampilkan widget yang sesuai (lihat bagian 17).
6. Jika gagal (username tidak ada, PIN salah, atau akun nonaktif), tampilkan pesan error generik: **"Username atau PIN salah"** (jangan bedakan pesan error agar tidak bocor informasi apakah username ada atau tidak).

### 6.2 Rule Pembuatan Akun

| Pembuat | Dapat Membuat |
|---|---|
| Owner | Admin Operasional, Dokter |
| Admin Operasional | Customer |
| Customer | — (tidak dapat membuat akun sendiri) |

Saat sebuah akun dibuat:
- Username harus unik (validasi di Zod + constraint database).
- PIN awal di-generate atau diinput manual oleh pembuat akun, langsung di-hash sebelum disimpan — tidak pernah dikirim via email/SMS (karena spesifikasi tidak mencakup notifikasi otomatis). Sampaikan PIN awal secara langsung/manual ke pemilik akun.
- Untuk akun Customer, buat juga baris di tabel `customers` yang terhubung ke `user_id` tersebut dalam satu transaksi (`prisma.$transaction`).

### 6.3 Proteksi Route

Gunakan `middleware.ts` untuk redirect otomatis jika:
- User belum login mengakses halaman `(dashboard)/*` → redirect ke `/login`.
- User login tapi mengakses menu di luar hak rolenya (mis. Customer akses `/users`) → redirect ke `/dashboard` atau halaman 403.

Selain proteksi di middleware (level UI), **setiap Server Action WAJIB melakukan pengecekan role ulang di dalam function-nya** menggunakan helper `lib/auth-guard.ts` — middleware saja tidak cukup karena Server Action bisa dipanggil langsung.

Contoh pola helper:
```ts
// lib/auth-guard.ts
export async function requireRole(allowedRoles: Role[]) {
  const session = await auth();
  if (!session || !allowedRoles.includes(session.user.role)) {
    throw new Error("UNAUTHORIZED");
  }
  return session;
}
```

### 6.4 Session Expiry

- JWT session default expiry 7 hari (bisa disesuaikan di `lib/auth.ts`), tidak ada requirement khusus dari spesifikasi awal — gunakan default NextAuth yang wajar untuk aplikasi internal klinik.

---

## 7. DATABASE SCHEMA (PRISMA LENGKAP)

> Ini adalah representasi lengkap untuk `schema.prisma`. Semua nama tabel, field, tipe, dan enum di bawah ini bersifat final — jangan menambah/mengurangi tanpa update dokumen ini dulu.

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "sqlite" // ganti "postgresql" saat production
  url      = env("DATABASE_URL")
}

enum Role {
  OWNER
  ADMIN
  DOKTER
  CUSTOMER
}

enum Gender {
  MALE
  FEMALE
}

enum AppointmentStatus {
  PENDING
  CONFIRMED
  CHECKED_IN
  IN_PROGRESS
  COMPLETED
  CANCELLED
}

enum HotelRoomStatus {
  AVAILABLE
  OCCUPIED
}

enum PetHotelStatus {
  BOOKED
  CHECKED_IN
  STAYING
  CHECKED_OUT
  CANCELLED
}

enum InvoiceStatus {
  UNPAID
  PAID
}

model User {
  id        String   @id @default(cuid())
  username  String   @unique
  pinHash   String   @map("pin_hash")
  role      Role
  name      String
  phone     String?
  isActive  Boolean  @default(true) @map("is_active")
  createdAt DateTime @default(now()) @map("created_at")

  customer  Customer?          // relasi 1-1 jika role = CUSTOMER
  doctorAppointments Appointment[] @relation("DoctorAppointments")

  @@map("users")
}

model Customer {
  id        String   @id @default(cuid())
  userId    String   @unique @map("user_id")
  user      User     @relation(fields: [userId], references: [id])
  address   String?
  createdAt DateTime @default(now()) @map("created_at")

  pets      Pet[]
  invoices  Invoice[]

  @@map("customers")
}

model Pet {
  id         String    @id @default(cuid())
  customerId String    @map("customer_id")
  customer   Customer  @relation(fields: [customerId], references: [id])
  name       String
  species    String
  breed      String?
  birthDate  DateTime? @map("birth_date")
  gender     Gender
  notes      String?

  appointments Appointment[]
  petHotels    PetHotel[]

  @@map("pets")
}

model Appointment {
  id           String            @id @default(cuid())
  petId        String            @map("pet_id")
  pet          Pet               @relation(fields: [petId], references: [id])
  doctorId     String            @map("doctor_id")
  doctor       User              @relation("DoctorAppointments", fields: [doctorId], references: [id])
  scheduledAt  DateTime          @map("scheduled_at")
  status       AppointmentStatus @default(PENDING)
  complaint    String?
  createdAt    DateTime          @default(now()) @map("created_at")

  medicalRecord MedicalRecord?
  invoices      Invoice[]

  @@map("appointments")
}

model MedicalRecord {
  id            String      @id @default(cuid())
  appointmentId String      @unique @map("appointment_id")
  appointment   Appointment @relation(fields: [appointmentId], references: [id])
  vitalSign     String?     @map("vital_sign")
  diagnosis     String?
  treatment     String?
  prescription  String?
  notes         String?
  createdAt     DateTime    @default(now()) @map("created_at")

  @@map("medical_records")
}

model HotelRoom {
  id         String          @id @default(cuid())
  roomNumber String          @unique @map("room_number")
  roomType   String          @map("room_type")
  status     HotelRoomStatus @default(AVAILABLE)

  petHotels  PetHotel[]

  @@map("hotel_rooms")
}

model PetHotel {
  id                String         @id @default(cuid())
  petId             String         @map("pet_id")
  pet               Pet            @relation(fields: [petId], references: [id])
  roomId            String         @map("room_id")
  room              HotelRoom      @relation(fields: [roomId], references: [id])
  checkInDate       DateTime       @map("check_in_date")
  checkOutDate      DateTime?      @map("check_out_date")
  food              String?
  feedingSchedule   String?        @map("feeding_schedule")
  medicineSchedule  String?        @map("medicine_schedule")
  notes             String?
  status            PetHotelStatus @default(BOOKED)

  invoices          Invoice[]

  @@map("pet_hotels")
}

model Service {
  id       String  @id @default(cuid())
  name     String
  price    Decimal
  category String

  invoiceItems InvoiceItem[]

  @@map("services")
}

model Invoice {
  id            String        @id @default(cuid())
  customerId    String        @map("customer_id")
  customer      Customer      @relation(fields: [customerId], references: [id])
  appointmentId String?       @map("appointment_id")
  appointment   Appointment?  @relation(fields: [appointmentId], references: [id])
  petHotelId    String?       @map("pet_hotel_id")
  petHotel      PetHotel?     @relation(fields: [petHotelId], references: [id])
  totalAmount   Decimal       @map("total_amount")
  status        InvoiceStatus @default(UNPAID)
  createdAt     DateTime      @default(now()) @map("created_at")

  items InvoiceItem[]

  @@map("invoices")
}

model InvoiceItem {
  id        String  @id @default(cuid())
  invoiceId String  @map("invoice_id")
  invoice   Invoice @relation(fields: [invoiceId], references: [id])
  serviceId String  @map("service_id")
  service   Service @relation(fields: [serviceId], references: [id])
  quantity  Int
  price     Decimal

  @@map("invoice_items")
}
```

### 7.1 Catatan Relasi Penting

- `User.customer` bersifat opsional karena hanya user dengan `role = CUSTOMER` yang punya baris `Customer`.
- `Appointment.doctorId` selalu merujuk ke `User` dengan `role = DOKTER` — validasi ini dilakukan di level aplikasi (Zod + Server Action), bukan di level database, karena Prisma tidak mendukung constraint enum-conditional secara native.
- `MedicalRecord` bersifat 1-1 dengan `Appointment` — satu appointment maksimal punya satu medical record.
- `Invoice` bisa terhubung ke `Appointment` **atau** `PetHotel` (nullable keduanya), tapi idealnya minimal salah satu terisi — validasi ini di level Server Action.
- Field `Decimal` di SQLite disimpan sebagai `Float`/`String` tergantung driver Prisma; pastikan format ke Rupiah dilakukan di layer presentasi, bukan di database.

---

## 8. KONTRAK SERVER ACTIONS PER MODUL

Semua Server Action mengembalikan bentuk response yang konsisten (lihat bagian 20). Berikut daftar minimal function yang harus ada:

### `user.actions.ts`
- `createStaffAccount(data)` — Owner only, buat Admin/Dokter
- `createCustomerAccount(data)` — Owner/Admin, buat Customer + baris Customer
- `resetPin(userId, newPin)` — Owner only
- `toggleUserActive(userId)` — Owner only
- `listUsers(filter?)` — Owner only

### `customer.actions.ts`
- `createCustomer(data)` — Owner/Admin
- `updateCustomer(id, data)` — Owner/Admin
- `listCustomers(search?)` — Owner/Admin
- `getCustomerById(id)` — Owner/Admin, atau Customer untuk dirinya sendiri

### `pet.actions.ts`
- `createPet(data)` — Owner/Admin
- `updatePet(id, data)` — Owner/Admin
- `listPetsByCustomer(customerId)` — scoped sesuai role
- `getPetById(id)` — scoped sesuai role

### `appointment.actions.ts`
- `createAppointment(data)` — Admin/Owner
- `updateAppointmentStatus(id, status)` — Dokter (hanya miliknya, hanya ke IN_PROGRESS/COMPLETED), Admin/Owner (semua status termasuk CANCELLED/CONFIRMED/CHECKED_IN)
- `listAppointments(filter)` — scoped sesuai role (Dokter: hanya miliknya, Customer: hanya pet miliknya)
- `getAppointmentById(id)` — scoped

### `medical-record.actions.ts`
- `createOrUpdateMedicalRecord(appointmentId, data)` — Dokter only, dan hanya untuk appointment miliknya sendiri
- `getMedicalRecordByAppointment(appointmentId)` — scoped sesuai role

### `pet-hotel.actions.ts`
- `createHotelRoom(data)` — Owner/Admin
- `bookPetHotel(data)` — Owner/Admin
- `checkInPetHotel(id)` — Owner/Admin
- `checkOutPetHotel(id)` — Owner/Admin
- `extendStay(id, newCheckOutDate)` — Owner/Admin
- `cancelBooking(id)` — Owner/Admin
- `listPetHotels(filter)` — scoped (Dokter: read-only semua)

### `service.actions.ts`
- `createService(data)` — Owner/Admin
- `updateService(id, data)` — Owner/Admin
- `listServices()` — semua role internal (untuk referensi saat generate invoice)

### `invoice.actions.ts`
- `generateInvoiceFromAppointment(appointmentId, items)` — Owner/Admin
- `generateInvoiceFromPetHotel(petHotelId, items)` — Owner/Admin
- `markInvoiceAsPaid(invoiceId)` — Owner/Admin
- `listInvoices(filter)` — scoped (Customer: hanya miliknya)
- `getInvoiceById(id)` — scoped

Setiap function di atas WAJIB diawali pemanggilan `requireRole([...])` dari `lib/auth-guard.ts` sebelum melakukan operasi database.

---

## 9. SKEMA VALIDASI ZOD

Contoh pola yang harus diikuti untuk semua modul (letakkan di `lib/validations/`):

```ts
// lib/validations/user.ts
import { z } from "zod";

export const createStaffSchema = z.object({
  username: z.string().min(3).max(30).regex(/^[a-zA-Z0-9_]+$/),
  pin: z.string().min(4).max(6).regex(/^[0-9]+$/),
  role: z.enum(["ADMIN", "DOKTER"]),
  name: z.string().min(2).max(100),
  phone: z.string().optional(),
});

export const createCustomerAccountSchema = z.object({
  username: z.string().min(3).max(30).regex(/^[a-zA-Z0-9_]+$/),
  pin: z.string().min(4).max(6).regex(/^[0-9]+$/),
  name: z.string().min(2).max(100),
  phone: z.string().optional(),
  address: z.string().optional(),
});
```

Pola yang sama diterapkan untuk `pet.ts`, `appointment.ts`, `medical-record.ts`, `pet-hotel.ts`, `service.ts`, `invoice.ts` — sesuai field yang tercantum di bagian 7. Semua Server Action WAJIB memanggil `.parse()` atau `.safeParse()` di awal sebelum eksekusi ke Prisma.

---

## 10. MODUL: USER MANAGEMENT

- Halaman `/users` (Owner only): list semua user, filter by role, toggle aktif/nonaktif, tombol reset PIN.
- Form tambah Admin/Dokter: username, PIN awal, nama, no. telp, role (dropdown Admin/Dokter saja).
- Form tambah Customer (bisa juga diakses dari halaman `/customers` oleh Admin): username, PIN awal, nama, no telp, alamat.
- Reset PIN: input PIN baru, langsung di-hash ulang.
- Nonaktifkan akun: `is_active = false` — user tidak bisa login lagi tapi datanya tetap ada (soft delete, bukan hard delete).

---

## 11. MODUL: CUSTOMER & PET

- Halaman `/customers` (Owner/Admin): list, search by nama/username, detail customer beserta daftar pet miliknya.
- Halaman `/pets` (Owner/Admin): list semua pet, filter by customer.
- Customer login hanya melihat halaman `/pets` berisi pet miliknya sendiri (query di-filter otomatis via `session.user.id` → `customer.id`).
- Form tambah/edit pet: nama, spesies, ras (opsional), tanggal lahir (opsional), gender, catatan.

---

## 12. MODUL: APPOINTMENT

- Status flow: `PENDING → CONFIRMED → CHECKED_IN → IN_PROGRESS → COMPLETED`, atau `CANCELLED` di titik manapun sebelum `COMPLETED`.
- Halaman `/appointments`:
  - Owner/Admin: lihat semua, buat baru (pilih pet, dokter, tanggal/jam, keluhan awal), ubah status manapun.
  - Dokter: lihat hanya appointment miliknya (`doctorId = session.user.id`), tombol untuk mulai pemeriksaan (`IN_PROGRESS`) dan selesaikan (`COMPLETED`).
  - Customer: lihat hanya appointment untuk pet miliknya, read-only, tidak ada tombol aksi.
- Validasi bisnis: tidak boleh membuat appointment baru untuk dokter yang sama pada jam yang bentrok (opsional tapi disarankan sebagai pengecekan sederhana di Server Action).

---

## 13. MODUL: MEDICAL RECORD

- Halaman `/medical-records` atau tab di dalam detail appointment yang berstatus `IN_PROGRESS`/`COMPLETED`.
- Dokter mengisi: vital sign, diagnosis, treatment, prescription, notes — hanya untuk appointment miliknya sendiri dan hanya bisa dibuat/diubah selama appointment belum `COMPLETED` (atau sesuai kebijakan: bisa diedit sebelum invoice digenerate).
- Owner/Admin: lihat saja, tidak ada tombol edit.
- Customer: lihat saja, hanya untuk pet miliknya, tidak ada tombol edit.

---

## 14. MODUL: PET HOTEL

- Halaman `/pet-hotel`:
  - Kelola kamar (`hotel_rooms`): Owner/Admin bisa tambah kamar baru dengan nomor & tipe kandang.
  - Booking: pilih pet, kamar yang `AVAILABLE`, tanggal check-in, catatan makanan/jadwal makan/obat.
  - Check-in: ubah status `BOOKED → CHECKED_IN`, otomatis ubah `hotel_rooms.status → OCCUPIED`.
  - Selama menginap: status `STAYING`.
  - Check-out: ubah status `→ CHECKED_OUT`, otomatis ubah `hotel_rooms.status → AVAILABLE`, isi `check_out_date`.
  - Perpanjang: update `check_out_date` tanpa mengubah status.
  - Cancel: hanya bisa dilakukan sebelum check-in, ubah status `→ CANCELLED`, kamar tetap/kembali `AVAILABLE`.
- Dokter: hanya bisa melihat (view-only), tidak ada tombol aksi apapun di halaman ini.
- Widget dashboard pet hotel: Total Kandang, Kandang Terisi, Kandang Kosong, Check In Hari Ini, Check Out Hari Ini (dihitung dari query `hotel_rooms` dan `pet_hotels` real-time).

---

## 15. MODUL: SERVICES

- Halaman `/services` (Owner/Admin): CRUD data layanan/obat — nama, harga, kategori (treatment, medicine, hotel, dll).
- Data ini menjadi referensi dropdown saat generate invoice (`invoice_items`).

---

## 16. MODUL: BILLING & INVOICE

- Invoice digenerate manual oleh Owner/Admin dari:
  - Appointment yang sudah `COMPLETED` (pilih service/obat yang dipakai dari daftar `services`, isi quantity, harga otomatis terisi dari `services.price` tapi bisa disesuaikan saat itu — disimpan sebagai snapshot di `invoice_items.price`).
  - Pet Hotel yang sudah/sedang `CHECKED_OUT` (hitung biaya per hari x jumlah hari menginap, plus service tambahan jika ada).
- Setelah invoice dibuat, `status = UNPAID`. Owner/Admin menekan tombol "Tandai Lunas" untuk ubah ke `PAID`.
- Dokter: tidak ada akses ke modul ini sama sekali (menu tidak muncul, dan Server Action menolak jika dipanggil).
- Customer: halaman `/invoices` hanya menampilkan invoice miliknya sendiri, read-only, bisa lihat rincian item.

---

## 17. MODUL: DASHBOARD

Widget per role (data real-time dari query database, bukan hardcode):

| Role | Widget |
|---|---|
| **Owner** | Total Customer, Total Pet, Appointment Hari Ini, Pet Hotel Hari Ini, Pendapatan Hari Ini, Pendapatan Bulanan (chart Recharts) |
| **Admin** | Appointment Hari Ini, Pet Hotel (ringkasan kamar), Waiting (appointment status PENDING/CONFIRMED), Checked In |
| **Dokter** | Appointment Hari Ini (miliknya), Pemeriksaan Hari Ini (status IN_PROGRESS/COMPLETED miliknya hari ini) |
| **Customer** | Hewan Saya (jumlah pet), Appointment (terdekat/aktif), Pet Hotel (jika sedang menginap), Invoice (jumlah UNPAID) |

Pendapatan dihitung dari `SUM(invoices.total_amount)` dengan filter `status = PAID` dan `created_at` sesuai rentang (hari ini/bulan ini).

---

## 18. MODUL: REPORT

- Halaman `/reports` (Owner only).
- Laporan sederhana (tidak perlu report builder generic):
  - Laporan pendapatan per periode (harian/bulanan), bisa difilter tanggal.
  - Laporan jumlah appointment per status.
  - Laporan okupansi pet hotel.
- Tampilkan sebagai tabel + chart Recharts. Tidak perlu export PDF/Excel di MVP kecuali diminta terpisah.

---

## 19. MODUL: SETTING

- Halaman `/settings` (Owner only).
- Minimal berisi: pengaturan info klinik (nama, alamat, no telp — bisa disimpan di tabel `settings` sederhana key-value jika dibutuhkan; jika ditambahkan, WAJIB update bagian 7 dokumen ini dulu).
- Untuk MVP, jika tidak ada kebutuhan konkret, cukup buat halaman placeholder dengan info dasar akun Owner yang login.

---

## 20. FORMAT RESPONSE & ERROR HANDLING

Semua Server Action mengembalikan bentuk konsisten:

```ts
type ActionResult<T> =
  | { success: true; data: T }
  | { success: false; error: string };
```

Aturan:
- Jangan `throw` error mentah ke client — selalu tangkap dengan try/catch dan kembalikan `{ success: false, error: "pesan yang aman ditampilkan" }`.
- Error validasi Zod: kembalikan pesan field pertama yang gagal, bahasa Indonesia yang jelas.
- Error otorisasi (`requireRole` gagal): kembalikan pesan generik **"Anda tidak memiliki akses untuk aksi ini"**, jangan expose detail role yang dibutuhkan.
- Error database (constraint unik, dll): tangani khusus (mis. username sudah dipakai → pesan **"Username sudah digunakan"**).

---

## 21. SEED DATA AWAL

Karena tidak ada self-registration, sistem butuh **satu akun Owner pertama** agar bisa login pertama kali. Buat `prisma/seed.ts`:

- 1 akun Owner: `username: "owner"`, PIN default (mis. `"123456"`, WAJIB diganti setelah first login).
- (Opsional untuk memudahkan testing) 1 akun Admin, 1 akun Dokter, 1 akun Customer beserta 1 pet contoh.
- Beberapa `services` contoh (konsultasi, vaksin, grooming, kandang per hari, dst).
- Beberapa `hotel_rooms` contoh (mis. 5 kandang dengan tipe berbeda).

Jalankan via `npx prisma db seed` (daftarkan script ini di `package.json`).

> Catatan keamanan: PIN default Owner di seed data HARUS diganti segera setelah aplikasi pertama kali dijalankan di lingkungan nyata.

---

## 22. ROUTING & HALAMAN

| Route | Akses | Deskripsi |
|---|---|---|
| `/login` | Publik | Form login Username + PIN |
| `/dashboard` | Semua role (login) | Widget sesuai role |
| `/users` | Owner | Kelola akun Admin/Dokter/Customer |
| `/customers` | Owner, Admin | List & detail customer |
| `/pets` | Owner, Admin, Customer (scoped) | List & detail pet |
| `/appointments` | Semua role (scoped) | List & aksi sesuai role |
| `/appointments/[id]` | Semua role (scoped) | Detail + medical record terkait |
| `/medical-records` | Dokter, Owner(RO), Admin(RO), Customer(RO, scoped) | Biasanya diakses via detail appointment |
| `/pet-hotel` | Owner, Admin, Dokter(RO) | Booking, check-in/out, status kamar |
| `/services` | Owner, Admin | CRUD layanan/obat |
| `/billing` atau `/invoices` | Owner, Admin, Customer(RO, scoped) | Generate & lihat invoice |
| `/reports` | Owner | Laporan |
| `/settings` | Owner | Pengaturan |

---

## 23. UI/UX GUIDELINES

- Gunakan layout sidebar (kiri) + konten utama, umum untuk dashboard admin — komponen shadcn/ui: `Sidebar` custom, `Card`, `Table`, `Dialog`/`Sheet` untuk form tambah/edit, `Badge` untuk status (warna berbeda per status appointment/invoice/pet hotel), `Toast`/`Sonner` untuk feedback aksi.
- Menu sidebar dirender secara dinamis berdasarkan `session.user.role` — jangan hardcode semua menu lalu disembunyikan dengan CSS (gunakan conditional rendering di server component).
- Warna badge status yang disarankan (konsisten di seluruh app):
  - `PENDING`/`BOOKED`/`UNPAID` → kuning/abu
  - `CONFIRMED`/`CHECKED_IN` → biru
  - `IN_PROGRESS`/`STAYING` → oranye
  - `COMPLETED`/`CHECKED_OUT`/`PAID` → hijau
  - `CANCELLED` → merah
- Semua tabel list wajib ada pagination sederhana (bisa client-side untuk MVP karena skala single-clinic tidak besar).
- Form wajib menampilkan pesan error per-field dari hasil validasi Zod, bukan hanya toast generik.

---

## 24. TESTING CHECKLIST PER ROLE

Setelah implementasi selesai, uji manual berikut untuk tiap role (login satu-satu):

**Owner:**
- [ ] Bisa buat akun Admin, Dokter, Customer
- [ ] Bisa reset PIN & nonaktifkan akun
- [ ] Bisa akses semua menu tanpa terkecuali
- [ ] Dashboard menampilkan angka yang benar (cocokkan manual dengan data di database)

**Admin:**
- [ ] Bisa buat akun Customer, TIDAK bisa buat Admin/Dokter (baik dari UI maupun coba panggil Server Action langsung)
- [ ] Bisa registrasi customer & pet
- [ ] Bisa booking appointment & pet hotel
- [ ] Bisa generate invoice & tandai PAID
- [ ] TIDAK bisa akses `/users`, `/reports`, `/settings`

**Dokter:**
- [ ] Hanya melihat appointment miliknya sendiri, bukan milik dokter lain
- [ ] Bisa isi medical record hanya untuk appointment miliknya
- [ ] TIDAK bisa mengubah status appointment dokter lain
- [ ] TIDAK bisa akses billing/invoice sama sekali (menu tidak muncul + Server Action ditolak)
- [ ] Pet Hotel hanya bisa lihat, tidak ada tombol aksi

**Customer:**
- [ ] Hanya melihat pet, appointment, pet hotel, invoice miliknya sendiri — coba akses ID milik customer lain langsung via URL harus ditolak
- [ ] Tidak ada tombol create/edit di manapun kecuali yang memang diizinkan (jika ada)
- [ ] Tidak bisa membuat akun sendiri (tidak ada halaman register)

**Umum:**
- [ ] Login dengan username/PIN salah → pesan error generik, tidak bocor info
- [ ] Akun nonaktif tidak bisa login
- [ ] Semua Server Action yang diuji manual via panggilan langsung (bukan lewat UI) tetap menolak akses tidak sah

---

## 25. RULE ABSOLUT (TIDAK BOLEH DILANGGAR)

1. Tidak ada registrasi mandiri / self-service signup.
2. Tidak ada login menggunakan email — hanya Username + PIN.
3. PIN wajib di-hash (bcrypt), tidak pernah disimpan atau ditampilkan plain text.
4. Hanya OWNER yang dapat membuat akun Admin Operasional dan Dokter.
5. Hanya ADMIN OPERASIONAL yang dapat membuat akun Customer.
6. CUSTOMER tidak dapat membuat akun sendiri.
7. OWNER memiliki akses penuh terhadap seluruh sistem.
8. Reset PIN, perubahan role, dan penonaktifan akun hanya dapat dilakukan oleh OWNER.
9. DOKTER tidak dapat mengubah data billing/pembayaran dalam kondisi apapun.
10. CUSTOMER hanya dapat mengakses/melihat data miliknya sendiri (scoped query berdasarkan relasi `user_id` → `customer_id`).
11. Tidak ada `clinic_id` atau struktur multi-tenant di manapun dalam sistem.
12. Semua mutasi data (create/update/delete) menggunakan Next.js Server Actions, bukan REST API custom.
13. Semua input form wajib divalidasi menggunakan Zod sebelum masuk ke database.
14. Setiap Server Action wajib memvalidasi ulang role pengguna di sisi server (`requireRole`), tidak boleh hanya mengandalkan penyembunyian UI/middleware.
15. Tidak boleh ada penambahan tabel, field, enum, atau endpoint di luar yang tercantum di dokumen ini tanpa memperbarui dokumen ini terlebih dahulu.

---

## 26. TARGET MVP (SCOPE TETAP)

1. Login Username + PIN
2. Dashboard (per role)
3. User Management
4. Customer Management
5. Pet Management
6. Appointment Management
7. Medical Record
8. Pet Hotel Management
9. Services Management
10. Billing & Invoice
11. Report
12. Setting

Tidak ada fitur di luar daftar ini yang ditambahkan tanpa update dokumen ini terlebih dahulu.

---

## 27. YANG SENGAJA TIDAK ADA DI MVP (OUT OF SCOPE)

- Multi-clinic / SaaS
- Self-registration
- Email/password login
- Notifikasi email/SMS/WhatsApp otomatis
- Payment gateway online
- Mobile app native
- Report builder generic/custom
- Permission granular per-fitur (cukup role-based sederhana)
- Export PDF/Excel otomatis (kecuali diminta terpisah di luar dokumen ini)
- Multi-bahasa (UI Bahasa Indonesia saja)

---

## 28. CATATAN DEPLOYMENT

- Dev: SQLite (`file:./dev.db`), cukup untuk development di Codespace.
- Production: migrasi ke PostgreSQL — cukup ubah `provider` di `schema.prisma` dan `DATABASE_URL`, jalankan ulang `prisma migrate deploy`.
- Pastikan `NEXTAUTH_SECRET` di production berbeda dari dev dan disimpan sebagai secret (bukan di-commit ke repo).
- File `.env` WAJIB masuk `.gitignore`. Hanya `.env.example` yang di-commit.

---

## 29. ROADMAP IMPLEMENTASI (REFERENSI FASE)

Dokumen ini dipecah menjadi 10 fase implementasi berurutan untuk dieksekusi via Copilot Chat:

1. Setup Project
2. Database Schema
3. Authentication
4. User Management
5. Customer & Pet Management
6. Appointment
7. Medical Record
8. Pet Hotel
9. Billing & Invoice (termasuk Services)
10. Dashboard, Report & Setting

Setiap fase harus merujuk balik ke dokumen ini secara spesifik (nomor bagian terkait) agar implementasi tidak menyimpang dari spesifikasi.

---

**Dokumen ini adalah rujukan utama.** Setiap penambahan fitur, perubahan role, atau perubahan struktur data harus diperbarui di sini dulu sebelum diimplementasikan di kode.
