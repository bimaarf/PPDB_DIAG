# Kamus Data Class Diagram PPDB

Dokumen ini membatasi kamus data hanya untuk fitur:

- PPDB
- Account / Users
- Template dan QR WhatsApp
- Notifikasi

Bagian lain di luar daftar tersebut tidak dimasukkan.

## Daftar Role

| Role | Kode | Deskripsi Akses |
| --- | --- | --- |
| Super Admin | SA | Akses penuh untuk konfigurasi PPDB, user, role, status akun, WhatsApp, template pesan, dan notifikasi terkait sistem. |
| Administrator | ADM | Akses operasional untuk melihat data, mengelola pertanyaan PPDB, membuat user, mengelola template pesan, dan memproses notifikasi yang diizinkan. |
| User | USR | Akses personal untuk akun sendiri, formulir PPDB, berkas pendaftaran sendiri, dan notifikasi sendiri. |

## Matriks Akses Fitur

| Fitur | Super Admin | Administrator | User |
| --- | --- | --- | --- |
| PPDB - Periode Pendaftaran | Create, Read, Update, Delete, Broadcast | Read | Read periode/form yang tersedia |
| PPDB - Daftar Pertanyaan | Create, Read, Update, Delete, Reorder | Create, Read, Update, Delete, Reorder | Read pertanyaan untuk mengisi form |
| PPDB - Berkas Pendaftaran Siswa | Read semua, verifikasi, update hasil, delete | Read semua, verifikasi, update hasil | Create/update jawaban sendiri, read hasil sendiri |
| Account / Users | Read semua, create, update profil, update role, suspend/activate, delete | Read user, create user, update data sendiri | Read/update profil sendiri, update password sendiri |
| Template WhatsApp | Create, Read, Update, Delete | Create, Read, Update, Delete | Tidak ada akses kelola |
| QR / Session WhatsApp | Read status/session, scan QR, disconnect, send message | Read status jika UI diberikan, tidak disconnect/send manual | Tidak ada akses |
| Notifikasi | Read notifikasi WhatsApp global + notifikasi sendiri, update/delete yang diizinkan, resend WhatsApp | Read notifikasi sendiri, delete yang diizinkan, resend WhatsApp | Read/mark-read/delete notifikasi sendiri |

## Relasi Class Utama

| Relasi | Kardinalitas | Keterangan |
| --- | --- | --- |
| User - Role | many-to-many | Melalui `role_user`; satu user punya role aktif seperti `super admin`, `administrator`, atau `user`. |
| User - Notification | one-to-many | Satu user dapat memiliki banyak notifikasi personal. |
| Period - Question | one-to-many | Satu periode PPDB memiliki banyak pertanyaan. |
| Period - Answer | one-to-many | Satu periode memiliki banyak jawaban/berkas pendaftaran. |
| Question - Answer | one-to-many | Satu pertanyaan dapat dijawab berkali-kali oleh submission berbeda. |
| User - Answer | one-to-many | Satu user dapat memiliki banyak jawaban PPDB. |
| Answer - Result | one-to-one logical | Satu paket submission di `tb_answers.submission_id` dapat punya satu hasil di `tb_results.submission_answers`. |
| NotificationWhatsAppMessage - Notification | logical reference | Template WA dipakai untuk membentuk isi pesan/notifikasi, tidak ada foreign key langsung. |
| WhatsAppSession - WhatsAppQRCode | one-to-one runtime | QR dan status koneksi dikelola di socket server, bukan tabel database permanen. |

## Class: User

Sumber data utama: tabel `users`.

| Attribute | Type | Key / Rule | Akses Role | Keterangan |
| --- | --- | --- | --- | --- |
| id | bigint | PK | SA/ADM read, USR read own | Identitas user. |
| name | string | unique, required | SA update, ADM/USR update own | Nama akun. |
| avatar | string nullable | file path / URL | SA update, ADM/USR update own | Foto profil user. |
| email | string | unique, required | SA update, ADM/USR update own | Email login dan identitas akun. |
| email_verified_at | timestamp nullable | nullable | SA read, ADM/USR read own | Waktu verifikasi email. |
| password | string | hidden, hashed | SA update, ADM/USR update own password | Password tersimpan dalam bentuk hash. |
| status | boolean | default false | SA update only | Status akun aktif/suspend. |
| otp_verified | boolean | default false | SA read, ADM/USR read own | Status verifikasi OTP. |
| phone_number | bigint/string numeric | required | SA update, ADM/USR update own sesuai aturan UI | Nomor WhatsApp/telepon user. |
| remember_token | string nullable | hidden | system only | Token remember session. |
| last_online_at | timestamp nullable | nullable | SA/ADM read, USR read own | Waktu terakhir user online. |
| created_at | timestamp | auto | SA/ADM read, USR read own | Waktu pembuatan akun. |
| updated_at | timestamp | auto | SA/ADM read, USR read own | Waktu perubahan akun. |

Operasi role:

| Operasi | SA | ADM | USR |
| --- | --- | --- | --- |
| listUsers() | Ya, semua | Ya | Terbatas data sendiri jika dibatasi UI/API |
| createUser() | Ya | Ya | Tidak |
| updateProfile() | Ya, semua | Hanya sendiri | Hanya sendiri |
| updateRole() | Ya | Tidak | Tidak |
| updateStatus() | Ya | Tidak | Tidak |
| deleteUser() | Ya | Tidak | Tidak |
| updatePassword() | Ya, semua | Hanya sendiri | Hanya sendiri |

## Class: Role

Sumber data utama: tabel `roles`.

| Attribute | Type | Key / Rule | Akses Role | Keterangan |
| --- | --- | --- | --- | --- |
| id | bigint | PK | SA/ADM read | Identitas role. |
| name | string | unique | SA manage, ADM read | Nama role teknis, contoh `super admin`, `administrator`, `user`. |
| display_name | string nullable | nullable | SA manage, ADM read | Nama role untuk tampilan. |
| description | string nullable | nullable | SA manage, ADM read | Deskripsi role. |
| created_at | timestamp | auto | SA/ADM read | Waktu role dibuat. |
| updated_at | timestamp | auto | SA/ADM read | Waktu role diperbarui. |

## Class: RoleUser

Sumber data utama: tabel pivot `role_user`.

| Attribute | Type | Key / Rule | Akses Role | Keterangan |
| --- | --- | --- | --- | --- |
| role_id | bigint | FK roles.id, composite PK | SA update | Role yang dipasang ke user. |
| user_id | bigint | composite PK | SA update | User penerima role. |
| user_type | string | composite PK | system | Tipe model user Laratrust. |

## Class: PPDBPeriod

Sumber data utama: tabel `tb_period`.

| Attribute | Type | Key / Rule | Akses Role | Keterangan |
| --- | --- | --- | --- | --- |
| id | bigint | PK | SA/ADM/USR read | Identitas periode. |
| key | string | unique | SA manage, ADM/USR read | Public key periode untuk URL/lookup. |
| title | string | required | SA manage, ADM/USR read | Judul periode PPDB. |
| description | longText nullable | nullable | SA manage, ADM/USR read | Deskripsi periode. |
| status | tinyint nullable | 1 aktif, 0 hidden | SA update, ADM/USR read | Status visibilitas periode. |
| is_published | tinyint nullable | 1 published | SA update, ADM/USR read | Penanda hasil/periode dipublish. |
| is_archived | tinyint nullable | 1 archived | SA update, ADM/USR read terbatas | Penanda arsip periode. |
| created_at | timestamp | auto | SA/ADM/USR read | Waktu periode dibuat. |
| updated_at | timestamp | auto | SA/ADM/USR read | Waktu periode diperbarui. |

Operasi role:

| Operasi | SA | ADM | USR |
| --- | --- | --- | --- |
| listPeriod() / viewPeriod() | Ya | Ya | Ya |
| createPeriod() | Ya | Tidak | Tidak |
| updatePeriod() | Ya | Tidak | Tidak |
| deletePeriod() | Ya | Tidak | Tidak |
| broadcastToSubmittedUsers() | Ya | Tidak | Tidak |

## Class: PPDBQuestion

Sumber data utama: tabel `tb_questions`.

| Attribute | Type | Key / Rule | Akses Role | Keterangan |
| --- | --- | --- | --- | --- |
| id | bigint | PK | SA/ADM/USR read | Identitas pertanyaan. |
| question | text | required | SA/ADM manage, USR read | Teks pertanyaan. |
| label | string nullable | nullable | SA/ADM manage, USR read | Label halaman/section pertanyaan. |
| period_id | bigint nullable | FK tb_period.id, set null on delete | SA/ADM manage, USR read | Relasi ke periode PPDB. |
| type | enum | radio, checkbox, text, file, multiple_file | SA/ADM manage, USR read | Tipe input jawaban. |
| options | json nullable | array option | SA/ADM manage, USR read | Pilihan untuk radio/checkbox. |
| author_id | bigint | FK users.id | SA/ADM create, USR read | Pembuat pertanyaan. |
| page | integer | default 1 | SA/ADM manage, USR read | Halaman pertanyaan. |
| sort_order | integer | default 0 | SA/ADM reorder, USR read | Urutan pertanyaan dalam halaman. |
| is_required | boolean | default false | SA/ADM manage, USR read | Penanda wajib diisi. |
| fileTypes | string/json nullable | daftar ekstensi | SA/ADM manage, USR read | Ekstensi file yang diperbolehkan. |
| created_at | timestamp | auto | SA/ADM/USR read | Waktu pertanyaan dibuat. |
| updated_at | timestamp | auto | SA/ADM/USR read | Waktu pertanyaan diperbarui. |

Operasi role:

| Operasi | SA | ADM | USR |
| --- | --- | --- | --- |
| findQuestionsByPeriod() | Ya | Ya | Ya untuk pengisian form |
| createQuestion() | Ya | Ya | Tidak |
| updateQuestion() | Ya | Ya | Tidak |
| updateQuestionOrder() | Ya | Ya | Tidak |
| deleteQuestion() | Ya | Ya | Tidak |

## Class: PPDBAnswer / StudentRegistrationFile

Sumber data utama: tabel `tb_answers`. Class ini mewakili jawaban dan berkas pendaftaran siswa.

| Attribute | Type | Key / Rule | Akses Role | Keterangan |
| --- | --- | --- | --- | --- |
| id | bigint | PK | SA/ADM read, USR read own | Identitas jawaban. |
| user_id | bigint nullable | FK users.id, set null | SA/ADM read, USR own | User pengisi formulir. |
| question_id | bigint nullable | FK tb_questions.id, set null | SA/ADM read, USR own | Pertanyaan asal. |
| period_id | bigint nullable | FK tb_period.id, set null | SA/ADM read, USR own | Periode pendaftaran. |
| submission_id | uuid nullable | grouping key | SA/ADM read, USR own | ID satu paket submission. |
| page | integer | default 1 | SA/ADM read, USR own | Halaman asal pertanyaan. |
| label | string nullable | nullable | SA/ADM read, USR own | Label halaman saat jawaban disimpan. |
| question | text nullable | snapshot | SA/ADM read, USR own | Snapshot teks pertanyaan. |
| sort_order | integer | default 0 | SA/ADM read, USR own | Snapshot urutan pertanyaan. |
| answer | string nullable | nullable | SA/ADM read/update, USR create/update own | Jawaban teks/pilihan. |
| file_path | longText nullable | nullable | SA/ADM read/update, USR create/update own | Path file upload pendaftaran. |
| type | enum | radio, checkbox, text, file, multiple_file | SA/ADM read, USR own | Tipe jawaban. |
| options | json nullable | nullable | SA/ADM read, USR own | Snapshot option pertanyaan. |
| created_at | timestamp | auto | SA/ADM/USR read sesuai akses | Waktu jawaban dibuat. |
| updated_at | timestamp | auto | SA/ADM/USR read sesuai akses | Waktu jawaban diperbarui. |

Operasi role:

| Operasi | SA | ADM | USR |
| --- | --- | --- | --- |
| listSubmissions() | Semua | Semua | Hanya submission sendiri |
| submitAnswers() | Ya | Ya | Ya |
| updateAnswers() | Ya | Ya | Hanya sendiri |
| deleteSubmission() | Ya | Ya sesuai UI/API | Hanya sendiri jika diizinkan |
| viewSubmissionResult() | Semua | Semua | Hanya hasil sendiri dan mengikuti status publish |

## Class: PPDBResult

Sumber data utama: tabel `tb_results`. Class ini menyimpan status verifikasi berkas dan hasil seleksi untuk satu `submission_id`.

| Attribute | Type | Key / Rule | Akses Role | Keterangan |
| --- | --- | --- | --- | --- |
| id | bigint | PK | SA/ADM read, USR read own | Identitas hasil seleksi. |
| submission_answers | char(36)/string | indexed, logical FK ke `tb_answers.submission_id` | SA/ADM manage, USR read own | ID submission yang diverifikasi. |
| selection_type | string nullable | nullable | SA/ADM update, USR read jika published | Jenis/tahap seleksi. |
| value | string nullable | nullable | SA/ADM update, USR read jika published | Nilai atau keterangan hasil seleksi. |
| status | boolean nullable | true lulus, false tidak lulus, null belum final | SA/ADM update, USR read jika published | Status kelulusan akhir. |
| is_approve | boolean nullable | true diterima, false dikembalikan, null belum diverifikasi | SA/ADM verify, USR read own | Status kelengkapan/verifikasi berkas. |
| created_at | timestamp | auto | SA/ADM/USR read sesuai akses | Waktu hasil dibuat. |
| updated_at | timestamp | auto | SA/ADM/USR read sesuai akses | Waktu hasil diperbarui. |

Operasi role:

| Operasi | SA | ADM | USR |
| --- | --- | --- | --- |
| showSubmissionResult() | Semua | Semua | Hanya submission sendiri dan mengikuti publish periode |
| verifySubmissionDocument() | Ya | Ya sesuai UI/API | Tidak |
| updateSelectionResult() | Ya | Ya sesuai UI/API | Tidak |
| downloadSubmissionFile() | Ya | Ya sesuai UI/API | Hanya file sendiri jika endpoint dipakai UI |

## Class: Notification

Sumber data utama: tabel `tb_notification`.

| Attribute | Type | Key / Rule | Akses Role | Keterangan |
| --- | --- | --- | --- | --- |
| id | bigint | PK | SA/ADM/USR read sesuai scope | Identitas notifikasi. |
| key | string | generated | SA/ADM/USR read sesuai scope | Key unik notifikasi. |
| label | string | email dikecualikan dari UI | SA manage, ADM/USR read allowed labels | Kategori notifikasi, contoh account, security, registration, whatsapp. |
| title | string | required | SA update allowed, ADM/USR read | Judul notifikasi. |
| message | longText | text/json | SA update/resend allowed, ADM/USR read | Isi notifikasi; WhatsApp dapat berisi JSON status pengiriman. |
| user_id | bigint nullable | FK users.id, null on delete | SA read global WA + own, ADM/USR own | Target user notifikasi. |
| is_read | boolean | default false | Owner mark-read | Status baca notifikasi. |
| created_at | timestamp | auto | SA/ADM/USR read sesuai scope | Waktu notifikasi dibuat. |
| updated_at | timestamp | auto | SA/ADM/USR read sesuai scope | Waktu notifikasi diperbarui. |

Scope label:

| Role | Label yang Ditampilkan |
| --- | --- |
| SA | `whatsapp` dari semua user, ditambah notifikasi pribadi: `account`, `[system]`, `security`, `registration`, `user management`. |
| ADM | Notifikasi sendiri: `account`, `security`, `registration`, `[system]`, `user management`. |
| USR | Notifikasi sendiri: `account`, `security`, `registration`, `[system]`. |

Operasi role:

| Operasi | SA | ADM | USR |
| --- | --- | --- | --- |
| listNotifications() | Ya sesuai scope SA | Ya sesuai scope ADM | Ya sesuai scope USR |
| viewNotification() | Semua jika SA, selain itu own only | Own only | Own only |
| markAsRead() | Own only | Own only | Own only |
| markAllAsRead() | Own only | Own only | Own only |
| updateNotification() | Own atau semua jika SA | Own only | Own only |
| deleteNotification() | Ya | Ya untuk notifikasi yang diizinkan | Own only |
| resendWhatsAppNotification() | Ya | Ya | Tidak |

## Class: NotificationWhatsAppMessage

Sumber data utama: tabel `notification_whatsapp_messages`.

| Attribute | Type | Key / Rule | Akses Role | Keterangan |
| --- | --- | --- | --- | --- |
| id | bigint | PK | SA/ADM read | Identitas template. |
| code | string | unique, required | SA/ADM manage | Kode template pesan. |
| label | string | required | SA/ADM manage | Nama template. |
| placeholder | string nullable | nullable | SA/ADM manage | Daftar placeholder yang didukung template. |
| message | longText | required | SA/ADM manage | Isi pesan WhatsApp. |
| created_at | timestamp | auto | SA/ADM read | Waktu template dibuat. |
| updated_at | timestamp | auto | SA/ADM read | Waktu template diperbarui. |

Operasi role:

| Operasi | SA | ADM | USR |
| --- | --- | --- | --- |
| listTemplates() | Ya | Ya | Tidak |
| createTemplate() | Ya | Ya | Tidak |
| updateTemplate() | Ya | Ya | Tidak |
| deleteTemplate() | Ya | Ya | Tidak |

## Class: WhatsAppSession

Sumber data utama: runtime socket server, bukan tabel database permanen.

| Attribute | Type | Key / Rule | Akses Role | Keterangan |
| --- | --- | --- | --- | --- |
| status / whatsapp_status | string | runtime | SA read, ADM read jika UI diberikan | Status koneksi seperti `qr-pending`, `authenticated`, `ready`, `disconnected`, `auth_failure`. |
| client_ready | boolean | runtime | SA read, ADM read jika UI diberikan | Penanda client WhatsApp siap kirim pesan. |
| session_exists | boolean | runtime | SA read | Penanda session tersimpan/aktif. |
| reconnect_attempts | integer | runtime | SA read | Jumlah percobaan reconnect. |
| timestamp | datetime/string | runtime | SA read | Waktu snapshot status. |

Operasi role:

| Operasi | SA | ADM | USR |
| --- | --- | --- | --- |
| getWhatsAppStatus() | Ya | Read-only jika UI diberikan | Tidak |
| getWhatsAppSession() | Ya | Read-only jika UI diberikan | Tidak |
| sendWhatsAppNotification() | Ya | Tidak | Tidak |
| disconnectWhatsApp() | Ya | Tidak | Tidak |

## Class: WhatsAppQRCode

Sumber data utama: event socket `whatsapp-qr` dan endpoint session.

| Attribute | Type | Key / Rule | Akses Role | Keterangan |
| --- | --- | --- | --- | --- |
| qr | string nullable | data URL | SA read/scan | QR code untuk pairing WhatsApp. |
| qr_available | boolean | runtime | SA read | Penanda QR tersedia. |
| qr_retry | integer | runtime | SA read | Counter retry QR. |
| qr_updated_at | datetime/string nullable | runtime | SA read | Waktu QR terakhir diperbarui. |
| qr_code_scanned | boolean | runtime | SA read | Status QR sudah discan. |

Operasi role:

| Operasi | SA | ADM | USR |
| --- | --- | --- | --- |
| receiveQRCodeEvent() | Ya | Tidak | Tidak |
| scanQRCode() | Ya | Tidak | Tidak |
| refreshQRCodeBySession() | Ya | Tidak | Tidak |

## Backend Controller Yang Bersangkutan

Controller di bawah mengikuti kode Laravel di `backend/app/Http/Controllers` dan route utama di `backend/routes/api.php`.

| Area | Controller | Method Utama | Model / Helper Terkait | Akses Role | Tanggung Jawab |
| --- | --- | --- | --- | --- | --- |
| Account / Auth | `API\AuthController` | `register`, `login`, `verifyOtp`, `requestLoginOtp`, `requestPasswordResetOtp`, `resendOtp`, `changePassword`, `changePhoneNumber`, `logout`, `isAuthenticated`, `updateLastOnline` | `User`, `Otp`, `UserLoginDetail`, `Notification`; `NotificationHelper`; socket WA untuk OTP | Public untuk register/login/OTP request, auth untuk logout/update online, own account untuk password/phone | Registrasi, login, OTP WhatsApp, update password/nomor, token session, dan status online user. |
| Account / Users | `UserController` | `search`, `getByUsername`, `all`, `index`, `view`, `store`, `updatePassword`, `update`, `actived`, `suspend`, `delete` | `User`, `Role`, `RoleUser`, `Notification`; `NotificationHelper`; S3 avatar | SA full termasuk role/status/suspend/delete, ADM create/read, USR own profile/password | Management user, account profile, update role, activate/suspend, delete user, avatar, dan notifikasi perubahan akun. |
| Account / Roles | `RoleController` | `all`, `index`, `view`, `store`, `update`, `delete` | `Role`, `RoleUser` | SA/ADM manage lewat route, delete ditolak jika role masih dipakai user | CRUD role dan proteksi penghapusan role yang masih terpasang ke user. |
| PPDB - Periode | `Form\PeriodController` | `all`, `index`, `view`, `store`, `update`, `delete`, `broadcastToUserSubmitted` | `Form\Period`, `Form\Answers`, `NotificationWhatsAppMessage`; `NotificationHelper`; socket HTTP | SA create/update/delete/broadcast, ADM/USR read periode yang tersedia | Kelola periode PPDB, status/publish/archive, jumlah question/answer, dan broadcast ke user yang sudah submit. |
| PPDB - Pertanyaan | `Form\QuestionController` | `show`, `view`, `find`, `store`, `update`, `updateOrder`, `destroy` | `Form\Questions`, `Form\Period` | SA/ADM CRUD dan reorder, USR read untuk isi form | Kelola daftar pertanyaan, tipe input, opsi, file type, required flag, page, dan sort order. |
| PPDB - Jawaban/Berkas | `Form\AnswerController` | `getSubmissionsPerDayByPeriod`, `getSubmissionsPerDayDetail`, `getSubmissionsByPeriod`, `getStatusTotals`, `getGroupedAnswersPublic`, `getGroupedAnswers`, `getBySubmission`, `submitAnswers`, `updateAnswers`, `deleteRespondentBySubmission` | `Form\Answers`, `Form\Questions`, `Form\Period`, `Form\Result`, `NotificationWhatsAppMessage`; `NotificationHelper`, `StatusHelper`; S3; socket HTTP | Auth required untuk mayoritas endpoint, public hanya grouped public; SA/ADM operasional semua submission, USR submission sendiri | Simpan/update jawaban, upload berkas, grouping submission, dashboard statistik, status kelulusan, notifikasi submission baru, dan delete respondent. |
| PPDB - Result/Verifikasi | `Form\ResultController` | `listPeriods`, `show`, `verify`, `update`, `download`, `downloadMultiple` | `Form\Answers`, `Form\Result`, `Form\Period`; `StatusHelper`, `NotificationHelper`, `SocketHelper`; S3 | Auth untuk result show/verify/update/list; file download endpoint berada di route public; secara UI SA/ADM memverifikasi, USR membaca hasil sendiri | Verifikasi berkas, update hasil seleksi, format status berdasarkan role/publish, kirim socket update, kirim WA result, dan download berkas. |
| PPDB - Excel | `Form\ExcelController` | `export`, `import`, `downloadTemplate` | `RespondentsExport`, `RespondentsImport`; `Form\Period`, `Form\Answers`, `Form\Questions`, `Form\Result` lewat import/export | Middleware route saat ini dikomentari; seharusnya dipakai untuk operator PPDB | Export, import, dan template Excel data respondent PPDB. |
| Template WhatsApp | `WhatsAppNotificationController` | `index`, `store`, `show`, `update`, `destroy` | `NotificationWhatsAppMessage` | `index` terbuka, create/update/delete SA/ADM | CRUD template pesan WhatsApp yang dipakai notifikasi PPDB/account. |
| QR / Session WhatsApp | `WhatsappController` | `sendWhatsappNotification`, `disconnectWhatsapp`, `getWhatsappStatus`, `getWhatsappSession` | Socket server WhatsApp; `API_KEY`, `SOCKET_SERVER_URL` | Send/disconnect route SA; status/session bergantung route/UI yang memanggil | Proxy Laravel ke socket server untuk kirim WA manual, disconnect session, health status, session info, dan QR runtime. |
| Notifikasi | `NotificationController` | `index`, `view`, `markAsRead`, `markAllAsRead`, `update`, `delete`, `resendNotification`, `bulkDelete` | `Notification`, `User`; socket WA saat resend | SA lihat semua WA + own, ADM/USR own sesuai label, resend WA SA/ADM | List/preview notifikasi, filter email, mark read, update/delete, unread count, dan resend WhatsApp. |

## Kamus Method Controller Backend

Kamus method di bawah hanya mencatat controller yang berhubungan dengan scope PPDB, account/users, template dan QR WhatsApp, serta notifikasi. Controller Blog, Gallery, Contact CMS, schedule management, dan role management tidak dijabarkan sebagai flow aktif karena menu management-nya tidak ditampilkan atau sedang dikomentari di UI.

### `App\Http\Controllers\API\AuthController`

| Method | Route | Akses | Input Utama | Output / Dampak |
| --- | --- | --- | --- | --- |
| `register(Request $request)` | `POST /api/register` | Public | `name`, `email`, `phone_number`, `password`, `passwordConfirm`, `image?` | Membuat user baru role `user`, normalisasi nomor WA, upload avatar ke S3 jika ada, membuat OTP `registration`, mengirim OTP WhatsApp, membuat notifikasi `registration`. |
| `login(Request $request)` | `POST /api/login` | Public | `login` email/nomor, `password` | Validasi kredensial dan status akun. Jika `otp_verified=true`, mengembalikan token Sanctum dan data role. Jika belum, membuat/mengirim OTP login. Akun suspend ditolak 403. |
| `verifyOtp(Request $request)` | `POST /api/verify-otp` | Public | `user_id`, `otp_code`, `type=registration/login/password_reset` | Menandai OTP sebagai digunakan. Untuk registration/login mengaktifkan `otp_verified` dan mengembalikan token; untuk password reset hanya membuka izin reset password. Membuat notifikasi security/registration. |
| `resendOtp(Request $request)` | `POST /api/resend-otp` | Public | `user_id`, `type` | Mengirim ulang OTP aktif jika masih valid, atau membuat OTP baru. Response berisi `otp_expires_at`, `user_id`, `phone_number`, `cooldown`. |
| `requestLoginOtp(Request $request)` | `POST /api/request-login-otp`, `POST /api/login/otp` | Public | `login`, `type=login/password_reset` | Mencari user dari email, membuat/mengirim OTP WhatsApp untuk login atau password reset. |
| `requestPasswordResetOtp(Request $request)` | `POST /api/request-password-reset-otp` | Public | `login`, `type=email/phone` | Mencari user dari email atau nomor WA, membuat/mengirim OTP `password_reset`, membuat notifikasi security. |
| `changePassword(Request $request)` | `POST /api/change-password` | Public dengan syarat OTP reset sudah valid | `user_id`, `password`, `password_confirmation` | Mengubah password setelah OTP reset diverifikasi, reset `otp_verified=false`, revoke semua token, membuat notifikasi security. |
| `changePhoneNumber(Request $request)` | `POST /api/change-phone-number` | Public untuk akun belum OTP verified | `user_id`, `new_phone_number` | Mengubah nomor WA akun yang belum verified, membatalkan OTP registration lama, membuat OTP baru, mengirim OTP ke nomor baru. |
| `logout(Request $request)` | `POST /api/logout` | Auth + `check.status` | Token aktif | Update `last_online_at`, hapus current access token, membuat notifikasi logout. |
| `isAuthenticated(Request $request)` | `GET /api/is-authenticated` | Auth + `check.status` | Token aktif | Validasi token, load role, deteksi device, update `last_online_at` jika perlu, mengembalikan profil login. |
| `updateLastOnline(Request $request)` | `POST /api/user/update-last-online` | Auth | `online` boolean | Update waktu online terakhir user ke timezone Asia/Jakarta. |
| `getUserById($id)` | Tidak terdaftar di `api.php` untuk AuthController | Internal / non-route aktif | `id` | Mengambil data ringkas user dan `last_online_at`. |
| `onload(Request $request)` | Tidak terdaftar di `api.php` | Internal / non-route aktif | Token aktif | Mengembalikan token current dan role pertama user. |
| `updatePassword(Request $request)` | Tidak terdaftar di `api.php`; endpoint aktif memakai `UserController::updatePassword` | Internal / non-route aktif | `new_password`, `password_confirmation` | Mengubah password user auth dan membuat notifikasi security. |

Method private penting:

| Method | Fungsi |
| --- | --- |
| `standardizePhoneNumber(string $phone)` | Normalisasi nomor Indonesia menjadi format `62xxxxxxxx`. |
| `uploadAvatar($file)` | Upload avatar ke S3 path `uploads/user/images`. |
| `saveLoginDetail(Request $request, int $userId)` | Simpan device, browser, IP, email/nama/nomor ke `user_login_details`. |
| `generateOtp($user, $type)` | Invalidate OTP lama untuk tipe sama, membuat OTP 6 digit baru dengan expired 10 menit. |
| `sendOtp($user, $otpCode, $type)` | Kirim OTP lewat WhatsApp socket server. |
| `sendWhatsappMessage($phoneNumber, $message)` | Proxy multipart ke socket server `/send-whatsapp` dengan `X-API-KEY`. |

### `App\Http\Controllers\UserController`

| Method | Route | Akses | Input Utama | Output / Dampak |
| --- | --- | --- | --- | --- |
| `search(Request $request)` | `GET /api/users/search` | Auth | `q?`, `limit?` | Search user untuk kebutuhan mention/search ringan; output `id`, `name`, `avatar`, `phone_number`, `email`. |
| `getByUsername($username)` | `GET /api/users/username/{username}` | Auth | `username` | Cari user berdasarkan nama/username dan format avatar URL. |
| `all(Request $request)` | `GET /api/users/all` | Auth + `check.status` | `sortKey`, `sortDirection`, `search` | Mengembalikan semua user tanpa pagination. Role `user` dibatasi hanya datanya sendiri. |
| `index(Request $request)` | `GET /api/users` | Auth + `check.status` | `perPage`, `sortKey`, `sortDirection`, `search` | Mengembalikan user paginated beserta total active/suspend. |
| `view($email)` | `GET /api/users/{email}` | Auth + `check.status` | `email` | Mengambil profil user. Non super admin hanya boleh melihat data sendiri. |
| `store(Request $request)` | `POST /api/users` | Route SA/ADM, method mengizinkan SA/ADM | `name`, `email`, `password`, `passwordConfirm`, `role_id`, `image?` | Membuat user baru, upload avatar ke S3, assign role, membuat notifikasi account dan user management. |
| `updatePassword(Request $request, $id)` | `POST /api/users/update-password/{id}` | Auth + `check.status`; SA boleh semua, user hanya sendiri | `new_password`, `password_confirmation` | Mengubah password user target, membuat notifikasi security dan user management jika diubah oleh orang lain. |
| `update(Request $request, $id)` | `POST /api/users/{id}` | Auth + `check.status`; SA boleh semua, user hanya sendiri | `name?`, `email?`, `phone_number?`, `status?`, `role_id?`, `image?` | Update profil/avatar. `status` dan `role_id` hanya diproses jika actor `super admin`. Role `user` tidak boleh update nomor dari account profile. Membuat notifikasi perubahan profil/status/role. |
| `delete($id)` | `DELETE /api/users/{id}` | Route SA/ADM, method hanya mengizinkan SA | `id` | Hapus user, hapus avatar lama dari S3 jika ada, membuat notifikasi user management untuk actor. |
| `actived($id)` | Tidak terdaftar di route aktif | Internal / non-route aktif | `id` | Mengaktifkan user dan membuat notifikasi. Logic aktif/suspend di UI saat ini memakai `update(status)`. |
| `suspend($id)` | Tidak terdaftar di route aktif | Internal / non-route aktif | `id` | Suspend user dan membuat notifikasi. Logic aktif/suspend di UI saat ini memakai `update(status)`. |

Catatan method user:

| Catatan | Dampak |
| --- | --- |
| `update(status, role_id)` | Ini method yang dipakai untuk update role dan suspend/active dari account profile. Backend hanya memproses field tersebut jika actor adalah `super admin`. |

### `App\Http\Controllers\Form\PeriodController`

| Method | Route | Akses | Input Utama | Output / Dampak |
| --- | --- | --- | --- | --- |
| `all()` | `GET /api/form/period/all` | Public | Tidak ada | Mengambil semua period beserta questions, answers, total question, dan total respondent. |
| `index(Request $request)` | `GET /api/form/period` | Public | `perPage`, `sortKey`, `sortDirection`, `q`, `updated?` | List period paginated, search title/key/description, hitung total visible/hidden/published/archived dengan cache. |
| `view($key)` | `GET /api/form/period/{key}` | Public | `key` period | Detail period berdasarkan key beserta questions dan total respondent. |
| `store(Request $request)` | `POST /api/form/period` | Auth + `role:super admin` | `title`, `description`, `status`, `is_published`, `is_archived` | Membuat period baru dengan random key, invalidasi cache, broadcast socket `notify-new-period`. |
| `update(Request $request, $key)` | `POST /api/form/period/{key}` | Auth + `role:super admin` | `title`, `description`, `status`, `is_published`, `is_archived` | Update detail period, status buka/tutup, publish, archive, invalidasi cache, broadcast socket `notify-period-updated`. |
| `delete($id)` | `DELETE /api/form/period/{id}` | Auth + `role:super admin` | `id` | Hapus period, invalidasi cache, broadcast socket `notify-period-deleted`. |
| `broadcastToUserSubmitted(Request $request, $key)` | `POST /api/form/period/broadcast/{key}` | Auth + `role:super admin` | `key` period | Ambil submission pada period, hitung status hasil, kirim broadcast WhatsApp berdasarkan template ke user yang sudah submit. |

### `App\Http\Controllers\Form\QuestionController`

| Method | Route | Akses | Input Utama | Output / Dampak |
| --- | --- | --- | --- | --- |
| `show()` | `GET /api/form/question/show` | Auth | User auth | Mengambil questions milik author login, urut `page` dan `sort_order`. |
| `view(Request $request)` | `GET /api/form/question` | Auth | User auth | Mengambil questions milik author login tanpa wrapper response. |
| `find($periodKey)` | `GET /api/form/question/find/{periodKey}` | Auth | `periodKey` | Mengambil period dan daftar questions-nya, urut page/order, untuk preview/form PPDB. |
| `store(Request $request)` | `POST /api/form/question` | Auth + `role:administrator|super admin` | `questions[]`, `question`, `period_key`, `type`, `options?`, `fileTypes?`, `page`, `localIndex?`, `sort_order?`, `is_required?` | Membuat satu/banyak question, insert sesuai sort order, default file type untuk `multiple_file`, normalize order. |
| `update(Request $request)` | `POST /api/form/question/update/{periodId}` | Auth + `role:administrator|super admin` | `questions[]` dengan `id`, `question`, `type`, `options?`, `fileTypes?`, `page`, `sort_order?`, `is_required?` | Update field question yang sudah ada. Catatan: route punya `{periodId}` tetapi method tidak menerima parameter itu. |
| `updateOrder(Request $request)` | `POST /api/form/question/order` | Auth + `role:administrator|super admin` | `period_key`, `page`, `questions[].id`, `questions[].sort_order` | Validasi semua question milik period/page, update urutan drag and drop, return daftar questions terbaru. |
| `destroy($id)` | `DELETE /api/form/question/{id}` | Auth + `role:administrator|super admin` | `id` question | Hapus question lalu normalize `sort_order` untuk page/period terkait. |

Method private penting:

| Method | Fungsi |
| --- | --- |
| `normalizeSortOrder($periodId, $page)` | Merapikan ulang `sort_order` mulai dari 0 setelah insert/delete. |
| `safeJsonDecode($json, $default)` | Decode JSON options/fileTypes dengan fallback aman. |

### `App\Http\Controllers\Form\AnswerController`

| Method | Route | Akses | Input Utama | Output / Dampak |
| --- | --- | --- | --- | --- |
| `getGroupedAnswersPublic(Request $request)` | `GET /api/form/answer/group/respondent/public` | Public | `perPage`, `search`, `fromDate`, `toDate`, `period_id` | List hasil publik yang sudah punya result dan period tidak archived; dipakai halaman hasil seleksi publik. |
| `getSubmissionsByPeriod(Request $request)` | `GET /api/form/answer/group/respondent/period` | Auth | `fromDate?`, `toDate?` | Statistik jumlah submission per period, termasuk status verified/pass/fail/undecided/unverified. |
| `getSubmissionsPerDayByPeriod(Request $request)` | `GET /api/form/answer/group/respondent/period/perday` | Auth | `fromDate?`, `toDate?` | Statistik submission harian per period untuk chart dashboard. |
| `getSubmissionsPerDayDetail($periodId)` | `GET /api/form/answer/group/respondent/perday/detail/{periodId}` | Auth | `periodId` | Detail statistik per hari untuk satu period, termasuk pass/fail/undecided/unverified. |
| `getStatusTotals(Request $request)` | `GET /api/form/answer/status-totals` | Auth | `search?`, `period_id?`, `fromDate?`, `toDate?`, `user_id?`, `userAll?` | Menghitung total status submission sesuai role dan publish state: belum diverifikasi, berkas diterima/dikembalikan, lulus/tidak lulus, dll. |
| `getGroupedAnswers(Request $request)` | `GET /api/form/answer/group/respondent` | Auth | `perPage`, `search`, `period_id`, `user_id`, `fromDate`, `toDate`, `sort_by`, `sort_direction` | List berkas pendaftaran grouped per `submission_id`; SA/ADM bisa lihat semua, user hanya milik sendiri. Mengembalikan answers, period, user, result, dan validation status. |
| `getBySubmission($submissionId)` | `GET /api/form/answer/group/respondent/{submissionId}` | Auth | `submissionId` | Detail satu submission beserta answers, user, period, result, dan visibility hasil sesuai role/publish. |
| `submitAnswers(Request $request)` | `POST /api/form/answer` | Auth | `answers` JSON array, `files[question_id]?` | Submit formulir PPDB, validasi period aktif, required answer/file, upload file ke S3, buat row `tb_answers`, response cepat, lalu background kirim notifikasi/socket/WhatsApp. |
| `updateAnswers(Request $request, $submissionId)` | `POST /api/form/answer/update-submission/{submissionId}` | Auth; owner atau SA/ADM | `answers`, `files[question_id]?` | Update jawaban dan file pada submission, validasi question masih milik period yang sama, upload file baru, update/create answer. |
| `deleteRespondentBySubmission($submissionId)` | `POST /api/form/answer/respondent/delete/{submissionId}` | Auth | `submissionId` | Hapus semua answer dan result untuk submission, hapus file dari S3, broadcast socket `notify-respondent-deleted`. |
| `deleteRespondent($submissionId)` | Tidak terdaftar di route aktif | Internal / non-route aktif | `submissionId` | Duplikasi logic delete respondent; endpoint aktif memakai `deleteRespondentBySubmission`. |

Method private penting:

| Method | Fungsi |
| --- | --- |
| `handleSecureFileUpload($file, $questionId)` | Upload file jawaban ke storage dengan nama/path aman. |
| `getMimeType($file)` | Deteksi MIME type file upload. |
| `validateFileSecurely($file)` | Validasi ukuran, extension, MIME, dan keamanan file. |
| `generateSecureFilename($extension)` | Membuat nama file aman/unik. |
| `containsSuspiciousContent($file)` | Deteksi konten file mencurigakan. |
| `getFileUrl($filePath)` | Format path file jawaban menjadi response URL/metadata. |
| `buildAnswerFileResponse($path)` | Membentuk metadata file untuk response answer. |
| `deleteFileSecurely($filePath)` | Menghapus file dari storage secara aman. |

### `App\Http\Controllers\Form\ResultController`

| Method | Route | Akses | Input Utama | Output / Dampak |
| --- | --- | --- | --- | --- |
| `listPeriods()` | `GET /api/respondent/result/get-periods` | Auth | Tidak ada | List period ringkas untuk filter result/verifikasi. |
| `show($submissionId)` | `GET /api/respondent/result/{submissionId}` | Auth | `submissionId` | Detail result dan answers untuk satu submission. Result final disembunyikan dari user jika period belum published. |
| `verify(Request $request, $submissionId)` | `POST /api/respondent/result/verify/{submissionId}` | Auth; secara UI SA/ADM | `is_approve` boolean | Membuat/update `tb_results` untuk status verifikasi berkas, reset hasil seleksi, response cepat, background socket `result-verified` dan WhatsApp document received/rejected. |
| `update(Request $request, $submissionId)` | `PUT /api/respondent/result/upadte/{submissionId}` | Auth; secara UI SA/ADM | `selection_type`, `value?`, `status?`, `is_approve?` | Update hasil seleksi tahap akhir, hitung validation status, response cepat, background socket `result-updated` dan WhatsApp hasil seleksi. Catatan route typo: `upadte`. |
| `download(Request $request)` | `POST /api/respondent/result/files/download` | Route public saat ini | `file_path` | Download satu file dari S3 setelah normalisasi path. |
| `downloadMultiple(Request $request)` | `POST /api/respondent/result/files/download-multiple` | Route public saat ini | `file_paths[]`, `zip_name?` | Download banyak file dari S3 sebagai ZIP sementara. |

Method private penting:

| Method | Fungsi |
| --- | --- |
| `getEnumValues($table, $column)` | Membaca opsi enum MySQL jika kolom ada. |
| `handleBackgroundProcessing(...)` | Background proses setelah verifikasi: socket + WhatsApp document received/rejected. |
| `handleBackgroundUpdateProcessing(...)` | Background proses setelah update hasil: socket + WhatsApp hasil seleksi. |
| `formatFilePath($filePath)` | Format file path answer agar bisa dipakai frontend. |
| `buildAnswerFileResponse($path)` | Membentuk metadata file result/answer. |

### `App\Http\Controllers\Form\ExcelController`

| Method | Route | Akses | Input Utama | Output / Dampak |
| --- | --- | --- | --- | --- |
| `export(Request $request)` | `POST /api/respondent/excel/export` | Route middleware auth/role saat ini dikomentari; UI export masih tampil | `period_id?` | Export respondent ke XLSX, semua period atau per period. |
| `downloadTemplate(Request $request)` | `GET /api/respondent/excel/template` | Route middleware auth/role saat ini dikomentari; tombol UI hidden | `period_id?` | Download template XLSX respondent. Tidak masuk flow aktif UI. |
| `import(Request $request)` | `POST /api/respondent/excel/import` | Route middleware auth/role saat ini dikomentari; tombol UI hidden | `file`, `period_id?` | Import respondent dari XLS/XLSX, bisa auto-create period. Tidak masuk flow aktif UI. |

### `App\Http\Controllers\NotificationController`

| Method | Route | Akses | Input Utama | Output / Dampak |
| --- | --- | --- | --- | --- |
| `index(Request $request)` | `GET /api/notifications` | Auth + `check.status` | `perPage`, `sortKey`, `sortDirection`, `filter`, `fetchAll?` | List notifikasi sesuai scope role. Label `email` selalu dikeluarkan. SA melihat semua WhatsApp + notifikasi pribadi; ADM/USR hanya notifikasi pribadi sesuai label allowed. |
| `view($id)` | `GET /api/notifications/{id}` | Auth + `check.status` | `id` | Detail notifikasi. Non-owner ditolak kecuali `super admin`. Label `email` dikembalikan 404. |
| `markAsRead($id)` | `POST /api/notifications/mark-read/{id}` | Auth + `check.status` | `id` | Menandai satu notifikasi milik user sebagai dibaca. Label `email` dikecualikan. |
| `markAllAsRead()` | `POST /api/notifications/mark-all` | Auth + `check.status` | User auth | Menandai semua notifikasi non-email milik user sebagai dibaca. |
| `update(Request $request, $id)` | `POST /api/notifications/{id}` | Auth + `check.status`; owner atau SA | `label`, `title`, `message` | Update isi notifikasi. |
| `delete($id)` | `DELETE /api/notifications/{id}` | Auth + `check.status`; ADM/SA bisa by id, user hanya own | `id` | Hapus notifikasi. |
| `resendNotification($id)` | `POST /api/notifications/resend/{id}` | Auth + `check.status`; hanya SA/ADM | `id` notifikasi label `whatsapp` | Kirim ulang WhatsApp dari notifikasi gagal/terpilih, update message JSON dengan status kirim terbaru. |
| `bulkDelete(Request $request)` | Tidak terdaftar di route aktif | Internal / non-route aktif | `ids[]` | Bulk delete notifikasi. SA bisa semua, role lain hanya own. |

Method private penting:

| Method | Fungsi |
| --- | --- |
| `getNotifications(Request $request)` | Query utama list notifikasi, filter role, filter label, unread count, pagination/fetchAll. |
| `formatResponse($notifications, $unreadCount)` | Format response pagination/non-pagination. |
| `transformNotifData($notif, $currentUserId)` | Format object notifikasi dan force read di response jika SA melihat notifikasi user lain. |
| `standardizePhoneNumber(string $phone)` | Normalisasi nomor sebelum resend WhatsApp. |
| `htmlToWhatsAppText($html)` | Konversi HTML template menjadi format teks WhatsApp. |

### `App\Http\Controllers\WhatsAppNotificationController`

| Method | Route | Akses | Input Utama | Output / Dampak |
| --- | --- | --- | --- | --- |
| `index()` | `GET /api/whatsapp-notifications` | Public route; UI super admin | Tidak ada | List semua template pesan WhatsApp, urut terbaru. |
| `store(Request $request)` | `POST /api/whatsapp-notifications` | Auth + `role:administrator|super admin` | `code`, `label`, `message` | Membuat template WhatsApp baru. |
| `show($id)` | `GET /api/whatsapp-notifications/{id}` | Auth + `role:administrator|super admin` | `id` | Detail satu template WhatsApp. |
| `update(Request $request, $id)` | `PUT /api/whatsapp-notifications/{id}` | Auth + `role:administrator|super admin` | `code`, `label`, `message` | Update template WhatsApp. UI saat ini fokus edit label/message; code tampak disabled di modal. |
| `destroy($id)` | `DELETE /api/whatsapp-notifications/{id}` | Auth + `role:administrator|super admin` | `id` | Hapus template WhatsApp. |

### `App\Http\Controllers\WhatsappController`

| Method | Route | Akses | Input Utama | Output / Dampak |
| --- | --- | --- | --- | --- |
| `sendWhatsappNotification(Request $request)` | `POST /api/send-whatsapp` | Auth + `role:super admin` | `phone_number`, `message`, `images[]?`, `files[]?` | Validasi pesan dan attachment, proxy multipart ke socket server `/send-whatsapp`, return status berhasil/gagal dari Node.js. |
| `disconnectWhatsapp(Request $request)` | `POST /api/logout-whatsapp` | Auth + `role:super admin` | Tidak ada | Proxy logout/disconnect ke socket server `/api/logout-whatsapp`, menghapus session WhatsApp dan memicu QR baru. |
| `getWhatsappStatus()` | Tidak terdaftar di `api.php` | Internal / non-route aktif | Tidak ada | Proxy health check ke socket server `/health`. Frontend saat ini lebih banyak memakai socket/session langsung dari Node.js. |
| `getWhatsappSession()` | Tidak terdaftar di `api.php` | Internal / non-route aktif | Tidak ada | Proxy session info ke socket server `/api/whatsapp-session`, termasuk QR runtime, status, retry, dan timestamp. |

## Backend Model Yang Bersangkutan

| Model | Tabel / Sumber | Field Utama | Relasi / Referensi | Dipakai Oleh |
| --- | --- | --- | --- | --- |
| `App\Models\User` | `users` | `name`, `phone_number`, `otp_verified`, `email`, `status`, `password`, `avatar`, `last_online_at` | many-to-many `roles`, hasMany `otps`, hasMany `notiffications`, hasMany `answers` secara logical | `AuthController`, `UserController`, `AnswerController`, `ResultController`, `NotificationController` |
| `App\Models\Role` | `roles` | `name`, `display_name`, `description` | belongsToMany `routes`, pivot `role_user` untuk user | `RoleController`, `UserController`, middleware role |
| `App\Models\RoleUser` | `role_user` | `role_id`, `user_id`, `user_type` | belongsTo `User`; pivot Laratrust | `RoleController`, `UserController` |
| `App\Models\Otp` | `otps` | `user_id`, `otp_code`, `type`, `expires_at`, `is_used` | belongs to user secara logical | `API\AuthController` |
| `App\Models\UserLoginDetail` | `user_login_details` | `user_id`, `platform`, `browser`, `ip_address`, `email`, `name` | tidak ada relasi eksplisit | `API\AuthController` |
| `App\Models\Form\Period` | `tb_period` | `key`, `title`, `description`, `status`, `is_published`, `is_archived` | hasMany `questions`, hasMany `answers`; scope `active`, `published`, `archived` | `PeriodController`, `QuestionController`, `AnswerController`, `ResultController`, `ExcelController` |
| `App\Models\Form\Questions` | `tb_questions` | `question`, `period_id`, `type`, `label`, `options`, `page`, `author_id`, `file_types`, `sort_order`, `is_required` | belongsTo `period`, hasMany `answers` | `QuestionController`, `AnswerController`, import/export PPDB |
| `App\Models\Form\Answers` | `tb_answers` | `question_id`, `period_id`, `submission_id`, `user_id`, `answer`, `file_path`, `type`, `options`, `page`, `sort_order` | belongsTo `user`, `period`, `question`; logical one-to-one ke `Result` via submission | `AnswerController`, `ResultController`, `PeriodController`, import/export PPDB |
| `App\Models\Form\Result` | `tb_results` | `submission_answers`, `selection_type`, `value`, `status`, `is_approve` | logical reference ke `tb_answers.submission_id` | `ResultController`, `AnswerController`, import/export PPDB |
| `App\Models\Notification` | `tb_notification` | `key`, `label`, `title`, `message`, `user_id`, `is_read` | belongsTo `User`; helper `createNotification`, `markAsRead` | `NotificationController`, `NotificationHelper`, `AuthController`, `UserController`, PPDB controller |
| `App\Models\NotificationWhatsAppMessage` | `notification_whatsapp_messages` | `code`, `label`, `message`, `placeholder` | logical reference dari template code | `WhatsAppNotificationController`, `NotificationHelper`, `AnswerController`, `ResultController`, `PeriodController` |
| `WhatsAppSession` | runtime socket server | `status`, `client_ready`, `session_exists`, `reconnect_attempts`, `timestamp` | tidak ada tabel permanen | `WhatsappController`, frontend QR/session page |
| `WhatsAppQRCode` | runtime socket event / session endpoint | `qr`, `qr_available`, `qr_retry`, `qr_updated_at`, `qr_code_scanned` | bagian dari session runtime | `WhatsappController`, socket server, frontend QR/session page |

## Backend Helper Yang Bersangkutan

| Helper | Method / Fungsi | Dipakai Oleh | Output / Dampak |
| --- | --- | --- | --- |
| `NotificationHelper` | `create`, `simple` | `AuthController`, `UserController`, PPDB controller | Membuat row `tb_notification` dengan label account/security/registration/user management/whatsapp/[system]. |
| `NotificationHelper` | `sendWhatsApp` | `AnswerController`, `ResultController`, `PeriodController` | Ambil template `NotificationWhatsAppMessage`, replace placeholder, kirim ke socket `/send-whatsapp`, lalu simpan log notifikasi `whatsapp` dan `[system]`. |
| `NotificationHelper` | `toE164PhoneNumber`, `htmlToWhatsAppText` | `sendWhatsApp` dan formatter resend | Normalisasi nomor Indonesia dan konversi HTML template menjadi format teks WhatsApp. |
| `NotificationHelper` | `sendEmail`, `convertToHtmlEmail` | Legacy helper | Masih ada di kode, tetapi fitur email notification tidak masuk fitur aktif karena label `email` difilter dari UI/notifikasi. |
| `StatusHelper` | `getSubmissionStatus` | `AnswerController`, `ResultController` | Menghasilkan label/icon/color status berdasarkan `Result`, role user, dan `Period.is_published`. |
| `StatusHelper` | `incrementStatusCount` | `AnswerController` | Mengisi agregasi status dashboard/statistik PPDB. |
| `SocketHelper` | `notify`, `notifyNewSubmission`, `notifyResultVerified`, `notifyResultUpdated`, `notifyRespondentDeleted`, `notifyNewPeriod`, `notifyPeriodUpdated`, `notifyPeriodDeleted` | `ResultController` dan flow PPDB lain yang memakai socket | Mengirim event ke socket server melalui `SOCKET_SERVER_URL` dengan `API_KEY`. |

## Catatan Route dan Boundary Backend

- `/form/period` read bersifat public, tetapi create/update/delete/broadcast dibatasi `auth:sanctum` + `role:super admin`.
- `/form/question` read membutuhkan auth, sedangkan create/update/order/delete dibatasi `role:administrator|super admin`.
- `/form/answer/group/respondent/public` public; endpoint answer lain membutuhkan auth.
- `/respondent/result/files/download` dan `/respondent/result/files/download-multiple` berada di luar middleware auth pada route saat ini.
- `/respondent/excel` route export/import/template saat ini middleware `auth:sanctum` + `role:administrator|super admin` masih dikomentari di route.
- `/notifications` selalu mengecualikan label `email`; resend hanya untuk notifikasi `whatsapp`.
- `/whatsapp-notifications` `index` terbuka; create/update/delete dibatasi `auth:sanctum` + `role:administrator|super admin`.
- `/send-whatsapp` dan `/logout-whatsapp` dibatasi `auth:sanctum` + `role:super admin`.

## Catatan Boundary

- Fitur email notification tidak dimasukkan sebagai fitur aktif karena label `email` difilter dari daftar notifikasi.
- QR WhatsApp dan session WhatsApp adalah class runtime untuk kebutuhan class diagram, bukan tabel database.
- Result/verifikasi dijabarkan hanya sampai kebutuhan PPDB. Detail import/export Excel, socket runtime internal, dan auth flow di luar account/users tidak diperluas menjadi class diagram terpisah.
