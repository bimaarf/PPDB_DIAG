# Full System Flow - PSB Digital Architecture

Dokumen ini menjelaskan alur utama sistem PSB: user login/terautentikasi, siswa mengisi formulir pendaftaran, admin/panitia memverifikasi berkas, lalu admin/panitia mengisi hasil seleksi. Flow ini hanya memuat fitur yang dipakai pada proses utama.

```mermaid
sequenceDiagram
    autonumber
    actor Siswa
    actor Admin as Admin/Panitia
    participant FE as React SPA
    participant Auth as AuthController
    participant Period as Period/Question Controller
    participant Answer as AnswerController
    participant Result as ResultController
    participant Status as StatusHelper
    participant Notif as NotificationHelper
    participant DB as MySQL Database
    participant Socket as Socket.IO Server
    participant WA as WhatsApp API

    Note over Siswa, Auth: Autentikasi dasar
    Siswa->>FE: Login
    FE->>Auth: POST /api/login
    Auth->>DB: Cek user, password, status, role
    alt sudah otp_verified
        Auth-->>FE: 200 token Sanctum + data user
    else belum otp_verified
        Auth->>DB: Ambil/generate OTP login
        Auth->>WA: Kirim OTP WhatsApp
        Auth-->>FE: 200 user_id + otp_expires_at
        Siswa->>FE: Input OTP
        FE->>Auth: POST /api/verify-otp
        Auth->>DB: Tandai OTP used, set otp_verified, buat token
        Auth-->>FE: 200 token Sanctum + data user
    end

    Note over Siswa, Answer: Pengisian formulir pendaftaran
    Siswa->>FE: Buka periode/form PPDB
    FE->>Period: GET /api/form/period
    Period->>DB: Ambil periode aktif
    Period-->>FE: Data periode
    FE->>Period: GET /api/form/question/find/{periodKey}
    Period->>DB: Ambil pertanyaan periode
    Period-->>FE: Schema form

    Siswa->>FE: Isi jawaban dan upload file
    FE->>Answer: POST /api/form/answer
    Answer->>DB: Validasi periode, pertanyaan, required field, file
    Answer->>DB: Simpan data ke tb_answers
    Answer-->>FE: 200 Answers submitted successfully

    par Background setelah response submit
        Answer->>Notif: sendWhatsApp(template: new_submission)
        Notif->>Socket: POST /send-whatsapp
        Socket->>WA: Kirim WhatsApp ke siswa
    and
        Answer->>Socket: POST /notify-new-submission
        Socket-->>Admin: Realtime notification pendaftar baru
    end

    Note over Admin, Result: Verifikasi berkas
    Admin->>FE: Buka daftar submission
    FE->>Answer: GET /api/form/answer/group/respondent
    Answer->>DB: Ambil tb_answers, tb_results, tb_period
    Answer->>Status: Hitung validation_status
    Answer-->>FE: List submission + status berkas

    Admin->>FE: Approve / kembalikan berkas
    FE->>Result: POST /api/respondent/result/verify/{submissionId}
    Result->>DB: Create/update tb_results.is_approve
    Result->>Status: Hitung status berkas
    Result-->>FE: 201 Data verified successfully

    par Background setelah response verifikasi
        Result->>Socket: POST /notify-result-verified
        Socket-->>FE: Update UI realtime
    and
        alt berkas diterima
            Result->>Notif: sendWhatsApp(template: document_received)
        else berkas dikembalikan
            Result->>Notif: sendWhatsApp(template: document_rejected)
        end
        Notif->>Socket: POST /send-whatsapp
        Socket->>WA: Kirim WhatsApp ke siswa
    end

    Note over Admin, Result: Update hasil seleksi
    Admin->>FE: Isi selection_type, value, status lulus/tidak lulus
    FE->>Result: PUT /api/respondent/result/upadte/{submissionId}
    Result->>DB: Update tb_results.selection_type, value, status
    Result->>Status: Hitung validation_status terbaru
    Result-->>FE: 200 Data updated successfully

    alt periode sudah is_published
        Result->>Socket: POST /notify-result-updated
        alt status lulus
            Result->>Notif: sendWhatsApp(template: selection_result_passed)
        else status tidak lulus
            Result->>Notif: sendWhatsApp(template: selection_result_failed)
        end
        Notif->>Socket: POST /send-whatsapp
        Socket->>WA: Kirim WhatsApp hasil seleksi
    else periode belum is_published
        Result-->>FE: Hasil tersimpan untuk admin, belum dikirim sebagai hasil seleksi siswa
    end

    Note over Siswa, Answer: Status yang terlihat oleh siswa
    Siswa->>FE: Lihat status pendaftaran
    FE->>Answer: GET /api/form/answer/group/respondent/{submissionId}
    Answer->>DB: Ambil Answers, Period, Result
    Answer->>Status: Hitung validation_status
    alt role siswa dan periode belum is_published
        Answer-->>FE: validation_status tampil; result.selection_type/value/status = null
    else admin atau periode sudah is_published
        Answer-->>FE: validation_status + data result sesuai akses
    end
```

## Catatan Arsitektur

1. **Frontend React SPA** mengakses backend via REST API dan token Laravel Sanctum.
2. **AuthController** menangani login, OTP, registrasi, reset password, dan token Sanctum.
3. **AnswerController** menangani submit jawaban, upload file, grouping submission, dan response status pendaftaran siswa.
4. **ResultController** menangani verifikasi berkas dan update hasil seleksi.
5. **StatusHelper** menghitung `validation_status`, misalnya `Belum_Diverifikasi`, `Berkas_Diterima`, `Berkas_Dikembalikan`, `Lulus`, atau `Tidak_Lulus`.
6. **NotificationHelper** mengambil template WhatsApp dari database dan mengirim payload ke Socket.IO server melalui endpoint `/send-whatsapp`.
7. **Socket.IO server** menerima event dari Laravel melalui `SOCKET_SERVER_URL`, lalu meneruskan realtime update ke frontend atau pesan WhatsApp ke user.
8. **Database utama** pada flow ini: `users`, `otps`, `tb_period`, `tb_questions`, `tb_answers`, `tb_results`, `notification_whatsapp_messages`, dan `tb_notification`.

## Aturan Visibility Hasil

Siswa tetap bisa melihat `validation_status` untuk status berkas. Data hasil kelulusan (`selection_type`, `value`, dan `status`) tidak ditampilkan ke siswa ketika `tb_period.is_published = false`; backend mengirim field tersebut sebagai `null`. Admin/panitia tetap bisa melihat dan mengelola hasil sesuai kebutuhan operasional.

Route update hasil seleksi saat ini mengikuti kode backend: `PUT /api/respondent/result/upadte/{submissionId}`.
