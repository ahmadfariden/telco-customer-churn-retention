# 🚀 Telco Customer Churn — Roadmap Proyek (Versi Bertahap)

> Dokumen ini adalah versi eksekusi dari master plan yang lo tulis. Isinya sama secara substansi, tapi disusun jadi **4 fase** biar bisa dikerjakan bertahap dan tiap fase punya "definition of done" yang jelas — bukan satu tumpukan besar yang bikin stuck di tengah jalan.

**Dataset:** IBM Telco Customer Churn (7043 baris, 21 kolom)
**Target akhir:** Bukan cuma prediksi churn, tapi *retention prioritization* berbasis risk × value.

---

## Cara Pakai Dokumen Ini

- Kerjain fase secara berurutan. Jangan loncat ke Fase 3 kalau Fase 1 belum solid — semua fase berikutnya bergantung pada kualitas data & model dari fase sebelumnya.
- Tiap fase punya checklist "Selesai kalau...". Itu patokan lo buat lanjut, bukan checklist administratif.
- Bagian "Stretch" di tiap fase boleh dilewati dulu di iterasi pertama. Prioritas: punya end-to-end pipeline yang jalan dulu, baru diperdalam.

---

## FASE 0 — Aturan Main (Dikerjakan Sekali di Awal, Dipegang Terus)

Ini bukan "langkah", tapi prinsip yang harus dipegang sepanjang proyek. Taruh ini di bagian paling atas notebook pertama sebagai reminder.

| # | Aturan | Kenapa Penting |
|---|--------|-----------------|
| 1 | **Tidak ada fitur yang dihitung dari `Churn`** (target leakage) | Model jadi curang & tidak valid dipakai di data baru |
| 2 | **`customerID` bukan feature** — simpan terpisah untuk drill-through/scoring | ID tidak punya makna prediktif, tapi tetap dibutuhkan untuk laporan per customer |
| 3 | **`TotalCharges` harus dikonversi ke numeric** (`pd.to_numeric(errors="coerce")`) — ada blank value di data mentah | Kalau tidak, kolom ini kebaca sebagai teks dan tidak bisa dipakai model |
| 4 | **Jangan andalkan Accuracy** sebagai metrik utama — churn itu imbalanced | Accuracy tinggi bisa dicapai cuma dengan menebak "tidak churn" terus |
| 5 | **Jangan simpulkan "Month-to-month = buruk" sebelum data menunjukkan itu** | Biarkan angka yang bicara, bukan asumsi awal |
| 6 | **Probability harus dicek kalibrasinya** sebelum dipakai untuk prioritas retention | Kalau tidak terkalibrasi, angka 0.80 bisa menyesatkan dibanding 0.30 |

✅ **Selesai kalau:** enam poin ini lo pahami alasannya (bukan cuma hafal), dan siap dicek ulang tiap kali bikin fitur baru.

---

## FASE 1 — Fondasi: Data Bersih + Model Dasar yang Jalan

**Tujuan fase ini:** dari data mentah sampai punya satu model yang bisa memprediksi churn dengan evaluasi yang benar. Selesai fase ini = lo sudah punya "proyek yang layak dipamerkan", meskipun sederhana.

### 1.1 Data Profiling & Cleaning
- Cek jumlah baris, tipe data, missing value per kolom
- Perbaiki `TotalCharges` (string → numeric, tentukan cara handle blank: drop atau isi dengan `tenure × MonthlyCharges`)
- Cek duplikasi `customerID`

### 1.2 Feature Engineering (secukupnya, jangan berlebihan)
Fokus ke fitur yang punya **alasan bisnis**, bukan sekadar kombinasi matematis:
- `Tenure_Group` (0–6, 7–12, 13–24, 25–48, 49+ bulan)
- `Service_Count` (jumlah layanan yang dipakai)
- `Family_Status` (dari Partner + Dependents)
- `Contract_Risk_Group`, `Payment_Risk_Group`

❌ Skip dulu: `MonthlyCharges_x_tenure`, fitur kuadrat/log — belum tentu perlu, tambahkan nanti kalau feature importance menunjukkan butuh.

### 1.3 EDA Inti (bukan semua 7 layer dulu — cukup yang paling actionable)
- Churn rate keseluruhan
- Churn by Contract type
- Churn by Tenure group
- Churn by Payment method

### 1.4 Train/Test Split + Pipeline
- Stratified split (jaga proporsi churn di train & test)
- `ColumnTransformer` + `Pipeline` (OneHotEncoder untuk kategorikal, StandardScaler untuk numerik, SimpleImputer kalau perlu)

### 1.5 Modeling Dasar
- **Logistic Regression** (baseline interpretable)
- **Random Forest** (benchmark nonlinear)

### 1.6 Evaluasi yang Benar
- ROC-AUC, PR-AUC, Precision, Recall, F1, Confusion Matrix
- Bandingkan performa 2 model, pilih salah satu sebagai model utama

✅ **Selesai kalau:** ada satu model final dengan evaluasi lengkap (bukan cuma accuracy), dan lo bisa jelaskan kenapa model itu yang dipilih.

---

## FASE 2 — Dari Prediksi ke Keputusan Bisnis

**Tujuan fase ini:** mengubah angka probabilitas jadi sesuatu yang bisa dipakai untuk keputusan retention. Ini bagian yang bikin proyek lo beda dari tutorial biasa.

### 2.1 Threshold Tuning
- Coba threshold 0.30 / 0.40 / 0.50 / 0.60 / 0.70
- Lihat trade-off Precision vs Recall vs jumlah customer yang masuk "predicted churn"
- Kaitkan dengan kapasitas retention tim (misal: tim cuma sanggup hubungi 20% customer)

### 2.2 Probability Calibration
- Reliability curve, Brier Score
- `CalibratedClassifierCV` kalau probabilitas mentah model belum reliable

### 2.3 Customer Value Proxy (Jangan Arbitrary)
Untuk MVP, jangan bikin composite score dengan bobot buatan sendiri (mis. MonthlyCharges 40% + TotalCharges 40% + tenure 20%) — sulit dipertanggungjawabkan kenapa bobotnya segitu.

Gunakan **`MonthlyCharges` sebagai primary value proxy**, karena rantai logikanya jelas dan defensible:
```
Customer churn → kehilangan recurring revenue → MonthlyCharges
```
`TotalCharges` dan `tenure` tetap dipakai, tapi sebagai **contextual metrics** (misal untuk validasi konsistensi value vs lama berlangganan), bukan komponen skor utama.

Kalau mau bikin value score komposit yang lebih kaya (mendekati CLV), itu masuk **Stretch**, bukan MVP.

### 2.4 Risk × Value Matrix
Bangun matrix sederhana dulu (2x2 atau 3x3):

```
                    CHURN RISK
                Low          High
             ┌──────────┬──────────┐
High Value   │ Maintain │ CRITICAL │
             ├──────────┼──────────┤
Low Value    │ Stable   │ Monitor  │
             └──────────┴──────────┘
```

⚠️ Catatan: dengan 7043 baris, sel-sel di matrix yang lebih detail (3x3/4x4) bisa jadi kecil jumlahnya. Jangan overinterpret angka kecil sebagai insight kuat.

### 2.5 Revenue / Value at Risk
```
Expected Monthly Revenue at Risk (per customer) = Churn Probability × MonthlyCharges
Total Expected Revenue at Risk = Σ (MonthlyCharges × Churn Probability)
```
Ini yang mengubah statement dari "500 customer berisiko tinggi" jadi "≈$X revenue bulanan berisiko hilang".

### 2.6 Retention Priority & Target List
Untuk MVP, **jangan hitung value dua kali**. Kalau Customer Value Proxy sudah berbasis `MonthlyCharges` (lihat 2.3), maka priority ranking langsung pakai:
```
Priority Ranking = urutkan berdasarkan Expected Revenue at Risk (dari 2.5)
```
Ini defensible dan gampang dijelaskan: *"Customer ini diprioritaskan karena probability churn tinggi DAN recurring revenue yang berisiko hilang juga tinggi."*

Kalau nanti mau bikin skor gabungan yang lebih kompleks (Stretch), baru masuk:
```
Retention Priority Score = Normalized Churn Probability × Normalized Customer Value
```

Hasil akhir: tabel `customerID | churn_probability | monthly_charges | expected_revenue_at_risk | rank`.

### 2.7 Lift & Cumulative Gain Analysis (Dua Metrik Terpisah, Jangan Dicampur)
Ini komponen yang tadinya kurang — penting untuk menjawab pertanyaan bisnis inti soal efektivitas model. **Lift dan Cumulative Gain adalah dua hal berbeda — hitung dan tampilkan keduanya secara terpisah, jangan sampai notebook cuma menghasilkan satu chart tapi disebut mewakili dua-duanya.**

- Urutkan customer berdasarkan churn probability (descending)
- **Cumulative Gain** — dari total churner yang ada, berapa persen tertangkap kalau ambil Top 10% / 20% / 30% / 40% customer teratas?
  - Contoh (harus dihitung dari data, bukan diasumsikan): *"Top 20% customer berdasarkan churn probability berhasil menangkap 48% dari seluruh actual churner."*
- **Lift** — seberapa lebih pekat (concentrated) churner di top-N% dibanding kalau menargetkan secara random?
  - Contoh: *"Top 20% memiliki churn concentration 2.4× dibanding random targeting."*
- Ini langsung menjawab: *"Kalau tim retention cuma sanggup hubungi 20% customer, apakah model ini cukup efektif?"*

✅ **Selesai kalau:** lo punya kurva Cumulative Gain **dan** kurva/tabel Lift secara terpisah, bukan satu grafik yang diberi dua nama.

✅ **Selesai kalau:** lo punya daftar customer terurut yang bisa dijawab pertanyaannya "siapa yang paling layak diprioritaskan dan kenapa".

---

## FASE 3 — Strategi & Batasan (Lapisan yang Membuat Proyek Ini "Naik Kelas")

### 3.1 Churn Driver Analysis (Framework, Bukan Sekadar Angka)
Untuk tiap temuan penting, tulis dengan format:
> **Driver** → **Evidence** → **Business Meaning** → **Potential Action**

Contoh:
- Driver: Month-to-month contract
- Evidence: churn rate lebih tinggi dibanding kontrak jangka panjang
- Business Meaning: komitmen kontraktual rendah
- Potential Action: tawarkan migrasi ke kontrak lebih panjang

### 3.2 Retention Strategy per Segmen
| Segmen | Aksi | Alasan |
|---|---|---|
| High Value + High Risk | Personal retention offer | Revenue exposure tinggi |
| High Risk + Month-to-month | Contract migration offer | Komitmen rendah |
| High Risk + Low Tenure | Onboarding intervention | Churn terkonsentrasi di awal masa pakai |
| Low Value + High Risk | Low-cost intervention | ROI retention kemungkinan rendah |

### 3.3 Quantitative Trade-Off (Simulasi, Bukan Data Riil)
Bandingkan strategi dengan asumsi biaya retention (`$C`/customer) yang **dinyatakan eksplisit sebagai asumsi ilustratif**. Jangan berhenti di reach/cost/value — tambahkan metrik efisiensi supaya perbandingan antar strategi apple-to-apple:

| Strategi | Reach | Revenue at Risk Targeted | Estimasi Biaya | Cost per Targeted Customer | Cost per $ of Revenue at Risk Addressed |
|---|---|---|---|---|---|
| Mass (100%) | 100% | ... | ... | ... | ... |
| Top 10% | 10% | ... | ... | ... | ... |
| Top 20% | 20% | ... | ... | ... | ... |
| Top 30% | 30% | ... | ... | ... | ... |
| Hybrid (top 20% + 10% random exploration) | 30% | ... | ... | ... | ... |

⚠️ **Penamaan harus hati-hati.** "Revenue at Risk Addressed/Targeted" **bukan** berarti revenue yang benar-benar berhasil diselamatkan — dataset ini tidak punya data treatment/outcome retention. Jadi:
```
Cost per $ of Revenue at Risk Addressed = Retention Cost / Revenue at Risk Targeted
```
Tulis eksplisit di README bahwa **"addressed/targeted" adalah scenario metric** (revenue yang masuk dalam cakupan target strategi), **bukan realized/saved revenue**. Ini konsisten dengan limitasi di 3.4 — jangan sampai metrik ini kebaca seolah membuktikan revenue yang beneran terselamatkan.

Strategi "Hybrid" bukan eksperimen dengan control group formal (tidak ada randomization + outcome yang benar-benar diukur), jadi jangan disebut "kontrol". Definisi yang lebih tepat:
```
Hybrid = Top 20% model-prioritized + 10% random exploration sample
```
Istilah "treatment/control" baru dipakai kalau nanti beneran bikin A/B testing simulation (masuk Stretch).

### 3.4 Limitasi yang Wajib Ditulis di README
> Dataset ini tidak punya data historical treatment/control, sehingga tidak bisa dipakai untuk klaim kausal seperti "diskon X akan menyelamatkan 70% customer". Yang bisa dilakukan hanyalah **risk-based prioritization** dan **scenario-based simulation**, bukan **causal uplift modeling**.

✅ **Selesai kalau:** README lo secara eksplisit membedakan mana yang "temuan dari data" dan mana yang "asumsi/simulasi ilustratif".

---

## FASE 4 — Presentasi: Power BI & Dokumentasi

### 4.1 Dashboard 1 — Executive Overview
KPI: Total Customers, Churn Rate, Revenue at Risk, High-Risk Customers
Visual: churn by contract, tenure, payment, service

### 4.2 Dashboard 2 — Predictive Retention (boleh menyusul, tidak wajib di iterasi pertama)
- Distribusi churn probability
- Risk × Value matrix interaktif
- Tabel customer + priority rank + recommended action
- Perbandingan strategi (reach vs value covered vs cost)

### 4.3 Dokumentasi
- `README.md` — ringkasan proyek, cara menjalankan, limitasi
- `data_dictionary.md`
- `methodology_notes.md`
- `business_recommendations.md`

✅ **Selesai kalau:** orang lain (recruiter/reviewer) bisa paham keseluruhan alur proyek hanya dari README + 1 dashboard, tanpa harus baca semua notebook.

---

## Struktur Folder (Tetap, Tidak Berubah dari Rencana Awal)

```
telco-customer-churn-retention/
│
├── data/
│   └── telco_customer_churn.csv
├── notebooks/
│   ├── 01_data_profiling_cleaning.ipynb
│   ├── 02_feature_engineering.ipynb
│   ├── 03_eda.ipynb
│   ├── 04_churn_driver_analysis.ipynb
│   ├── 05_risk_value_segmentation.ipynb
│   ├── 06_modeling.ipynb
│   ├── 07_probability_scoring.ipynb
│   └── 08_retention_optimization.ipynb
├── data_processed/
│   ├── telco_customer_clean.csv
│   └── customer_retention_scoring.csv
├── models/
│   ├── logistic_regression.pkl
│   ├── random_forest.pkl
│   ├── calibrated_model.pkl
│   └── model_evaluation.csv
├── powerbi/
│   └── telco_churn_retention_dashboard.pbix
├── docs/
│   ├── data_dictionary.md
│   ├── methodology_notes.md
│   └── business_recommendations.md
└── README.md
```

---

## Stretch (Setelah 4 Fase di Atas Beneran Selesai)

Jangan sentuh ini sebelum Fase 1–4 solid:
- XGBoost + SHAP (untuk explainability lebih dalam)
- Customer Lifetime Value
- Survival analysis (waktu-sampai-churn)
- A/B testing simulation untuk uplift retention
- Model drift monitoring

---

## Ringkasan Prioritas

1. **Fase 1 dulu, sampai tuntas.** Ini yang menentukan apakah fondasi model lo valid.
2. **Fase 2** adalah nilai jual utama proyek — jangan diskip demi buru-buru ke dashboard.
3. **Fase 3** adalah yang membedakan "proyek ML biasa" dari "proyek yang paham batasan data & bisnis".
4. **Fase 4** adalah pembungkus — penting untuk presentasi, tapi jangan dikerjakan sebelum insight-nya sendiri sudah matang.

🏆 Overall architecture

                    TELCO DATA
                        │
                        ▼
                DATA QUALITY
                        │
                        ▼
                FEATURE ENGINEERING
                        │
                        ▼
                      EDA
                        │
                        ▼
               CHURN DIAGNOSIS
                        │
                        ▼
                MACHINE LEARNING
                        │
              ┌─────────┴─────────┐
              ▼                   ▼
        Logistic Regression   Random Forest
              └─────────┬─────────┘
                        ▼
                 MODEL EVALUATION
                        │
                        ▼
                PROBABILITY CALIBRATION
                        │
                        ▼
                 CHURN PROBABILITY
                        │
              ┌─────────┴─────────┐
              ▼                   ▼
          CHURN RISK        MONTHLY CHARGES
              │                   │
              └─────────┬─────────┘
                        ▼
                 RISK × VALUE
                        │
                        ▼
              REVENUE AT RISK
                        │
                        ▼
               PRIORITY RANKING
                        │
                 ┌──────┴──────┐
                 ▼             ▼
            Lift/Gain      Target List
                 │             │
                 └──────┬──────┘
                        ▼
             RETENTION STRATEGY
                        │
                        ▼
              COST / VALUE TRADE-OFF
                        │
                        ▼
                 BUSINESS ACTION
                        │
                        ▼
                    POWER BI