# MBG Guard

Prototipe sistem peringatan dini keamanan dan kualitas makanan untuk program Makan Bergizi Gratis (MBG). Keluhan siswa diklasifikasikan oleh workflow Langflow, sinyalnya dinilai dengan aturan yang transparan, lalu petugas menelusurinya lewat percakapan dengan IBM Bob.

**Prinsip:** sistem tidak menyatakan makanan berbahaya dan tidak menentukan penyebab keracunan. Sistem hanya menandai pola yang perlu diperiksa manusia, beserta buktinya. Siswa hanya menerima dua kalimat tetap, bukan hasil generate AI.

> \\\*\\\*Status:\\\*\\\* prototipe hackathon dengan data contoh. `MBG\\\_Guard.html` adalah demonstrasi tampilan. Kode pemanggil Langflow sudah ditulis di dalamnya, tetapi belum diuji menyeluruh, dan Bob tidak terhubung ke halaman itu. Lihat \\\[Status pengujian](#status-pengujian) dan \\\[Keterbatasan](#keterbatasan).

## Isi repositori

|Berkas|Fungsi|
|-|-|
|`MBG\\\_Guard.html`|Tampilan satu berkas: login murid/petugas, jadwal MBG, tulis review, cek review, ringkasan|
|`flows/MBG\\\_Guard\\\_-\\\_Flow\\\_A\\\_Klasifikasi.json`|Flow A: Gemini mengubah teks siswa menjadi data terstruktur. Balasannya JSON `{reply, valid, items}`|
|`flows/MBG\\\_Guard\\\_-\\\_Flow\\\_B\\\_Rule-Based\\\_Detection.json`|Flow B: aturan deteksi, status hijau/kuning/merah per sekolah beserta bukti (tanpa AI)|
|`flows/MBG\\\_Guard\\\_-\\\_Flow\\\_C\\\_Daily\\\_Summary.json`|Flow C: ringkasan per sekolah dari keluaran Flow B (tanpa AI)|
|`bob\\\_custom\\\_mode\\\_fields.md`|Isi custom mode Bob untuk petugas|
|`Proposal\\\MBG\\\Guard.docx`|Proposal|
|`demo/demo\\\_payload\\\_flow\\\_b.json`|Contoh input Flow B (data buatan tangan)|

## Cara kerja

```
Murid menulis review -> Flow A (Gemini) -> {reply, valid, items}
   siswa hanya melihat kalimat tetap; hasil klasifikasi dipakai petugas

Data review + catatan operasional -> Flow B (aturan) -> Flow C (ringkasan)

Bob (MCP, server lf-mbg) memanggil Flow A/B/C dan menjelaskan hasilnya kepada petugas
```

### Aturan deteksi (Flow B)

* **Merah:** minimal 1 keluhan severity tinggi (misalnya gejala fisik atau benda asing), atau 3 keluhan serupa atau lebih untuk menu dan kategori yang sama. Merah artinya perlu ditinjau segera oleh manusia, bukan kesimpulan bahwa makanan berbahaya.
* **Kuning:** keluhan yang perlu ditinjau (kurang dari 3 serupa), catatan operasional belum lengkap, atau pengiriman terlambat.
* **Hijau:** tidak ada sinyal yang memenuhi aturan di atas.

### Format input Flow B

Satu string JSON. Ketiga daftar wajib ada (boleh kosong):

```json
{
  "schools": \\\[{"school\\\_id": "", "school\\\_name": ""}],
  "operational\\\_records": \\\[{"school\\\_id": "", "temperature\\\_c": 65, "serving\\\_time": "12:00",
                           "distribution\\\_status": "on\\\_time|late", "last\\\_updated": "2026-10-05 12:00"}],
  "student\\\_reviews": \\\[{"school\\\_id": "", "jenis": "", "menu": "", "category": "", "symptom": "",
                       "severity": "rendah|sedang|tinggi", "needs\\\_review": true}]
}
```

`student\\\_reviews` berasal dari `items` pada keluaran Flow A, ditambah `school\\\_id`.

## Menjalankan

Yang dibutuhkan: Langflow yang berjalan di komputermu, browser, dan Python (untuk server lokal sederhana).

### Tampilan saja (tanpa Langflow)

1. Di folder repo: `python -m http.server 8000`, lalu buka `http://localhost:8000/MBG\\\_Guard.html`. Jangan klik dua kali berkasnya.
2. Login, jadwal, cek review (data contoh), dan profil bisa dicoba. **Kirim review dan Ringkasan butuh Langflow** dan akan menampilkan pesan gagal terhubung.

### Dengan Langflow (eksperimental, belum teruji)

1. Impor ketiga flow di folder `flows/` ke satu project Langflow. Isi API key Google AI di node *Language Model* pada Flow A.
2. Salin `MBG\\\_Guard.html` menjadi `MBG\\\_Guard.local.html` (berkas ini sudah diabaikan oleh `.gitignore`). Di salinan itu, cari `const CFG=` lalu isi alamat Langflow, API key Langflow, dan ID tiap flow (bagian setelah `/flow/` pada alamat flow di browser).
3. Buka `MBG\\\_Guard.local.html` lewat server lokal seperti di atas. Kalau muncul error CORS atau gagal terhubung, periksa alamat, API key, dan apakah Langflow sedang berjalan.

### Dengan IBM Bob (opsional)

Daftarkan Langflow sebagai server MCP (kami memakai nama `lf-mbg`), buat custom mode dari `bob\\\_custom\\\_mode\\\_fields.md`, dan pastikan mode itu boleh memakai tool MCP. Cek dengan bertanya "tools apa saja yang kamu punya?": tool Flow A, B, dan C harus muncul. Untuk Flow B dan C, minta Bob membaca `demo/demo\\\_payload\\\_flow\\\_b.json` sebagai data contoh. Menurut aturan Flow B, hasilnya: SDN Merdeka 01 dan SMPN Nusantara 03 merah, SDN Harapan Bangsa 02 hijau.

## Status pengujian

|Bagian|Status|
|-|-|
|Flow A, B, C dijalankan sendiri-sendiri di Langflow|Sudah dicoba dengan data contoh|
|Bob memanggil tool Flow A/B/C lewat MCP|Tool terdeteksi dan dipanggil; alur lengkap dengan custom mode terbaru belum diverifikasi ulang|
|`MBG\\\_Guard.html` memanggil Flow A, B, C|Kode ada, belum diuji menyeluruh|
|Akurasi klasifikasi pada data berlabel|Belum diuji|

## Keterbatasan

* **Tampilan belum terhubung penuh.** Lihat status di atas. Tanpa Langflow, hanya tampilan dan data contoh yang berjalan.
* **Belum ada database.** Review disimpan di `localStorage` browser, jadi petugas di perangkat lain tidak melihat review siswa.
* **Login hanya tiruan.** Email apa pun diterima, tanpa kata sandi, dan murid memilih sekolahnya saat masuk. Rancangan akhir: sekolah ditentukan oleh akun atau kode sekolah.
* **Review berlabel "contoh"** disimulasikan dengan aturan sederhana di browser, bukan oleh Flow A.
* **Keputusan petugas belum disimpan.** Bob mencatatnya hanya di dalam percakapan.
* **Belum ada formulir catatan operasional** untuk guru/dapur. Flow B sudah membaca datanya, tetapi tampilan mengirim daftar kosong.
* **Batas suhu dan jeda masak-sajian belum dinilai**, menunggu ambang dari pihak berwenang.
* **Flow C belum punya filter periode:** semua data yang dikirim ikut dihitung.
* **Batas 10 review per hari** hanya diterapkan di browser, sehingga mudah dilewati.

## Keamanan dan privasi

* **Jangan pernah commit API key.** Di versi ini kunci Langflow ditulis langsung di `const CFG=` dalam berkas HTML, jadi isi hanya di salinan `MBG\\\_Guard.local.html`. Berkas di repo harus tetap berisi placeholder.
* Halaman memanggil Langflow langsung dari browser, sehingga kunci terbaca oleh siapa saja yang memakai browser itu. Untuk penggunaan nyata, panggilan harus lewat server perantara.
* Keluaran Flow A hanya boleh dipakai petugas. Layar siswa hanya menampilkan `reply`.
* Kolom gejala adalah data sensitif anak: pakai data contoh untuk demo dan batasi aksesnya pada penggunaan nyata.

