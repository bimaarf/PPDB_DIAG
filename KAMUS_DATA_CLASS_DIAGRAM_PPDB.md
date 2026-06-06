# Kamus Data Class Diagram PPDB

Dokumen ini dipakai sebagai acuan class diagram untuk proses utama PSB/PPDB:
autentikasi akun, pengisian formulir siswa, verifikasi berkas oleh admin/panitia,
pengisian hasil seleksi oleh admin/panitia, notifikasi, dan template WhatsApp.

Dokumen ini tidak memuat fitur yang disembunyikan atau method internal yang tidak
dipakai route utama.

## Aturan Utama Visibility Hasil

Siswa tetap melihat status berkas pendaftaran. Hasil kelulusan tidak ditampilkan ke
siswa selama periode pendaftaran belum dipublish.

| Kondisi | Data yang Terlihat ke Siswa |
| --- | --- |
| `tb_period.is_published = false` | `validation_status` / status berkas tetap tampil. Field hasil final `selection_type`, `value`, dan `status` dikirim `null`. |
| `tb_period.is_published = true` | `validation_status` dan hasil final boleh tampil sesuai data `tb_results`. |
| Admin/Panitia | Dapat melihat dan mengelola status berkas serta hasil seleksi untuk kebutuhan operasional. |

## Role Aktif

| Role | Kode | Fungsi di Flow PPDB |
| --- | --- | --- |
| Super Admin | SA | Mengelola periode PPDB, user, role user, template WhatsApp, verifikasi, hasil seleksi, dan broadcast. |
| Administrator / Panitia | ADM | Mengelola pertanyaan, melihat submission, memverifikasi berkas, dan mengisi hasil seleksi sesuai akses route/UI. |
| User / Siswa | USR | Registrasi, login, mengisi formulir, upload berkas, melihat status berkas, dan melihat hasil hanya jika periode dipublish. |

## Class Utama

| Class Diagram | Model Laravel | Tabel / Sumber | Fungsi |
| --- | --- | --- | --- |
| `User` | `App\Models\User` | `users` | Akun login, profil siswa/admin, status akun, nomor WhatsApp, dan token Sanctum. |
| `Role` | `App\Models\Role` | `roles` | Data role teknis seperti `super admin`, `administrator`, dan `user`. |
| `RoleUser` | `App\Models\RoleUser` | `role_user` | Pivot pemasangan role ke user. |
| `Otp` | `App\Models\Otp` | `otps` | OTP untuk registrasi, login, dan reset password. |
| `UserLoginDetail` | `App\Models\UserLoginDetail` | `user_login_details` | Riwayat perangkat/browser saat login. |
| `PPDBPeriod` | `App\Models\Form\Period` | `tb_period` | Periode pendaftaran, status aktif, publish hasil, dan archive. |
| `PPDBQuestion` | `App\Models\Form\Questions` | `tb_questions` | Pertanyaan/form field PPDB per periode. |
| `PPDBAnswer` | `App\Models\Form\Answers` | `tb_answers` | Jawaban dan file berkas siswa, digroup dengan `submission_id`. |
| `PPDBResult` | `App\Models\Form\Result` | `tb_results` | Status verifikasi berkas dan hasil seleksi untuk satu submission. |
| `Notification` | `App\Models\Notification` | `tb_notification` | Notifikasi akun, security, registrasi, sistem, dan WhatsApp. |
| `NotificationWhatsAppMessage` | `App\Models\NotificationWhatsAppMessage` | `notification_whatsapp_messages` | Template pesan WhatsApp yang dipakai helper notifikasi. |

WhatsApp gateway dan Socket.IO adalah integrasi eksternal. Keduanya cukup digambar
sebagai dependency/service, bukan class database utama.

## Relasi Class

| Relasi | Kardinalitas | Keterangan |
| --- | --- | --- |
| `User` - `Role` | many-to-many | Melalui `RoleUser` / tabel `role_user`. |
| `User` - `Otp` | one-to-many | Satu user dapat punya banyak OTP, tetapi OTP aktif dibatasi per tipe. |
| `User` - `UserLoginDetail` | one-to-many logical | Detail login menyimpan `user_id`, perangkat, browser, IP, email, dan nama. |
| `User` - `Notification` | one-to-many | Notifikasi personal milik user. |
| `PPDBPeriod` - `PPDBQuestion` | one-to-many | Satu periode memiliki banyak pertanyaan. |
| `PPDBPeriod` - `PPDBAnswer` | one-to-many | Satu periode memiliki banyak jawaban/submission. |
| `PPDBQuestion` - `PPDBAnswer` | one-to-many | Satu pertanyaan dapat dijawab oleh banyak submission. |
| `User` - `PPDBAnswer` | one-to-many | Satu siswa dapat memiliki banyak row jawaban dalam satu atau lebih submission. |
| `PPDBAnswer` - `PPDBResult` | one-to-one logical | `tb_results.submission_answers` mengacu ke `tb_answers.submission_id`. |
| `NotificationWhatsAppMessage` - `Notification` | logical reference | Template WA dipakai helper untuk membentuk pesan dan log notifikasi. |

## Detail Attribute Class

### User

| Attribute | Type | Rule | Keterangan |
| --- | --- | --- | --- |
| `id` | bigint | PK | Identitas user. |
| `name` | string | required | Nama user. |
| `phone_number` | string/numeric | required | Nomor WhatsApp untuk OTP/notifikasi. |
| `email` | string | unique | Email login. |
| `password` | string | masked, hashed | Password tersimpan dalam hash. |
| `avatar` | string nullable | path/URL | Foto profil. |
| `status` | boolean/int | active/inactive | Status akun. |
| `otp_verified` | boolean | default false | Status verifikasi OTP akun. |
| `email_verified_at` | timestamp nullable | nullable | Waktu verifikasi email jika dipakai. |
| `last_online_at` | timestamp nullable | nullable | Waktu online terakhir. |
| `created_at`, `updated_at` | timestamp | auto | Audit waktu. |

### Role

| Attribute | Type | Rule | Keterangan |
| --- | --- | --- | --- |
| `id` | bigint | PK | Identitas role. |
| `name` | string | unique | Nama teknis role. |
| `display_name` | string nullable | nullable | Nama tampilan role. |
| `description` | string nullable | nullable | Deskripsi role. |
| `created_at`, `updated_at` | timestamp | auto | Audit waktu. |

### RoleUser

| Attribute | Type | Rule | Keterangan |
| --- | --- | --- | --- |
| `role_id` | bigint | FK `roles.id` | Role yang dipasang. |
| `user_id` | bigint | FK/logical ke `users.id` | User penerima role. |
| `user_type` | string | Laratrust morph type | Tipe model user. |

### Otp

| Attribute | Type | Rule | Keterangan |
| --- | --- | --- | --- |
| `id` | bigint | PK | Identitas OTP. |
| `user_id` | bigint | FK/logical ke `users.id` | Pemilik OTP. |
| `otp_code` | string | 6 digit | Kode OTP. |
| `type` | string | `registration`, `login`, `password_reset` | Jenis OTP. |
| `expires_at` | timestamp | required | Masa berlaku OTP. |
| `is_used` | boolean | default false | Status penggunaan OTP. |
| `created_at`, `updated_at` | timestamp | auto | Audit waktu. |

### UserLoginDetail

| Attribute | Type | Rule | Keterangan |
| --- | --- | --- | --- |
| `id` | bigint | PK | Identitas detail login. |
| `user_id` | bigint nullable | nullable | User yang login. |
| `platform`, `platform_version` | string nullable | nullable | OS/perangkat. |
| `browser`, `browser_version` | string nullable | nullable | Browser. |
| `is_mobile`, `is_desktop` | boolean/int | nullable | Tipe device. |
| `ip_address` | string nullable | nullable | IP login. |
| `email`, `name` | string nullable | snapshot | Snapshot identitas login. |

### PPDBPeriod

| Attribute | Type | Rule | Keterangan |
| --- | --- | --- | --- |
| `id` | bigint | PK | Identitas periode. |
| `key` | string | unique | Key periode untuk URL/lookup. |
| `title` | string | required | Nama periode PPDB. |
| `description` | longText nullable | nullable | Deskripsi periode. |
| `status` | tinyint/int | `1` aktif, `0` tidak aktif | Penanda periode bisa dipakai. |
| `is_published` | tinyint/int | `1` published, `0` belum | Penentu hasil final boleh tampil ke siswa. |
| `is_archived` | tinyint/int | `1` archived | Penanda periode arsip. |
| `created_at`, `updated_at` | timestamp | auto | Audit waktu. |

### PPDBQuestion

| Attribute | Type | Rule | Keterangan |
| --- | --- | --- | --- |
| `id` | bigint | PK | Identitas pertanyaan. |
| `period_id` | bigint nullable | FK/logical ke `tb_period.id` | Periode pemilik pertanyaan. |
| `author_id` | bigint | FK/logical ke `users.id` | Admin/panitia pembuat. |
| `question` | text | required | Teks pertanyaan. |
| `label` | string nullable | nullable | Label section/page. |
| `type` | enum/string | `radio`, `checkbox`, `text`, `file`, `multiple_file` | Tipe input. |
| `options` | json nullable | array | Pilihan untuk radio/checkbox. |
| `file_types` | string/json nullable | nullable | Ekstensi file yang diizinkan. |
| `page` | integer | default 1 | Halaman form. |
| `sort_order` | integer | default 0 | Urutan pertanyaan. |
| `is_required` | boolean | default false | Penanda wajib. |
| `created_at`, `updated_at` | timestamp | auto | Audit waktu. |

### PPDBAnswer

| Attribute | Type | Rule | Keterangan |
| --- | --- | --- | --- |
| `id` | bigint | PK | Identitas row jawaban. |
| `submission_id` | uuid/string | grouping key | ID satu paket pendaftaran. |
| `user_id` | bigint nullable | FK/logical ke `users.id` | Siswa pengisi. |
| `period_id` | bigint nullable | FK/logical ke `tb_period.id` | Periode pendaftaran. |
| `question_id` | bigint nullable | FK/logical ke `tb_questions.id` | Pertanyaan asal. |
| `question` | text nullable | snapshot | Snapshot teks pertanyaan. |
| `label` | string nullable | snapshot | Snapshot label. |
| `page` | integer | snapshot | Snapshot halaman. |
| `sort_order` | integer | snapshot | Snapshot urutan. |
| `answer` | string nullable | nullable | Jawaban teks/pilihan. |
| `file_path` | longText nullable | nullable | Path file upload. |
| `type` | enum/string | sesuai question | Tipe jawaban. |
| `options` | json nullable | snapshot | Snapshot opsi. |
| `created_at`, `updated_at` | timestamp | auto | Audit waktu. |

### PPDBResult

| Attribute | Type | Rule | Keterangan |
| --- | --- | --- | --- |
| `id` | bigint | PK | Identitas result. |
| `submission_answers` | string/uuid | logical FK ke `tb_answers.submission_id` | Submission yang diverifikasi. |
| `is_approve` | boolean nullable | `true` diterima, `false` dikembalikan | Status verifikasi berkas. Ini tetap menjadi dasar status berkas siswa. |
| `selection_type` | string nullable | nullable | Tahap/jenis seleksi final. |
| `value` | string nullable | nullable | Nilai/keterangan hasil seleksi. |
| `status` | boolean nullable | `true` lulus, `false` tidak lulus | Status kelulusan final. Disembunyikan dari siswa jika periode belum publish. |
| `created_at`, `updated_at` | timestamp | auto | Audit waktu. |

### Notification

| Attribute | Type | Rule | Keterangan |
| --- | --- | --- | --- |
| `id` | bigint | PK | Identitas notifikasi. |
| `key` | string | generated | Key unik notifikasi. |
| `label` | string | category | Contoh: `account`, `security`, `registration`, `whatsapp`, `[system]`, `user management`. |
| `title` | string | required | Judul notifikasi. |
| `message` | longText | text/json | Isi notifikasi atau log status WhatsApp. |
| `user_id` | bigint nullable | FK/logical ke `users.id` | Target user. |
| `is_read` | boolean | default false | Status baca. |
| `created_at`, `updated_at` | timestamp | auto | Audit waktu. |

### NotificationWhatsAppMessage

| Attribute | Type | Rule | Keterangan |
| --- | --- | --- | --- |
| `id` | bigint | PK | Identitas template. |
| `code` | string | unique | Kode template, misalnya OTP, submission baru, verifikasi berkas, atau hasil seleksi. |
| `label` | string | required | Nama template. |
| `message` | longText | required | Isi pesan WhatsApp. |
| `placeholder` | string nullable | nullable | Placeholder yang dapat diganti helper. |
| `created_at`, `updated_at` | timestamp | auto | Audit waktu. |

## Operasi Aktif per Class

| Area | Controller | Method Aktif | Class Terkait |
| --- | --- | --- | --- |
| Auth | `API\AuthController` | `register`, `login`, `verifyOtp`, `resendOtp`, `requestLoginOtp`, `requestPasswordResetOtp`, `changePassword`, `changePhoneNumber`, `logout`, `isAuthenticated`, `updateLastOnline` | `User`, `Otp`, `UserLoginDetail`, `Notification` |
| User | `UserController` | `search`, `getByUsername`, `all`, `index`, `view`, `store`, `updatePassword`, `update`, `delete` | `User`, `Role`, `RoleUser`, `Notification` |
| Periode | `Form\PeriodController` | `all`, `index`, `view`, `store`, `update`, `delete`, `broadcastToUserSubmitted` | `PPDBPeriod`, `PPDBAnswer`, `NotificationWhatsAppMessage` |
| Pertanyaan | `Form\QuestionController` | `show`, `view`, `find`, `store`, `update`, `updateOrder`, `destroy` | `PPDBQuestion`, `PPDBPeriod` |
| Jawaban/Berkas | `Form\AnswerController` | `getGroupedAnswers`, `getBySubmission`, `getGroupedAnswersPublic`, `getStatusTotals`, `getSubmissionsByPeriod`, `getSubmissionsPerDayByPeriod`, `getSubmissionsPerDayDetail`, `submitAnswers`, `updateAnswers`, `deleteRespondentBySubmission` | `PPDBAnswer`, `PPDBQuestion`, `PPDBPeriod`, `PPDBResult`, `NotificationWhatsAppMessage` |
| Verifikasi/Hasil | `Form\ResultController` | `listPeriods`, `show`, `verify`, `update`, `download`, `downloadMultiple` | `PPDBResult`, `PPDBAnswer`, `PPDBPeriod`, `NotificationWhatsAppMessage` |
| Notifikasi | `NotificationController` | `index`, `view`, `markAsRead`, `markAllAsRead`, `update`, `delete`, `resendNotification` | `Notification`, `User` |
| Template WhatsApp | `WhatsAppNotificationController` | `index`, `store`, `show`, `update`, `destroy` | `NotificationWhatsAppMessage` |

## Matriks Akses Flow Utama

| Flow | SA | ADM | USR |
| --- | --- | --- | --- |
| Registrasi, login, OTP, reset password | Own account | Own account | Own account |
| Kelola user | Full | Create/read sesuai route/UI | Profil dan password sendiri |
| Kelola role user | Full lewat update user | Tidak | Tidak |
| Kelola periode PPDB | Create/read/update/delete/broadcast | Read | Read periode tersedia |
| Kelola pertanyaan PPDB | Create/read/update/delete/reorder | Create/read/update/delete/reorder | Read untuk isi form |
| Submit/update formulir | Bisa operasional jika diberi akses | Bisa operasional jika diberi akses | Submission sendiri |
| Lihat daftar submission | Semua | Semua sesuai UI/API | Submission sendiri |
| Verifikasi berkas | Ya | Ya sesuai UI/API | Tidak |
| Isi hasil seleksi | Ya | Ya sesuai UI/API | Tidak |
| Lihat status berkas | Semua | Semua sesuai UI/API | Submission sendiri |
| Lihat hasil kelulusan | Semua | Semua sesuai UI/API | Hanya jika `is_published=true` |
| Template WhatsApp | CRUD | CRUD sesuai route | Tidak |
| Notifikasi | Sesuai scope role dan owner | Sesuai scope role dan owner | Notifikasi sendiri |

## Route Utama yang Menjadi Acuan

| Flow | Route |
| --- | --- |
| Register | `POST /api/register` |
| Login password | `POST /api/login` |
| Request OTP login | `POST /api/request-login-otp`, `POST /api/login/otp` |
| Verify OTP | `POST /api/verify-otp` |
| Reset password OTP | `POST /api/request-password-reset-otp`, `POST /api/change-password` |
| Ganti nomor sebelum OTP verified | `POST /api/change-phone-number` |
| Auth check | `GET /api/is-authenticated` |
| Period PPDB | `GET /api/form/period`, `GET /api/form/period/all`, `GET /api/form/period/{key}`, `POST /api/form/period`, `POST /api/form/period/{key}`, `DELETE /api/form/period/{id}` |
| Pertanyaan PPDB | `GET /api/form/question/find/{periodKey}`, `POST /api/form/question`, `POST /api/form/question/update/{periodId}`, `POST /api/form/question/order`, `DELETE /api/form/question/{id}` |
| Submit formulir | `POST /api/form/answer` |
| List/detail submission | `GET /api/form/answer/group/respondent`, `GET /api/form/answer/group/respondent/{submissionId}` |
| Update/delete submission | `POST /api/form/answer/update-submission/{submissionId}`, `POST /api/form/answer/respondent/delete/{submissionId}` |
| Statistik submission | `GET /api/form/answer/status-totals`, `GET /api/form/answer/group/respondent/period`, `GET /api/form/answer/group/respondent/period/perday`, `GET /api/form/answer/group/respondent/perday/detail/{periodId}` |
| Verifikasi berkas | `POST /api/respondent/result/verify/{submissionId}` |
| Update hasil seleksi | `PUT /api/respondent/result/upadte/{submissionId}` |
| Detail result | `GET /api/respondent/result/{submissionId}` |
| Notifikasi | `GET /api/notifications`, `GET /api/notifications/{id}`, `POST /api/notifications/mark-read/{id}`, `POST /api/notifications/mark-all`, `POST /api/notifications/{id}`, `DELETE /api/notifications/{id}`, `POST /api/notifications/resend/{id}` |
| Template WhatsApp | `GET /api/whatsapp-notifications`, `POST /api/whatsapp-notifications`, `GET /api/whatsapp-notifications/{id}`, `PUT /api/whatsapp-notifications/{id}`, `DELETE /api/whatsapp-notifications/{id}` |

## Helper yang Digambarkan sebagai Dependency

| Helper | Fungsi di Flow PPDB |
| --- | --- |
| `NotificationHelper` | Membuat notifikasi, mengambil template WhatsApp, replace placeholder, mengirim pesan ke gateway, dan menyimpan log notifikasi. |
| `StatusHelper` | Menghasilkan status berkas/hasil berdasarkan `PPDBResult`, role user, dan `PPDBPeriod.is_published`. |
| `SocketHelper` | Mengirim event realtime untuk submission baru, result verified, result updated, respondent deleted, dan perubahan periode. |

## Boundary Class Diagram

1. Class diagram utama cukup memuat class pada bagian "Class Utama" dan dependency helper/service pada bagian helper.
2. Method yang tidak terdaftar pada route utama tidak perlu dibuat sebagai operasi class diagram.
3. Hasil kelulusan siswa wajib mengikuti `PPDBPeriod.is_published`.
4. Field `PPDBResult.is_approve` dipakai untuk status berkas. Field `selection_type`, `value`, dan `status` dipakai untuk hasil seleksi final.
5. Route update hasil seleksi memang tertulis `upadte` di backend: `PUT /api/respondent/result/upadte/{submissionId}`.
