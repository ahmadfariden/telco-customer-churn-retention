# Methodology Notes — Telco Customer Churn Analytics & Retention Optimization

Dokumen ini menjelaskan **bagaimana** setiap angka dan keputusan di proyek ini dihasilkan — pendekatan teknis, asumsi, dan alasan di balik setiap pilihan metodologis. Untuk hasil dan temuan, lihat `README.md`. Untuk kode lengkap, lihat `python-scripts-lengkap.md`.

---

## 1. Data & Scope

- **Dataset:** IBM Telco Customer Churn, 7,043 baris, 21 kolom
- **Unit analisis:** 1 baris = 1 customer, diidentifikasi oleh `customerID`
- **Target:** `Churn` (Yes/No), diencode menjadi `Churn_Flag` (1/0)

### Catatan Penting soal Scope Data di Seluruh Dashboard
Ada dua populasi berbeda yang dipakai di proyek ini, dan **tidak boleh dicampur tanpa disebutkan eksplisit**:

| Populasi | Ukuran | Dipakai untuk |
|---|---|---|
| **Full population** | 7,043 customer | EDA deskriptif (churn rate by contract/tenure/payment), total revenue |
| **Test set (scored sample)** | 1,409 customer (20% dari full population) | Semua metrik yang butuh output model: churn probability, revenue at risk, risk group, priority ranking, lift/gain |

Alasan test set lebih kecil: model dilatih di 80% data (train set, 5,634 customer) dan probabilitasnya hanya dievaluasi/dipakai di 20% sisanya (test set) untuk menghindari data leakage — kalau probability dihitung di data yang sama dengan yang dipakai melatih model, hasilnya akan overfit dan tidak representasi performa di dunia nyata.

Setiap kartu/visual di Power BI yang bersumber dari `retention_strategy_final`, `customer_retention_scoring`, `lift_gain_table`, atau `quantitative_tradeoff` otomatis berskala ke 1,409 customer. Ini sudah ditandai eksplisit di dashboard dengan label "(Scored Sample)" dan catatan tambahan.

---

## 2. Data Cleaning

### 2.1 Masalah `TotalCharges`
Kolom ini terbaca sebagai `string`/`object`, bukan numeric, karena 11 baris memiliki value kosong/spasi (`" "`). Setelah diinvestigasi, semua 11 baris tersebut punya `tenure = 0` (customer baru, belum ada histori tagihan) — bukan data korup, melainkan kondisi bisnis yang valid.

**Keputusan:** isi `TotalCharges = 0` untuk baris dengan `tenure = 0`, bukan drop baris. Alasan: mereka tetap customer yang valid untuk dianalisis (beberapa churn, beberapa tidak), dan menghapusnya berarti kehilangan informasi tanpa alasan yang kuat.

### 2.2 `customerID` Bukan Fitur
`customerID` di-drop dari `X` (fitur model) tapi tetap disimpan sepanjang pipeline untuk keperluan drill-through, scoring per-customer, dan retention target list. Ini mencegah model "menghafal" ID individual alih-alih mempelajari pola yang generalizable.

---

## 3. Feature Engineering

Prinsip yang dipegang: **fitur dibuat berdasarkan alasan bisnis, bukan untuk menambah kompleksitas.** Fitur turunan matematis murni (misal `MonthlyCharges × tenure`, `log(MonthlyCharges)`) sengaja tidak dibuat kecuali terbukti dibutuhkan.

| Fitur | Definisi | Alasan Bisnis |
|---|---|---|
| `Tenure_Group` | Bucket 0-6, 7-12, 13-24, 25-48, 49+ bulan | Mempermudah analisis pola churn per fase siklus hidup customer |
| `Service_Count` | Jumlah layanan aktif (dari 9 kolom servis) | Proxy keterikatan customer terhadap ekosistem layanan |
| `Family_Status` | Gabungan `Partner` + `Dependents` | Menangkap konteks rumah tangga tanpa membuat kombinasi kategori berlebihan |
| `Contract_Risk_Group` | Relabel netral dari `Contract` (Flexible/Mid-Term/Long-Term) | Label dibuat netral secara sengaja — **tidak** langsung dinamai "High/Low Risk" agar tidak menyimpulkan sebelum EDA membuktikan |
| `Payment_Risk_Group` | Kelompokkan `PaymentMethod` menjadi Manual vs Automatic | Sama seperti di atas — netral, divalidasi lewat data |

⚠️ **Redundansi yang diketahui dan disengaja dibiarkan:** `Contract_Risk_Group` adalah relabel langsung dari `Contract`, `Payment_Risk_Group` dari `PaymentMethod`. Ini terkonfirmasi di koefisien model (Bagian 6) — kedua pasang fitur ini menghasilkan koefisien identik karena secara matematis membawa informasi yang sama. Diperlakukan sebagai **satu driver**, bukan bukti ganda.

---

## 4. Modeling

### 4.1 Train/Test Split
- **Metode:** `train_test_split` dengan `stratify=y`, `test_size=0.2`, `random_state=42`
- **Alasan stratify:** menjaga proporsi churn (26.5%) identik di train dan test set — penting karena dataset imbalanced, tanpa stratify proporsi bisa melenceng di split acak
- **Verifikasi:** train churn rate = 0.2654, test churn rate = 0.2654 (identik, stratify berhasil)

### 4.2 Preprocessing
- **Kolom numerik** (`SeniorCitizen`, `tenure`, `MonthlyCharges`, `TotalCharges`, `Service_Count`): `StandardScaler`
- **Kolom kategorikal** (19 kolom termasuk fitur turunan): `OneHotEncoder(handle_unknown="ignore", drop="if_binary")`
- Digabung lewat `ColumnTransformer` dalam satu `Pipeline` — memastikan transformasi konsisten antara train dan test, dan mencegah data leakage dari scaling/encoding yang "melihat" data test saat fit

### 4.3 Model yang Dibandingkan

| Model | Alasan Dipilih untuk Dibandingkan |
|---|---|
| Logistic Regression | Baseline interpretable — koefisien bisa langsung dijelaskan arah pengaruhnya |
| Random Forest | Benchmark nonlinear — bisa menangkap interaksi antar fitur yang tidak ditangkap model linear |

Keduanya menggunakan `class_weight="balanced"` untuk mengompensasi imbalance (26.5% churn) tanpa perlu oversampling/undersampling eksplisit.

### 4.4 Model Terpilih: Logistic Regression

| Metrik | Logistic Regression | Random Forest |
|---|---|---|
| ROC-AUC | 0.8450 | 0.8403 |
| PR-AUC | 0.6513 | 0.6524 |
| Recall (churn) | 0.80 | 0.78 |
| Precision (churn) | 0.51 | 0.53 |

Performa keduanya secara statistik setara (selisih <1 poin persentase di sebagian besar metrik). Logistic Regression dipilih karena:
1. **Interpretability** — dibutuhkan untuk Churn Driver Analysis (Bagian 6), di mana koefisien harus bisa dijelaskan arah pengaruhnya ke stakeholder non-teknis
2. **Recall sedikit lebih tinggi** — dalam konteks retensi, false negative (customer churn yang tidak terdeteksi) lebih mahal daripada false positive (customer stabil yang salah di-flag)

---

## 5. Threshold Tuning

Default threshold 0.5 tidak diambil begitu saja. Diuji 5 titik (0.3, 0.4, 0.5, 0.6, 0.7), diukur precision, recall, F1, dan jumlah customer yang ter-flag.

**Threshold dipilih: 0.6** (F1 tertinggi = 0.625).

**Batasan yang diakui secara eksplisit:** pemilihan ini murni berdasarkan keseimbangan matematis precision-recall (F1), **bukan** berdasarkan kapasitas retensi riil — dataset ini tidak menyediakan informasi berapa customer yang sanggup dikontak tim retensi dalam periode tertentu. Jika diterapkan di dunia nyata, threshold ini adalah parameter pertama yang harus disesuaikan dengan kapasitas operasional aktual.

---

## 6. Probability Calibration

### Kenapa Diperlukan
`class_weight="balanced"` efektif meningkatkan recall, tapi punya efek samping: probability yang dihasilkan model menjadi **overpredict secara sistematis**. Contoh: pada bin prediksi rata-rata 0.547, kenyataan aktual churn hanya 0.292.

Ini penting karena tahap berikutnya (Revenue at Risk, Retention Priority) menggunakan probability sebagai **pengali langsung** terhadap MonthlyCharges — kalau probability tidak akurat, seluruh estimasi revenue at risk akan bias ke atas.

### Metode
`CalibratedClassifierCV` dengan `method="sigmoid"`, `cv=5` — dipilih sigmoid (bukan isotonic) karena ukuran dataset test relatif kecil (1,409), dan isotonic regression cenderung overfit pada sampel kecil.

### Hasil
| | Brier Score |
|---|---|
| Uncalibrated | 0.1659 |
| **Calibrated** | **0.1363** |

Brier Score turun (lebih baik), dan reliability table menunjukkan predicted probability jauh lebih nempel ke actual rate setelah kalibrasi. **Semua probability yang dipakai di tahap selanjutnya adalah versi terkalibrasi.**

---

## 7. Customer Value Proxy

### Keputusan Desain: MonthlyCharges sebagai Basis Tunggal
Alternatif yang dipertimbangkan tapi **tidak dipakai**: skor komposit dari `MonthlyCharges` + `TotalCharges` + `tenure` dengan bobot tertentu (misal 40/40/20). Ditolak karena bobot semacam itu **arbitrary** — tidak ada dasar yang bisa dipertanggungjawabkan kenapa memilih angka bobot tertentu dibanding yang lain.

**Metode yang dipakai:** `MonthlyCharges` sebagai satu-satunya basis value, dengan alasan rantai logika yang jelas:
```
Customer churn → kehilangan recurring revenue → MonthlyCharges
```
`TotalCharges` dan `tenure` tetap dilaporkan sebagai **context metrics** (lihat tabel context check di README), tapi tidak masuk perhitungan skor.

### Pembagian Grup
`pd.qcut` (kuantil) dengan `q=3` — dipilih karena menghasilkan grup dengan jumlah customer yang seimbang (masing-masing ~470), lebih defensible dibanding batas angka arbitrary (misal "< $35 = Low Value").

---

## 8. Risk Group & Risk × Value Matrix

### Cutoff Risk Group
Bukan angka baru yang dibuat sembarangan — cutoff **0.3** dan **0.6** diambil langsung dari titik-titik yang sudah dianalisis di Threshold Tuning (Bagian 5), sehingga setiap batas grup risiko punya dasar precision/recall yang sudah diketahui:
- Low Risk: < 0.3
- Medium Risk: 0.3 – 0.6
- High Risk: > 0.6

### Segmentasi Bisnis
Value Group (3 level) × Risk Group (3 level) menghasilkan 8 kombinasi (bukan 9, karena Medium Value diperlakukan seragam sebagai "Monitor" terlepas dari risk level, untuk menyederhanakan aksi tanpa kehilangan makna bisnis). Lihat README untuk detail definisi tiap segmen.

⚠️ **Peringatan overinterpretasi:** segmen dengan populasi kecil (misal "Selective", n=11) rentan menunjukkan angka ekstrem (churn rate 100%) yang murni kebetulan sampel kecil, bukan pola statistik yang kuat. Ini dicatat eksplisit agar tidak disalahartikan sebagai temuan robust.

---

## 9. Revenue at Risk & Retention Priority

### Formula
```
Expected Revenue at Risk = Churn Probability (calibrated) × MonthlyCharges
```

### Kenapa Tidak Pakai Skor Komposit untuk Priority Ranking
Alternatif yang dipertimbangkan: `Retention Priority Score = Churn Probability × Customer Value Score`. Ditolak karena **double-counting** — jika Customer Value Score sudah dibangun dari MonthlyCharges (Bagian 7), maka mengalikannya lagi dengan MonthlyCharges secara implisit (lewat probability) membuat interpretasi jadi kabur.

**Keputusan:** ranking langsung memakai `Expected Revenue at Risk`, sehingga bisa dijelaskan dengan satu kalimat yang jelas: *"Customer ini diprioritaskan karena probability churn tinggi DAN recurring revenue yang berisiko hilang juga tinggi."*

---

## 10. Lift & Cumulative Gain Analysis

### Kenapa Dua Metrik Dipisah
Lift dan Cumulative Gain sering disebut bersamaan tapi **mengukur hal berbeda**:
- **Cumulative Gain** — dari seluruh churner aktual di populasi, berapa persen tertangkap kalau menarget top-N% customer?
- **Lift** — seberapa lebih pekat (concentrated) churner di top-N% dibanding random targeting?

Keduanya dihitung dan dilaporkan terpisah untuk menghindari kesan bahwa satu angka/chart mewakili dua konsep berbeda.

### Baseline
`baseline_churn_rate` = churn rate keseluruhan test set (26.5%) — merepresentasikan performa "random targeting" sebagai pembanding lift.

---

## 11. Churn Driver Analysis

### Metode
Koefisien dari `LogisticRegression` (setelah `OneHotEncoder`/`StandardScaler`) diambil langsung, diurutkan berdasarkan nilai absolut. Koefisien positif = menaikkan risiko churn; negatif = menurunkan.

### Catatan Interpretasi Penting: EDA vs Koefisien Model
Ditemukan **perbedaan arah** antara EDA univariate dan koefisien multivariate untuk `MonthlyCharges`:
- **EDA (univariate):** Medium/High Value churn rate (~31%) lebih tinggi dari Low Value (16%)
- **Koefisien model (multivariate, dikontrol variabel lain):** `MonthlyCharges` berkoefisien negatif (-0.65)

**Penjelasan:** setelah mengontrol `InternetService_Fiber optic` (koefisien +0.72, driver terbesar), efek "MonthlyCharges tinggi → churn tinggi" yang terlihat di EDA sebagian besar ternyata didorong oleh Fiber optic, bukan harga itu sendiri secara independen. Ini bukan kontradiksi, melainkan perbedaan cara baca antara analisis univariate (EDA) dan multivariate (model) — keduanya valid untuk konteks masing-masing, tapi tidak boleh dicampur tanpa penjelasan.

### Fitur Berstatus "Hypothesis"
`Service_Count` (+0.37) dan `Paperless Billing = Yes` (+0.35) ditandai sebagai **hipotesis**, bukan driver yang mekanismenya sudah jelas — keduanya korelasi yang belum punya penjelasan kausal yang meyakinkan, dan butuh investigasi lanjut sebelum dijadikan dasar aksi besar.

---

## 12. Quantitative Trade-Off

### Asumsi Biaya
`RETENTION_COST_PER_CUSTOMER = $15` — **ilustratif, bukan data riil**. Dataset tidak menyediakan informasi biaya retensi aktual. Angka ini dipakai semata untuk mendemonstrasikan cara berpikir cost-efficiency, bukan sebagai rekomendasi biaya yang presisi.

### Definisi "Hybrid" Strategy
```
Hybrid = Top 20% model-prioritized + 10% random exploration sample
```
Sengaja **tidak** disebut sebagai "treatment/control group" karena istilah itu menyiratkan desain eksperimen formal (randomization + outcome yang diukur secara sistematis), yang tidak dilakukan di sini. 10% tambahan di luar Top 20% murni sampel acak untuk eksplorasi, bukan kelompok kontrol dalam pengertian A/B testing.

### Penamaan Metrik "Cost per $ Revenue at Risk Addressed"
Istilah "addressed/targeted" dipilih dengan sengaja, **bukan** "saved" atau "recovered" — karena metrik ini adalah **scenario metric** (revenue yang masuk cakupan target strategi secara hipotetis), bukan revenue yang benar-benar terbukti terselamatkan. Dataset tidak memiliki data historical treatment/outcome untuk memvalidasi klaim penyelamatan revenue yang sesungguhnya.

---

## 13. Limitasi Menyeluruh

> **Dataset ini tidak memiliki data historical treatment/outcome retensi.** Tidak diketahui apakah upaya retensi di masa lalu (diskon, kontak personal, dll) benar-benar mengurangi churn.

Implikasi metodologis:
- Semua analisis di proyek ini bersifat **risk-based prioritization** (mengurutkan berdasarkan risiko × nilai) dan **scenario-based simulation** (membandingkan strategi hipotetis dengan asumsi eksplisit) — **bukan** causal uplift modeling.
- Klaim seperti "strategi X akan menyelamatkan Y% revenue" tidak didukung oleh data ini dan sengaja dihindari di seluruh dokumentasi proyek.
- Angka-angka "Revenue at Risk", "Cost per $ Addressed", dan sejenisnya harus dibaca sebagai **potensi/skenario**, bukan hasil yang telah terbukti terjadi.

Limitasi tambahan spesifik per tahap sudah dicatat di bagian masing-masing di atas (redundansi fitur di Bagian 3, overinterpretasi sampel kecil di Bagian 8, status hipotesis driver di Bagian 11).

---

## Referensi Silang

- Hasil dan temuan lengkap: `README.md`
- Kode lengkap tiap tahap: `python-scripts-lengkap.md`
- File output per tahap: lihat tabel "File Output yang Dihasilkan" di `README.md`
