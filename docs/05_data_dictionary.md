# Data Dictionary — Telco Customer Churn Analytics & Retention Optimization

Dokumen ini mendefinisikan setiap kolom yang digunakan di proyek ini: kolom asli dari dataset, fitur turunan hasil feature engineering, dan kolom hasil analisis (skor, segmen, ranking) di file-file output.

---

## 1. Kolom Asli (dari `WA_Fn-UseC_-Telco-Customer-Churn.csv`)

| Kolom | Tipe | Deskripsi | Nilai yang Mungkin |
|---|---|---|---|
| `customerID` | string | ID unik customer. **Bukan fitur model** — hanya identifier untuk drill-through dan scoring | Format: `XXXX-XXXXX` |
| `gender` | string | Jenis kelamin customer | `Male`, `Female` |
| `SeniorCitizen` | int | Indikator apakah customer adalah lansia | `0` (bukan), `1` (ya) |
| `Partner` | string | Apakah customer memiliki pasangan | `Yes`, `No` |
| `Dependents` | string | Apakah customer memiliki tanggungan | `Yes`, `No` |
| `tenure` | int | Lama berlangganan dalam bulan | 0–72 |
| `PhoneService` | string | Apakah berlangganan layanan telepon | `Yes`, `No` |
| `MultipleLines` | string | Apakah punya lebih dari satu jalur telepon | `Yes`, `No`, `No phone service` |
| `InternetService` | string | Jenis layanan internet | `DSL`, `Fiber optic`, `No` |
| `OnlineSecurity` | string | Apakah berlangganan layanan keamanan online | `Yes`, `No`, `No internet service` |
| `OnlineBackup` | string | Apakah berlangganan layanan backup online | `Yes`, `No`, `No internet service` |
| `DeviceProtection` | string | Apakah berlangganan proteksi perangkat | `Yes`, `No`, `No internet service` |
| `TechSupport` | string | Apakah berlangganan dukungan teknis | `Yes`, `No`, `No internet service` |
| `StreamingTV` | string | Apakah berlangganan streaming TV | `Yes`, `No`, `No internet service` |
| `StreamingMovies` | string | Apakah berlangganan streaming film | `Yes`, `No`, `No internet service` |
| `Contract` | string | Jenis kontrak | `Month-to-month`, `One year`, `Two year` |
| `PaperlessBilling` | string | Apakah menggunakan tagihan tanpa kertas | `Yes`, `No` |
| `PaymentMethod` | string | Metode pembayaran | `Electronic check`, `Mailed check`, `Bank transfer (automatic)`, `Credit card (automatic)` |
| `MonthlyCharges` | float | Tagihan bulanan dalam $ | Numerik kontinu |
| `TotalCharges` | float | Total tagihan sejak awal berlangganan dalam $ | Numerik kontinu. **Dibersihkan**: awalnya string dengan 11 baris blank (tenure=0), diisi 0 |
| `Churn` | string | **Target** — apakah customer berhenti berlangganan | `Yes`, `No` |

---

## 2. Fitur Turunan (Feature Engineering, Fase 1.2)

| Fitur | Tipe | Formula/Logika | Alasan Bisnis |
|---|---|---|---|
| `Churn_Flag` | int (0/1) | `1` jika `Churn == "Yes"`, else `0` | Encoding target untuk model |
| `Tenure_Group` | string (kategori) | Bucket dari `tenure`: `0-6`, `7-12`, `13-24`, `25-48`, `49+` | Analisis pola churn per fase siklus hidup customer |
| `Service_Count` | int | Jumlah layanan aktif dari 9 kolom servis (`PhoneService`, `MultipleLines`, `InternetService`, `OnlineSecurity`, `OnlineBackup`, `DeviceProtection`, `TechSupport`, `StreamingTV`, `StreamingMovies`), di luar nilai `No`/`No internet service`/`No phone service` | Proxy keterikatan customer terhadap ekosistem layanan |
| `Family_Status` | string (kategori) | Gabungan `Partner` + `Dependents`: `Single`, `Partner Only`, `Dependents Only`, `Partner+Dependents` | Konteks rumah tangga |
| `Contract_Risk_Group` | string (kategori) | Relabel `Contract`: `Month-to-month`→`Flexible`, `One year`→`Mid-Term`, `Two year`→`Long-Term` | Label netral, divalidasi lewat EDA sebelum diberi makna risiko |
| `Payment_Risk_Group` | string (kategori) | Relabel `PaymentMethod`: check-based→`Manual`, automatic→`Automatic` | Label netral, divalidasi lewat EDA |

⚠️ **Catatan redundansi:** `Contract_Risk_Group` dan `Payment_Risk_Group` adalah relabel langsung dari `Contract` dan `PaymentMethod` — membawa informasi identik, terbukti lewat koefisien model yang sama persis (lihat `methodology_notes.md` Bagian 3 & 11).

---

## 3. Kolom Hasil Model & Analisis (di file `data_processed/`)

### 3.1 Probability & Scoring

| Kolom | File Sumber | Tipe | Deskripsi |
|---|---|---|---|
| `logreg_proba` | `model_predictions_test.csv` | float (0–1) | Probabilitas churn dari Logistic Regression, **belum dikalibrasi** |
| `rf_proba` | `model_predictions_test.csv` | float (0–1) | Probabilitas churn dari Random Forest, **belum dikalibrasi** |
| `churn_proba` | `final_churn_scores.csv` dan seterusnya | float (0–1) | Probabilitas churn **setelah kalibrasi** (`CalibratedClassifierCV`, sigmoid). **Ini yang dipakai di semua analisis lanjutan** |
| `y_true` | Berbagai file | int (0/1) | Label aktual (ground truth) — sama dengan `Churn_Flag` |

### 3.2 Value & Risk Segmentation

| Kolom | File Sumber | Tipe | Deskripsi |
|---|---|---|---|
| `Value_Group` | `customer_value_scored.csv` dan seterusnya | string (kategori) | `Low Value`, `Medium Value`, `High Value` — berdasarkan kuantil `MonthlyCharges` |
| `Risk_Group` | `risk_value_segmented.csv` dan seterusnya | string (kategori) | `Low Risk` (<0.3), `Medium Risk` (0.3–0.6), `High Risk` (>0.6) — cutoff dari threshold tuning |
| `Business_Segment` | `risk_value_segmented.csv` dan seterusnya | string (kategori) | Hasil silang Value × Risk: `Critical`, `Protect`, `Maintain`, `Retain`, `Monitor`, `Selective`, `Low Cost`, `Normal` (definisi lengkap di README) |

### 3.3 Revenue & Priority

| Kolom | File Sumber | Tipe | Deskripsi |
|---|---|---|---|
| `Expected_Revenue_at_Risk` | `revenue_at_risk_scored.csv` dan seterusnya | float ($) | `churn_proba × MonthlyCharges` — estimasi revenue bulanan yang berisiko hilang per customer |
| `Priority_Rank` | `customer_retention_scoring.csv` | int | Ranking customer berdasarkan `Expected_Revenue_at_Risk` (1 = prioritas tertinggi) |
| `Recommended_Action` | `retention_strategy_final.csv` | string | Aksi retensi yang disarankan berdasarkan `Business_Segment` |
| `Strategy_Reason` | `retention_strategy_final.csv` | string | Alasan bisnis di balik `Recommended_Action` |
| `Driver_Context` | `retention_strategy_final.csv` | string | Catatan driver spesifik per customer (kontrak fleksibel, Fiber optic, early tenure) |

### 3.4 Lift, Gain & Trade-off

| Kolom | File Sumber | Tipe | Deskripsi |
|---|---|---|---|
| `top_%_customers` | `lift_gain_table.csv` | int | Persentil populasi (10, 20, 30, ... 100) yang ditarget, diurutkan berdasarkan `churn_proba` |
| `cumulative_gain_%` | `lift_gain_table.csv` | float (%) | Persentase churner aktual yang tertangkap di top-N% tersebut |
| `lift` | `lift_gain_table.csv` | float (×) | Rasio konsentrasi churn di top-N% dibanding random targeting |
| `Strategy` | `quantitative_tradeoff.csv` | string | Nama strategi penargetan: `Mass (100%)`, `Top 10%`, `Top 20%`, `Top 30%`, `Hybrid` |
| `Reach_%` | `quantitative_tradeoff.csv` | float (%) | Persentase populasi yang ditarget oleh strategi tersebut |
| `Revenue_at_Risk_Targeted_$` | `quantitative_tradeoff.csv` | float ($) | Total `Expected_Revenue_at_Risk` yang tercakup oleh strategi — **scenario metric, bukan revenue yang benar-benar terselamatkan** |
| `Cost_per_Targeted_Customer_$` | `quantitative_tradeoff.csv` | float ($) | Asumsi biaya retensi per customer (**ilustratif**, $15) |
| `Cost_per_$_Revenue_at_Risk_Addressed` | `quantitative_tradeoff.csv` | float | Efisiensi biaya: total biaya dibagi revenue at risk yang tercakup |

### 3.5 Driver Analysis

| Kolom | File Sumber | Tipe | Deskripsi |
|---|---|---|---|
| `feature` | `churn_driver_coefficients.csv` | string | Nama fitur setelah encoding (misal `cat__Contract_Two year`) |
| `coefficient` | `churn_driver_coefficients.csv` | float | Koefisien Logistic Regression. Positif = menaikkan risiko churn, negatif = menurunkan |
| `abs_coefficient` | `churn_driver_coefficients.csv` | float | Nilai absolut `coefficient`, dipakai untuk mengurutkan berdasarkan besaran pengaruh |

---

## 4. Populasi Data per File (Penting untuk Interpretasi)

| Ukuran Populasi | File yang Menggunakan |
|---|---|
| **7,043 customer (full)** | `telco_customer_clean.csv` |
| **1,409 customer (test set / scored sample)** | Semua file lain: `model_predictions_test.csv`, `final_churn_scores.csv`, `customer_value_scored.csv`, `risk_value_segmented.csv`, `revenue_at_risk_scored.csv`, `customer_retention_scoring.csv`, `lift_gain_table.csv`, `retention_strategy_final.csv`, `quantitative_tradeoff.csv` |

⚠️ Jangan menjumlahkan/membandingkan angka dari `telco_customer_clean.csv` (7,043) langsung dengan file test-set (1,409) tanpa menyadari perbedaan skala populasi ini.

---

## Referensi Silang

- Alasan setiap keputusan desain kolom: `methodology_notes.md`
- Hasil dan temuan dari setiap kolom: `README.md`
- Kode yang menghasilkan setiap kolom: `python-scripts-lengkap.md`
