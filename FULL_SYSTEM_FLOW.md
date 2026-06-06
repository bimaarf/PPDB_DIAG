# Full System Flow - PSB Digital Architecture

Dokumen ini menjelaskan alur utama sistem PSB dengan bahasa proses: pengguna login, siswa mengisi formulir pendaftaran, admin/panitia memeriksa berkas, lalu admin/panitia mengisi hasil seleksi. Flow ini hanya memuat fitur yang dipakai pada proses utama.

```mermaid
sequenceDiagram
    autonumber
    actor Siswa
    actor Admin as Admin/Panitia
    participant FE as Halaman Web
    participant Auth as Layanan Login
    participant Period as Layanan Formulir
    participant Answer as Layanan Pendaftaran
    participant Result as Layanan Berkas dan Hasil
    participant Status as Aturan Status
    participant Notif as Layanan Notifikasi
    participant DB as Database
    participant Socket as Server Realtime
    participant WA as WhatsApp

    Note over Siswa, Auth: Login pengguna
    Siswa->>FE: Login
    FE->>Auth: Kirim data login ke server
    Auth->>DB: Cocokkan akun, password, status, dan role
    alt akun sudah terverifikasi OTP
        Auth-->>FE: Login berhasil, masuk dashboard sesuai role
    else akun belum terverifikasi OTP
        Auth->>DB: Gunakan OTP aktif atau buat OTP baru
        Auth->>WA: Kirim kode OTP ke WhatsApp siswa
        Auth-->>FE: Tampilkan kolom OTP dan masa berlaku kode
        Siswa->>FE: Masukkan OTP
        FE->>Auth: Kirim kode OTP untuk diperiksa
        Auth->>DB: Cek OTP masih berlaku dan belum dipakai
        Auth-->>FE: Login berhasil, masuk dashboard sesuai role
    end

    Note over Siswa, Answer: Pengisian formulir pendaftaran
    Siswa->>FE: Buka periode/form PPDB
    FE->>Period: Minta daftar periode pendaftaran
    Period->>DB: Cari periode yang sedang dibuka
    Period-->>FE: Tampilkan periode tersedia
    FE->>Period: Minta daftar pertanyaan pada periode
    Period->>DB: Ambil pertanyaan formulir
    Period-->>FE: Tampilkan formulir pendaftaran

    Siswa->>FE: Isi jawaban dan unggah berkas
    FE->>Answer: Kirim jawaban dan file ke server
    Answer->>DB: Pastikan periode aktif, jawaban lengkap, dan file sesuai aturan
    Answer->>DB: Simpan jawaban, path file S3, dan nomor pendaftaran
    Answer-->>FE: Formulir berhasil dikirim

    par Notifikasi setelah formulir tersimpan
        Answer->>Notif: Minta kirim pesan formulir berhasil diterima
        Notif->>Socket: Teruskan pesan ke gateway WhatsApp
        Socket->>WA: Kirim WhatsApp ke siswa
    and
        Answer->>Socket: Kabari dashboard admin
        Socket-->>Admin: Notifikasi pendaftar baru secara realtime
    end

    Note over Admin, Result: Verifikasi berkas
    Admin->>FE: Buka daftar pendaftar
    FE->>Answer: Minta daftar pendaftar
    Answer->>DB: Ambil data pendaftar, berkas, hasil, dan periode
    Answer->>Status: Hitung status berkas
    Answer-->>FE: Tampilkan daftar pendaftar dan status berkas

    Admin->>FE: Pilih berkas diterima atau dikembalikan
    FE->>Result: Kirim keputusan verifikasi berkas
    Result->>DB: Simpan status verifikasi berkas
    Result->>Status: Hitung status berkas
    Result-->>FE: Berkas berhasil diverifikasi

    par Notifikasi setelah verifikasi
        Result->>Socket: Kabari dashboard bahwa berkas sudah dicek
        Socket-->>FE: Perbarui tampilan secara realtime
    and
        alt berkas diterima
            Result->>Notif: Siapkan pesan berkas diterima
        else berkas dikembalikan
            Result->>Notif: Siapkan pesan berkas dikembalikan
        end
        Notif->>Socket: Teruskan pesan ke gateway WhatsApp
        Socket->>WA: Kirim WhatsApp ke siswa
    end

    Note over Admin, Result: Pengisian hasil seleksi
    Admin->>FE: Isi jenis seleksi, nilai, dan status lulus/tidak lulus
    FE->>Result: Kirim hasil seleksi akhir ke server
    Result->>DB: Simpan hasil seleksi
    Result->>Status: Hitung status terbaru yang boleh ditampilkan
    Result-->>FE: Hasil seleksi berhasil diperbarui

    alt periode sudah dipublish
        Result->>Socket: Kabari dashboard bahwa hasil diperbarui
        alt status lulus
            Result->>Notif: Siapkan pesan siswa lulus
        else status tidak lulus
            Result->>Notif: Siapkan pesan siswa tidak lulus
        end
        Notif->>Socket: Teruskan pesan ke gateway WhatsApp
        Socket->>WA: Kirim WhatsApp hasil seleksi
    else periode belum dipublish
        Result-->>FE: Hasil tersimpan untuk admin, belum dikirim sebagai hasil seleksi siswa
    end

    Note over Siswa, Answer: Status yang terlihat oleh siswa
    Siswa->>FE: Lihat status pendaftaran
    FE->>Answer: Minta detail status pendaftaran
    Answer->>DB: Ambil jawaban, periode, dan hasil pendaftaran
    Answer->>Status: Tentukan status yang boleh dilihat siswa
    alt siswa membuka periode yang belum dipublish
        Answer-->>FE: Status berkas tampil, hasil final masih kosong
    else admin/panitia atau periode sudah dipublish
        Answer-->>FE: Status berkas dan data hasil tampil sesuai akses
    end
```

## Catatan Arsitektur

1. **Halaman Web** dipakai siswa dan admin untuk mengirim data ke server.
2. **Layanan Login** menangani login, OTP, registrasi, reset password, dan token login.
3. **Layanan Pendaftaran** menyimpan jawaban siswa, path file dari bucket storage S3, dan nomor pendaftaran.
4. **Layanan Berkas dan Hasil** menangani verifikasi berkas dan hasil seleksi.
5. **Aturan Status** menentukan status yang boleh terlihat oleh siswa atau admin.
6. **Layanan Notifikasi** mengambil template WhatsApp dan mengirim pesan melalui gateway.
7. **Server Realtime** meneruskan perubahan status ke dashboard dan WhatsApp.
8. **Database utama** pada flow ini: `users`, `otps`, `tb_period`, `tb_questions`, `tb_answers`, `tb_results`, `notification_whatsapp_messages`, dan `tb_notification`.

## Aturan Visibility Hasil

Siswa tetap bisa melihat status berkas. Jika periode belum dipublish, hasil kelulusan belum ditampilkan ke siswa. Secara teknis backend mengosongkan field `selection_type`, `value`, dan `status` sampai `tb_period.is_published = true`. Admin/panitia tetap bisa melihat dan mengelola hasil sesuai kebutuhan operasional.

Route update hasil seleksi saat ini mengikuti kode backend: `PUT /api/respondent/result/upadte/{submissionId}`.
