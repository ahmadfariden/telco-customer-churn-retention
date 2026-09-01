# Business Recommendations — Telco Customer Churn & Retention Optimization

Dokumen ini ditujukan untuk pembaca non-teknis (manajemen, tim retensi, tim bisnis). Fokusnya pada **apa yang perlu dilakukan dan kenapa**, bukan detail teknis model. Untuk detail metodologi, lihat `methodology_notes.md`.

---

## Ringkasan Eksekutif

Dari 7,043 pelanggan yang dianalisis, **26.54% (1,869 pelanggan) telah berhenti berlangganan (churn)**. Dari populasi yang dianalisis lebih dalam (1,409 pelanggan sampel), diperkirakan **≈31% dari total pendapatan bulanan (≈$27,988) berisiko hilang** akibat churn.

Alih-alih menghubungi semua pelanggan berisiko secara merata, analisis ini menunjukkan bahwa **fokus pada 20% pelanggan berprioritas tertinggi sudah cukup untuk menjangkau 50% dari seluruh pelanggan yang berpotensi churn** — jauh lebih efisien dibanding kampanye retensi massal.

---

## Temuan Utama

### 1. Jenis Kontrak adalah Faktor Risiko Terbesar
Pelanggan dengan kontrak **bulanan (month-to-month)** churn di angka **42.71%**, dibanding hanya **2.83%** untuk kontrak dua tahun — perbedaan hampir 15 kali lipat.

**Artinya:** komitmen kontraktual jangka panjang secara alami melindungi dari churn.

### 2. Risiko Churn Tertinggi di Awal Masa Berlangganan
Pelanggan dengan masa berlangganan **0–6 bulan** memiliki churn rate **52.94%** — jauh di atas pelanggan lama (49+ bulan: 9.51%).

**Artinya:** periode awal adalah fase paling kritis untuk membangun loyalitas.

### 3. Layanan Fiber Optic Berasosiasi Kuat dengan Churn
Setelah dianalisis lebih dalam, penggunaan layanan internet **Fiber optic** adalah faktor pendorong churn paling kuat dari seluruh faktor yang diuji — lebih kuat dari sekadar tingginya tagihan bulanan.

**Artinya:** masalah kemungkinan bukan pada harga semata, melainkan pada kepuasan terhadap kualitas layanan Fiber optic itu sendiri.

### 4. Nilai Pelanggan Tinggi Tidak Menjamin Loyalitas
Pelanggan dengan tagihan bulanan tinggi (High Value) memiliki churn rate **31.56%** — hampir dua kali lipat dari pelanggan dengan tagihan rendah (Low Value: 16.38%).

**Artinya:** pelanggan paling berharga secara pendapatan justru butuh perhatian retensi yang setara, bukan diasumsikan otomatis loyal.

### 5. Segmen dengan Eksposur Pendapatan Terbesar Bukan Selalu yang "Paling Berisiko"
Segmen **Protect** (nilai tinggi, risiko belum "tinggi") ternyata menyimpan potensi kerugian pendapatan terbesar (23.2% dari total), melebihi segmen **Critical** (nilai tinggi + risiko tinggi, 21.0%) — karena jumlah pelanggannya lebih banyak dan nilainya sama-sama tinggi.

**Artinya:** strategi retensi tidak boleh hanya fokus ke pelanggan yang "sudah kelihatan mau pergi", tapi juga pelanggan bernilai tinggi yang belum menunjukkan tanda risiko jelas.

---

## Rekomendasi Aksi per Segmen Pelanggan

| Segmen | Karakteristik | Aksi yang Disarankan |
|---|---|---|
| **Critical** | Nilai tinggi + risiko churn tinggi | Penawaran retensi personal (diskon, program loyalitas langsung) — prioritas tertinggi |
| **Protect** | Nilai tinggi + risiko sedang | Proactive check-in dan loyalty perk preventif — cegah sebelum berubah jadi risiko tinggi |
| **Retain** | Nilai menengah + risiko tinggi | Tawaran migrasi kontrak + peninjauan layanan |
| **Monitor** | Nilai menengah, risiko bervariasi | Kampanye engagement otomatis (email/notifikasi aplikasi) — biaya rendah |
| **Maintain** | Nilai tinggi + risiko rendah | Pertahankan pengalaman saat ini, tidak perlu spending retensi agresif |
| **Low Cost** | Nilai rendah + risiko sedang | Intervensi otomatis biaya rendah |
| **Normal** | Nilai rendah + risiko rendah | Layanan standar, tanpa intervensi khusus |
| **Selective** | Nilai rendah + risiko tinggi, populasi sangat kecil | Evaluasi kasus per kasus — jangan buat kebijakan umum dari segmen ini |

---

## Rekomendasi Strategi Penargetan

Berdasarkan simulasi (dengan asumsi biaya retensi $15/pelanggan — **ilustratif**, bukan angka riil perusahaan):

| Strategi | Jangkauan Pelanggan | Cakupan Potensi Kerugian Pendapatan | Efisiensi Biaya |
|---|---|---|---|
| Kontak semua pelanggan (Mass) | 100% | 100% | Paling boros ($0.755 per $1 tercakup) |
| **Top 10% prioritas tertinggi** | 10% | 32.4% | **Paling efisien ($0.233 per $1 tercakup)** |
| Top 20% prioritas tertinggi | 20% | 56.3% | $0.268 per $1 tercakup |
| Top 30% prioritas tertinggi | 30% | 73.5% | $0.308 per $1 tercakup |

**Rekomendasi:** mulai dengan menargetkan **10–20% pelanggan berprioritas tertinggi** (berdasarkan kombinasi risiko churn dan nilai pendapatan) sebagai titik awal operasional yang realistis dan hemat biaya. Perluasan ke top 30% bisa dipertimbangkan jika kapasitas tim retensi memungkinkan dan hasil awal menunjukkan efektivitas.

---

## Rencana Aksi Bisnis (Ringkas)

| Siapa | Aksi | Kenapa | Ekspektasi Manfaat | Cara Mengukur |
|---|---|---|---|---|
| Pelanggan nilai tinggi + risiko tinggi | Retensi prioritas personal | Eksposur pendapatan tertinggi per pelanggan | Kurangi kehilangan pendapatan | Churn rate segmen ini dari waktu ke waktu |
| Pelanggan tenure awal (0-6 bulan) berisiko tinggi | Program onboarding intensif | Churn terkonsentrasi di fase awal | Tingkatkan retensi 90 hari pertama | Retention rate 90 hari |
| Pelanggan kontrak bulanan berisiko tinggi | Tawaran migrasi kontrak | Komitmen rendah adalah faktor risiko utama | Tingkatkan retensi jangka panjang | Tingkat konversi ke kontrak lebih panjang |
| Pengguna Fiber optic (semua segmen risiko) | Investigasi kepuasan layanan | Driver churn terkuat, kemungkinan masalah kualitas layanan | Kurangi churn rate Fiber optic | Survei kepuasan, churn rate Fiber optic dari waktu ke waktu |
| Pelanggan nilai rendah + risiko tinggi | Kampanye biaya rendah | Nilai ekonomi lebih rendah, ROI retensi mahal kemungkinan rendah | Kendalikan biaya retensi | Biaya per pelanggan yang dipertahankan |

---

## Batasan Penting yang Perlu Dipahami Sebelum Bertindak

1. **Rekomendasi ini berbasis pola historis, bukan bukti hasil nyata.** Perusahaan tidak memiliki data tentang apakah upaya retensi sebelumnya (diskon, kontak personal, dll) benar-benar berhasil mengurangi churn. Karena itu, angka-angka seperti "potensi pendapatan yang bisa diamankan" adalah **proyeksi skenario**, bukan jaminan hasil.

2. **Threshold penargetan (siapa yang dianggap "berisiko tinggi") saat ini ditentukan secara statistik**, bukan berdasarkan kapasitas tim retensi yang sesungguhnya. Sebelum diterapkan, sebaiknya disesuaikan dengan berapa banyak pelanggan yang benar-benar bisa dihubungi tim dalam periode tertentu.

3. **Segmen dengan jumlah pelanggan sangat kecil** (misalnya segmen "Selective") sebaiknya tidak dijadikan dasar kebijakan umum — datanya terlalu sedikit untuk disimpulkan sebagai pola yang kuat.

4. **Rekomendasi terkait Fiber optic dan faktor lain bersifat korelasional**, bukan sebab-akibat yang telah terbukti. Diperlukan investigasi lanjutan (misalnya survei kepuasan pelanggan) sebelum mengambil keputusan investasi besar berdasarkan temuan ini.

5. **Simulasi biaya retensi ($15/pelanggan) adalah angka ilustratif** untuk keperluan demonstrasi cara berpikir efisiensi biaya, bukan angka biaya riil perusahaan.

---

## Langkah Selanjutnya yang Disarankan

1. Validasi kapasitas operasional tim retensi (berapa pelanggan yang realistis bisa dihubungi per periode) untuk mengkalibrasi ulang ambang batas prioritas
2. Lakukan investigasi kualitas layanan Fiber optic sebagai tindak lanjut atas temuan driver churn terkuat
3. Jika memungkinkan, mulai kumpulkan data hasil intervensi retensi (siapa yang dihubungi, aksi apa yang diberikan, apakah mereka tetap bertahan) — ini akan memungkinkan analisis yang lebih kuat (causal/uplift) di masa depan
4. Uji coba strategi Top 10-20% pada skala kecil terlebih dahulu sebelum diperluas, untuk memvalidasi asumsi efektivitas di dunia nyata

---

## Referensi Silang

- Detail teknis dan metodologi: `methodology_notes.md`
- Hasil dan angka lengkap: `README.md`
- Definisi setiap kolom data: `data_dictionary.md`
