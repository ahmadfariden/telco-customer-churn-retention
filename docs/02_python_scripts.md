# Telco Customer Churn — Kumpulan Script Python (Urutan Eksekusi Lengkap)

Semua script dijalankan dari root folder project:

```cmd
cd "C:\Users\ahmad farid\Downloads\04_Telco_Customer_Churn"
```

Jalankan berurutan sesuai nomor, karena tiap script bergantung pada output script sebelumnya (file di folder `data_processed/`).

---

## 1. `cleaning.py` — Data Profiling & Cleaning (Fase 1.1)

```cmd
python cleaning.py
```

```python
import pandas as pd
import numpy as np
import os

df = pd.read_csv("WA_Fn-UseC_-Telco-Customer-Churn.csv")

# Cek dulu baris mana yang bermasalah di TotalCharges
mask_problem = df["TotalCharges"].str.strip() == ""
print("Jumlah baris dengan TotalCharges kosong/spasi:", mask_problem.sum())
print("\nContoh baris bermasalah:")
print(df[mask_problem][["customerID", "tenure", "MonthlyCharges", "TotalCharges"]])

# Convert ke numeric — yang gagal convert otomatis jadi NaN
df["TotalCharges"] = pd.to_numeric(df["TotalCharges"], errors="coerce")

# Cek ulang missing value setelah convert
print("\nMissing value TotalCharges setelah convert:", df["TotalCharges"].isnull().sum())

# Isi TotalCharges = 0 untuk customer baru (tenure = 0)
df.loc[df["tenure"] == 0, "TotalCharges"] = 0

# Verifikasi
print("Missing setelah fill:", df["TotalCharges"].isnull().sum())
print(df[df["tenure"] == 0][["customerID", "tenure", "MonthlyCharges", "TotalCharges", "Churn"]])

# Cek duplikasi customerID
print("\nJumlah customerID duplikat:", df["customerID"].duplicated().sum())

# Cek isi kolom Churn
print("\nValue counts Churn:")
print(df["Churn"].value_counts())

# Simpan hasil cleaning
os.makedirs("data_processed", exist_ok=True)
df.to_csv("data_processed/telco_customer_clean.csv", index=False)
print("\nSaved to data_processed/telco_customer_clean.csv")
```

---

## 2. `feature_engineering.py` — Feature Engineering (Fase 1.2)

```cmd
python feature_engineering.py
```

```python
import pandas as pd
import numpy as np

df = pd.read_csv("data_processed/telco_customer_clean.csv")

# 1. Tenure_Group — kelompok lama berlangganan
def tenure_group(t):
    if t <= 6:
        return "0-6"
    elif t <= 12:
        return "7-12"
    elif t <= 24:
        return "13-24"
    elif t <= 48:
        return "25-48"
    else:
        return "49+"

df["Tenure_Group"] = df["tenure"].apply(tenure_group)

# 2. Service_Count — jumlah layanan aktif yang dipakai customer
service_cols = [
    "PhoneService", "MultipleLines", "InternetService",
    "OnlineSecurity", "OnlineBackup", "DeviceProtection",
    "TechSupport", "StreamingTV", "StreamingMovies"
]

def count_services(row):
    count = 0
    for col in service_cols:
        val = row[col]
        if val not in ["No", "No internet service", "No phone service"]:
            count += 1
    return count

df["Service_Count"] = df.apply(count_services, axis=1)

# 3. Family_Status — gabungan Partner + Dependents
def family_status(row):
    if row["Partner"] == "Yes" and row["Dependents"] == "Yes":
        return "Partner+Dependents"
    elif row["Partner"] == "Yes":
        return "Partner Only"
    elif row["Dependents"] == "Yes":
        return "Dependents Only"
    else:
        return "Single"

df["Family_Status"] = df.apply(family_status, axis=1)

# 4. Contract_Risk_Group — mapping netral, label divalidasi ulang di EDA
contract_map = {
    "Month-to-month": "Flexible",
    "One year": "Mid-Term",
    "Two year": "Long-Term"
}
df["Contract_Risk_Group"] = df["Contract"].map(contract_map)

# 5. Payment_Risk_Group — kelompokkan metode pembayaran
payment_map = {
    "Electronic check": "Manual",
    "Mailed check": "Manual",
    "Bank transfer (automatic)": "Automatic",
    "Credit card (automatic)": "Automatic"
}
df["Payment_Risk_Group"] = df["PaymentMethod"].map(payment_map)

# Cek hasil
print(df[["customerID", "tenure", "Tenure_Group", "Service_Count",
          "Family_Status", "Contract_Risk_Group", "Payment_Risk_Group"]].head(10))

print("\nDistribusi Tenure_Group:")
print(df["Tenure_Group"].value_counts())

print("\nDistribusi Service_Count:")
print(df["Service_Count"].value_counts().sort_index())

print("\nDistribusi Family_Status:")
print(df["Family_Status"].value_counts())

df.to_csv("data_processed/telco_customer_clean.csv", index=False)
print("\nSaved with new features.")
```

---

## 3. `eda.py` — EDA Inti (Fase 1.3)

```cmd
python eda.py
```

```python
import pandas as pd

df = pd.read_csv("data_processed/telco_customer_clean.csv")

# 1. Churn rate keseluruhan
total = len(df)
churned = (df["Churn"] == "Yes").sum()
churn_rate = churned / total * 100
print(f"Total customers: {total}")
print(f"Churned: {churned}")
print(f"Churn rate: {churn_rate:.2f}%")

# 2. Churn by Contract
print("\n=== Churn by Contract ===")
contract_summary = df.groupby("Contract").agg(
    customer_count=("customerID", "count"),
    churned=("Churn", lambda x: (x == "Yes").sum())
)
contract_summary["churn_rate_%"] = (contract_summary["churned"] / contract_summary["customer_count"] * 100).round(2)
print(contract_summary)

# 3. Churn by Tenure Group
print("\n=== Churn by Tenure Group ===")
tenure_summary = df.groupby("Tenure_Group").agg(
    customer_count=("customerID", "count"),
    churned=("Churn", lambda x: (x == "Yes").sum())
)
tenure_summary["churn_rate_%"] = (tenure_summary["churned"] / tenure_summary["customer_count"] * 100).round(2)
tenure_order = ["0-6", "7-12", "13-24", "25-48", "49+"]
tenure_summary = tenure_summary.reindex(tenure_order)
print(tenure_summary)

# 4. Churn by Payment Method
print("\n=== Churn by Payment Method ===")
payment_summary = df.groupby("PaymentMethod").agg(
    customer_count=("customerID", "count"),
    churned=("Churn", lambda x: (x == "Yes").sum())
)
payment_summary["churn_rate_%"] = (payment_summary["churned"] / payment_summary["customer_count"] * 100).round(2)
print(payment_summary.sort_values("churn_rate_%", ascending=False))

# Bonus: revenue exposure per tenure group
print("\n=== Revenue Exposure by Tenure Group ===")
revenue_summary = df.groupby("Tenure_Group").agg(
    total_monthly_revenue=("MonthlyCharges", "sum")
)
revenue_summary = revenue_summary.reindex(tenure_order)
print(revenue_summary.round(2))
```

---

## 4. `modeling_prep.py` — Train/Test Split + Pipeline (Fase 1.4)

```cmd
python modeling_prep.py
```

```python
import pandas as pd
import numpy as np
from sklearn.model_selection import train_test_split
from sklearn.compose import ColumnTransformer
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import OneHotEncoder, StandardScaler

df = pd.read_csv("data_processed/telco_customer_clean.csv")

# Encode target jadi 0/1
df["Churn_Flag"] = (df["Churn"] == "Yes").astype(int)

# Simpan customerID terpisah (bukan fitur)
customer_ids = df["customerID"]

# X dan y — drop customerID dan Churn dari fitur
X = df.drop(columns=["customerID", "Churn", "Churn_Flag"])
y = df["Churn_Flag"]

print("Shape X:", X.shape)
print("Distribusi y:\n", y.value_counts(normalize=True))

# Identifikasi kolom numerik vs kategorikal
numeric_cols = X.select_dtypes(include=["int64", "float64"]).columns.tolist()
categorical_cols = X.select_dtypes(include=["object", "str"]).columns.tolist()

print("\nKolom numerik:", numeric_cols)
print("\nKolom kategorikal:", categorical_cols)

# Stratified split — jaga proporsi churn di train & test
X_train, X_test, y_train, y_test, id_train, id_test = train_test_split(
    X, y, customer_ids,
    test_size=0.2,
    random_state=42,
    stratify=y
)

print("\nTrain shape:", X_train.shape)
print("Test shape:", X_test.shape)
print("Train churn rate:", y_train.mean().round(4))
print("Test churn rate:", y_test.mean().round(4))

# Preprocessing pipeline
preprocessor = ColumnTransformer(transformers=[
    ("num", StandardScaler(), numeric_cols),
    ("cat", OneHotEncoder(handle_unknown="ignore", drop="if_binary"), categorical_cols)
])

print("\nPipeline siap. Lanjut ke modeling.")
```

---

## 5. `modeling.py` — Logistic Regression & Random Forest (Fase 1.5–1.6)

```cmd
python modeling.py
```

```python
import pandas as pd
import numpy as np
from sklearn.model_selection import train_test_split
from sklearn.compose import ColumnTransformer
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import OneHotEncoder, StandardScaler
from sklearn.linear_model import LogisticRegression
from sklearn.ensemble import RandomForestClassifier
from sklearn.metrics import roc_auc_score, average_precision_score, classification_report, confusion_matrix

df = pd.read_csv("data_processed/telco_customer_clean.csv")
df["Churn_Flag"] = (df["Churn"] == "Yes").astype(int)
customer_ids = df["customerID"]

X = df.drop(columns=["customerID", "Churn", "Churn_Flag"])
y = df["Churn_Flag"]

numeric_cols = X.select_dtypes(include=["int64", "float64"]).columns.tolist()
categorical_cols = X.select_dtypes(include=["object", "str"]).columns.tolist()

X_train, X_test, y_train, y_test, id_train, id_test = train_test_split(
    X, y, customer_ids, test_size=0.2, random_state=42, stratify=y
)

preprocessor = ColumnTransformer(transformers=[
    ("num", StandardScaler(), numeric_cols),
    ("cat", OneHotEncoder(handle_unknown="ignore", drop="if_binary"), categorical_cols)
])

# ============ MODEL 1: Logistic Regression ============
logreg_pipeline = Pipeline([
    ("preprocessor", preprocessor),
    ("classifier", LogisticRegression(max_iter=1000, class_weight="balanced", random_state=42))
])

logreg_pipeline.fit(X_train, y_train)
logreg_proba = logreg_pipeline.predict_proba(X_test)[:, 1]
logreg_pred = logreg_pipeline.predict(X_test)

print("=" * 50)
print("LOGISTIC REGRESSION")
print("=" * 50)
print(f"ROC-AUC: {roc_auc_score(y_test, logreg_proba):.4f}")
print(f"PR-AUC:  {average_precision_score(y_test, logreg_proba):.4f}")
print("\nClassification Report (threshold=0.5):")
print(classification_report(y_test, logreg_pred))
print("Confusion Matrix:")
print(confusion_matrix(y_test, logreg_pred))

# ============ MODEL 2: Random Forest ============
rf_pipeline = Pipeline([
    ("preprocessor", preprocessor),
    ("classifier", RandomForestClassifier(
        n_estimators=300,
        max_depth=10,
        class_weight="balanced",
        random_state=42,
        n_jobs=-1
    ))
])

rf_pipeline.fit(X_train, y_train)
rf_proba = rf_pipeline.predict_proba(X_test)[:, 1]
rf_pred = rf_pipeline.predict(X_test)

print("\n" + "=" * 50)
print("RANDOM FOREST")
print("=" * 50)
print(f"ROC-AUC: {roc_auc_score(y_test, rf_proba):.4f}")
print(f"PR-AUC:  {average_precision_score(y_test, rf_proba):.4f}")
print("\nClassification Report (threshold=0.5):")
print(classification_report(y_test, rf_pred))
print("Confusion Matrix:")
print(confusion_matrix(y_test, rf_pred))

# Simpan probabilities untuk tahap berikutnya
results_df = pd.DataFrame({
    "customerID": id_test.values,
    "y_true": y_test.values,
    "logreg_proba": logreg_proba,
    "rf_proba": rf_proba
})
results_df.to_csv("data_processed/model_predictions_test.csv", index=False)
print("\nSaved test predictions to data_processed/model_predictions_test.csv")
```

---

## 6. `threshold_tuning.py` — Threshold Tuning (Fase 2.1)

```cmd
python threshold_tuning.py
```

```python
import pandas as pd
import numpy as np
from sklearn.metrics import precision_score, recall_score, f1_score, confusion_matrix

df = pd.read_csv("data_processed/model_predictions_test.csv")

y_true = df["y_true"]
y_proba = df["logreg_proba"]

thresholds = [0.30, 0.40, 0.50, 0.60, 0.70]

results = []
for t in thresholds:
    y_pred = (y_proba >= t).astype(int)

    precision = precision_score(y_true, y_pred)
    recall = recall_score(y_true, y_pred)
    f1 = f1_score(y_true, y_pred)

    predicted_churn_count = y_pred.sum()
    predicted_churn_pct = predicted_churn_count / len(y_pred) * 100

    tn, fp, fn, tp = confusion_matrix(y_true, y_pred).ravel()

    results.append({
        "threshold": t,
        "precision": round(precision, 3),
        "recall": round(recall, 3),
        "f1": round(f1, 3),
        "predicted_churn_count": predicted_churn_count,
        "predicted_churn_%": round(predicted_churn_pct, 1),
        "true_positive": tp,
        "false_positive": fp,
        "false_negative": fn,
        "true_negative": tn
    })

results_df = pd.DataFrame(results)
print(results_df.to_string(index=False))

results_df.to_csv("data_processed/threshold_tuning_results.csv", index=False)
print("\nSaved to data_processed/threshold_tuning_results.csv")
```

> **Threshold dipilih: 0.6** (F1 tertinggi, ~34% customer di-flag).

---

## 7. `calibration.py` — Probability Calibration (Fase 2.2)

```cmd
python calibration.py
```

```python
import pandas as pd
import numpy as np
from sklearn.model_selection import train_test_split
from sklearn.compose import ColumnTransformer
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import OneHotEncoder, StandardScaler
from sklearn.linear_model import LogisticRegression
from sklearn.calibration import CalibratedClassifierCV, calibration_curve
from sklearn.metrics import brier_score_loss
import matplotlib.pyplot as plt

df = pd.read_csv("data_processed/telco_customer_clean.csv")
df["Churn_Flag"] = (df["Churn"] == "Yes").astype(int)
customer_ids = df["customerID"]

X = df.drop(columns=["customerID", "Churn", "Churn_Flag"])
y = df["Churn_Flag"]

numeric_cols = X.select_dtypes(include=["int64", "float64"]).columns.tolist()
categorical_cols = X.select_dtypes(include=["object", "str"]).columns.tolist()

X_train, X_test, y_train, y_test, id_train, id_test = train_test_split(
    X, y, customer_ids, test_size=0.2, random_state=42, stratify=y
)

preprocessor = ColumnTransformer(transformers=[
    ("num", StandardScaler(), numeric_cols),
    ("cat", OneHotEncoder(handle_unknown="ignore", drop="if_binary"), categorical_cols)
])

# Model uncalibrated
base_pipeline = Pipeline([
    ("preprocessor", preprocessor),
    ("classifier", LogisticRegression(max_iter=1000, class_weight="balanced", random_state=42))
])
base_pipeline.fit(X_train, y_train)
proba_uncalibrated = base_pipeline.predict_proba(X_test)[:, 1]

# Model calibrated
calibrated_pipeline = CalibratedClassifierCV(base_pipeline, method="sigmoid", cv=5)
calibrated_pipeline.fit(X_train, y_train)
proba_calibrated = calibrated_pipeline.predict_proba(X_test)[:, 1]

# Bandingkan Brier Score
brier_uncalibrated = brier_score_loss(y_test, proba_uncalibrated)
brier_calibrated = brier_score_loss(y_test, proba_calibrated)

print(f"Brier Score - Uncalibrated: {brier_uncalibrated:.4f}")
print(f"Brier Score - Calibrated:   {brier_calibrated:.4f}")

# Reliability curve data
frac_pos_uncal, mean_pred_uncal = calibration_curve(y_test, proba_uncalibrated, n_bins=10)
frac_pos_cal, mean_pred_cal = calibration_curve(y_test, proba_calibrated, n_bins=10)

print("\n=== Reliability Table (Uncalibrated) ===")
for mp, fp in zip(mean_pred_uncal, frac_pos_uncal):
    print(f"Predicted: {mp:.3f}  |  Actual: {fp:.3f}")

print("\n=== Reliability Table (Calibrated) ===")
for mp, fp in zip(mean_pred_cal, frac_pos_cal):
    print(f"Predicted: {mp:.3f}  |  Actual: {fp:.3f}")

# Plot
plt.figure(figsize=(7, 7))
plt.plot([0, 1], [0, 1], linestyle="--", color="gray", label="Perfect calibration")
plt.plot(mean_pred_uncal, frac_pos_uncal, marker="o", label="Uncalibrated")
plt.plot(mean_pred_cal, frac_pos_cal, marker="s", label="Calibrated")
plt.xlabel("Mean Predicted Probability")
plt.ylabel("Fraction of Positives (Actual)")
plt.title("Calibration Curve - Logistic Regression")
plt.legend()
plt.savefig("data_processed/calibration_curve.png", dpi=120, bbox_inches="tight")
print("\nCalibration curve saved to data_processed/calibration_curve.png")

# Simpan probability calibrated final untuk tahap berikutnya
final_results = pd.DataFrame({
    "customerID": id_test.values,
    "y_true": y_test.values,
    "churn_proba": proba_calibrated
})
final_results.to_csv("data_processed/final_churn_scores.csv", index=False)
print("Saved final calibrated scores to data_processed/final_churn_scores.csv")
```

---

## 8. `customer_value.py` — Customer Value Proxy (Fase 2.3)

```cmd
python customer_value.py
```

```python
import pandas as pd
import numpy as np

df_full = pd.read_csv("data_processed/telco_customer_clean.csv")
scores = pd.read_csv("data_processed/final_churn_scores.csv")

# Gabungkan skor churn dengan data customer
merged = scores.merge(
    df_full[["customerID", "MonthlyCharges", "TotalCharges", "tenure", "Contract"]],
    on="customerID",
    how="left"
)

# Customer Value Proxy — primary: MonthlyCharges (dibagi 3 kuantil seimbang)
merged["Value_Group"] = pd.qcut(
    merged["MonthlyCharges"],
    q=3,
    labels=["Low Value", "Medium Value", "High Value"]
)

print("=== Distribusi Value Group ===")
print(merged["Value_Group"].value_counts())

print("\n=== Rentang MonthlyCharges per Value Group ===")
print(merged.groupby("Value_Group", observed=True)["MonthlyCharges"].agg(["min", "max", "mean"]).round(2))

print("\n=== Context Check: TotalCharges & Tenure per Value Group ===")
print(merged.groupby("Value_Group", observed=True)[["TotalCharges", "tenure"]].mean().round(2))

print("\n=== Churn Rate Aktual per Value Group ===")
value_churn = merged.groupby("Value_Group", observed=True).agg(
    customer_count=("customerID", "count"),
    churned=("y_true", "sum")
)
value_churn["churn_rate_%"] = (value_churn["churned"] / value_churn["customer_count"] * 100).round(2)
print(value_churn)

merged.to_csv("data_processed/customer_value_scored.csv", index=False)
print("\nSaved to data_processed/customer_value_scored.csv")
```

---

## 9. `risk_value_matrix.py` — Risk × Value Matrix (Fase 2.4)

```cmd
python risk_value_matrix.py
```

```python
import pandas as pd
import numpy as np

df = pd.read_csv("data_processed/customer_value_scored.csv")

# Risk Group — cutoff berdasarkan threshold tuning sebelumnya
def risk_group(p):
    if p < 0.3:
        return "Low Risk"
    elif p < 0.6:
        return "Medium Risk"
    else:
        return "High Risk"

df["Risk_Group"] = df["churn_proba"].apply(risk_group)

print("=== Distribusi Risk Group ===")
print(df["Risk_Group"].value_counts())

# Risk x Value Matrix
matrix = pd.crosstab(df["Value_Group"], df["Risk_Group"], margins=True)
risk_order = ["Low Risk", "Medium Risk", "High Risk", "All"]
matrix = matrix[[c for c in risk_order if c in matrix.columns]]
print("\n=== Risk x Value Matrix (Customer Count) ===")
print(matrix)

# Mapping ke Segment Bisnis
def segment_mapping(row):
    value = row["Value_Group"]
    risk = row["Risk_Group"]

    if value == "High Value" and risk == "High Risk":
        return "Critical"
    elif value == "High Value" and risk == "Medium Risk":
        return "Protect"
    elif value == "High Value" and risk == "Low Risk":
        return "Maintain"
    elif value == "Medium Value" and risk == "High Risk":
        return "Retain"
    elif value == "Medium Value":
        return "Monitor"
    elif value == "Low Value" and risk == "High Risk":
        return "Selective"
    elif value == "Low Value" and risk == "Medium Risk":
        return "Low Cost"
    else:
        return "Normal"

df["Business_Segment"] = df.apply(segment_mapping, axis=1)

print("\n=== Distribusi Business Segment ===")
segment_summary = df.groupby("Business_Segment").agg(
    customer_count=("customerID", "count"),
    avg_churn_proba=("churn_proba", "mean"),
    avg_monthly_charges=("MonthlyCharges", "mean"),
    actual_churn_rate=("y_true", "mean")
).round(3)
segment_summary["actual_churn_rate_%"] = (segment_summary["actual_churn_rate"] * 100).round(1)
print(segment_summary.sort_values("customer_count", ascending=False))

df.to_csv("data_processed/risk_value_segmented.csv", index=False)
print("\nSaved to data_processed/risk_value_segmented.csv")
```

---

## 10. `revenue_at_risk.py` — Revenue at Risk (Fase 2.5)

```cmd
python revenue_at_risk.py
```

```python
import pandas as pd
import numpy as np

df = pd.read_csv("data_processed/risk_value_segmented.csv")

# Expected Monthly Revenue at Risk (per customer)
df["Expected_Revenue_at_Risk"] = df["churn_proba"] * df["MonthlyCharges"]

print("=== Total Expected Revenue at Risk (Seluruh Test Set) ===")
total_revenue_at_risk = df["Expected_Revenue_at_Risk"].sum()
total_monthly_revenue = df["MonthlyCharges"].sum()
pct_at_risk = total_revenue_at_risk / total_monthly_revenue * 100

print(f"Total Monthly Revenue (semua customer di test set): ${total_monthly_revenue:,.2f}")
print(f"Total Expected Revenue at Risk: ${total_revenue_at_risk:,.2f}")
print(f"Persentase revenue yang secara ekspektasi berisiko: {pct_at_risk:.2f}%")

# Revenue at Risk per Business Segment
print("\n=== Expected Revenue at Risk per Business Segment ===")
segment_revenue = df.groupby("Business_Segment").agg(
    customer_count=("customerID", "count"),
    total_monthly_revenue=("MonthlyCharges", "sum"),
    total_expected_revenue_at_risk=("Expected_Revenue_at_Risk", "sum")
).round(2)
segment_revenue["%_of_total_risk"] = (
    segment_revenue["total_expected_revenue_at_risk"] / total_revenue_at_risk * 100
).round(1)
segment_revenue = segment_revenue.sort_values("total_expected_revenue_at_risk", ascending=False)
print(segment_revenue)

# Top 10 Customer dengan Revenue at Risk Tertinggi
print("\n=== Top 10 Customer - Expected Revenue at Risk Tertinggi ===")
top10 = df.nlargest(10, "Expected_Revenue_at_Risk")[
    ["customerID", "MonthlyCharges", "churn_proba", "Business_Segment", "Expected_Revenue_at_Risk"]
]
print(top10.to_string(index=False))

df.to_csv("data_processed/revenue_at_risk_scored.csv", index=False)
print("\nSaved to data_processed/revenue_at_risk_scored.csv")
```

---

## 11. `retention_priority.py` — Retention Priority & Ranking (Fase 2.6)

```cmd
python retention_priority.py
```

```python
import pandas as pd

df = pd.read_csv("data_processed/revenue_at_risk_scored.csv")

# Retention Priority Ranking — langsung berdasarkan Expected Revenue at Risk
df["Priority_Rank"] = df["Expected_Revenue_at_Risk"].rank(ascending=False, method="first").astype(int)

df_sorted = df.sort_values("Priority_Rank")

retention_list = df_sorted[[
    "Priority_Rank",
    "customerID",
    "churn_proba",
    "MonthlyCharges",
    "Value_Group",
    "Risk_Group",
    "Business_Segment",
    "Expected_Revenue_at_Risk"
]]

print("=== Top 20 Retention Priority List ===")
print(retention_list.head(20).to_string(index=False))

# Berapa banyak customer diperlukan untuk cover X% dari total revenue at risk
df_sorted["Cumulative_Revenue_at_Risk"] = df_sorted["Expected_Revenue_at_Risk"].cumsum()
total_risk = df_sorted["Expected_Revenue_at_Risk"].sum()
df_sorted["Cumulative_%"] = (df_sorted["Cumulative_Revenue_at_Risk"] / total_risk * 100).round(2)

print("\n=== Berapa Customer Dibutuhkan untuk Cover Sekian % Revenue at Risk ===")
for pct in [20, 40, 50, 60, 80]:
    row = df_sorted[df_sorted["Cumulative_%"] >= pct].iloc[0]
    n_customers = df_sorted[df_sorted["Cumulative_%"] <= row["Cumulative_%"]].shape[0]
    pct_of_total_customers = n_customers / len(df_sorted) * 100
    print(f"Untuk cover ~{pct}% revenue at risk: butuh {n_customers} customer ({pct_of_total_customers:.1f}% dari total)")

retention_list.to_csv("data_processed/customer_retention_scoring.csv", index=False)
print("\nSaved to data_processed/customer_retention_scoring.csv")
```

---

## 12. `lift_gain_analysis.py` — Lift & Cumulative Gain (Fase 2.7)

```cmd
python lift_gain_analysis.py
```

```python
import pandas as pd
import numpy as np

df = pd.read_csv("data_processed/revenue_at_risk_scored.csv")

# Urutkan berdasarkan churn_proba (bukan revenue)
df_sorted = df.sort_values("churn_proba", ascending=False).reset_index(drop=True)

total_customers = len(df_sorted)
total_actual_churners = df_sorted["y_true"].sum()
baseline_churn_rate = total_actual_churners / total_customers

deciles = [10, 20, 30, 40, 50, 60, 70, 80, 90, 100]

results = []
for pct in deciles:
    n = int(np.ceil(total_customers * pct / 100))
    subset = df_sorted.iloc[:n]

    churners_captured = subset["y_true"].sum()
    cumulative_gain_pct = churners_captured / total_actual_churners * 100

    segment_churn_rate = churners_captured / n
    lift = segment_churn_rate / baseline_churn_rate

    results.append({
        "top_%_customers": pct,
        "n_customers": n,
        "churners_captured": churners_captured,
        "cumulative_gain_%": round(cumulative_gain_pct, 1),
        "segment_churn_rate": round(segment_churn_rate, 3),
        "lift": round(lift, 2)
    })

results_df = pd.DataFrame(results)
print(f"Total customers: {total_customers}")
print(f"Total actual churners: {total_actual_churners}")
print(f"Baseline churn rate (random targeting): {baseline_churn_rate:.3f}")
print()
print(results_df.to_string(index=False))

results_df.to_csv("data_processed/lift_gain_table.csv", index=False)
print("\nSaved to data_processed/lift_gain_table.csv")
```

---

## 13. `churn_driver_analysis.py` — Churn Driver Analysis (Fase 3.1)

```cmd
python churn_driver_analysis.py
```

```python
import pandas as pd
import numpy as np
from sklearn.model_selection import train_test_split
from sklearn.compose import ColumnTransformer
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import OneHotEncoder, StandardScaler
from sklearn.linear_model import LogisticRegression

df = pd.read_csv("data_processed/telco_customer_clean.csv")
df["Churn_Flag"] = (df["Churn"] == "Yes").astype(int)

X = df.drop(columns=["customerID", "Churn", "Churn_Flag"])
y = df["Churn_Flag"]

numeric_cols = X.select_dtypes(include=["int64", "float64"]).columns.tolist()
categorical_cols = X.select_dtypes(include=["object", "str"]).columns.tolist()

X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42, stratify=y
)

preprocessor = ColumnTransformer(transformers=[
    ("num", StandardScaler(), numeric_cols),
    ("cat", OneHotEncoder(handle_unknown="ignore", drop="if_binary"), categorical_cols)
])

pipeline = Pipeline([
    ("preprocessor", preprocessor),
    ("classifier", LogisticRegression(max_iter=1000, class_weight="balanced", random_state=42))
])
pipeline.fit(X_train, y_train)

# Ambil nama fitur setelah encoding
feature_names = pipeline.named_steps["preprocessor"].get_feature_names_out()
coefficients = pipeline.named_steps["classifier"].coef_[0]

coef_df = pd.DataFrame({
    "feature": feature_names,
    "coefficient": coefficients
})

coef_df["abs_coefficient"] = coef_df["coefficient"].abs()
coef_df = coef_df.sort_values("abs_coefficient", ascending=False)

print("=== Top 15 Fitur Paling Berpengaruh terhadap Churn ===")
print("(Koefisien positif = meningkatkan risiko churn, negatif = menurunkan risiko churn)\n")
print(coef_df.head(15)[["feature", "coefficient"]].to_string(index=False))

coef_df.to_csv("data_processed/churn_driver_coefficients.csv", index=False)
print("\nSaved to data_processed/churn_driver_coefficients.csv")
```

---

## 14. `retention_strategy.py` — Retention Strategy per Segmen (Fase 3.2)

```cmd
python retention_strategy.py
```

```python
import pandas as pd

df = pd.read_csv("data_processed/revenue_at_risk_scored.csv")

# Retention Strategy Mapping per Segmen
strategy_map = {
    "Critical":  {"action": "Personal retention offer (diskon/loyalty program langsung)",
                  "reason": "High value + high risk, revenue exposure tertinggi per customer"},
    "Retain":    {"action": "Contract migration offer + service review",
                  "reason": "Medium value tapi churn rate aktual tertinggi (70%), sering month-to-month"},
    "Protect":   {"action": "Proactive check-in / loyalty perk preventif",
                  "reason": "High value, risiko belum kritis tapi revenue-at-risk terbesar secara agregat"},
    "Monitor":   {"action": "Automated engagement campaign (email/app notif)",
                  "reason": "Populasi besar, risiko sedang — tidak butuh personal touch mahal"},
    "Maintain":  {"action": "Pertahankan pengalaman, tanpa retention spending agresif",
                  "reason": "High value + low risk, sudah stabil"},
    "Low Cost":  {"action": "Low-cost automated intervention (email diskon kecil)",
                  "reason": "Low value + medium risk, ROI retensi mahal kemungkinan rendah"},
    "Selective": {"action": "Evaluasi kasus per kasus (populasi sangat kecil)",
                  "reason": "Low value + high risk, hati-hati overinterpret karena n kecil"},
    "Normal":    {"action": "Standard service, tanpa intervensi khusus",
                  "reason": "Low value + low risk"}
}

df["Recommended_Action"] = df["Business_Segment"].map(lambda s: strategy_map[s]["action"])
df["Strategy_Reason"] = df["Business_Segment"].map(lambda s: strategy_map[s]["reason"])

def add_driver_context(row):
    notes = []
    if row["Contract"] == "Month-to-month":
        notes.append("Kontrak fleksibel — pertimbangkan migration offer")
    if row.get("InternetService") == "Fiber optic":
        notes.append("Pengguna Fiber optic — investigasi kepuasan layanan")
    if row["tenure"] <= 6:
        notes.append("Early tenure — kandidat program onboarding")
    return "; ".join(notes) if notes else "-"

df_full = pd.read_csv("data_processed/telco_customer_clean.csv")
df = df.merge(df_full[["customerID", "InternetService"]], on="customerID", how="left")
df["Driver_Context"] = df.apply(add_driver_context, axis=1)

print("=== Contoh Retention Strategy per Customer (Top 10 Priority) ===")
sample = df.sort_values("Expected_Revenue_at_Risk", ascending=False).head(10)
print(sample[["customerID", "Business_Segment", "Recommended_Action", "Driver_Context"]].to_string(index=False))

print("\n=== Ringkasan Strategi per Segmen ===")
summary = df.groupby("Business_Segment").agg(
    customer_count=("customerID", "count"),
    total_revenue_at_risk=("Expected_Revenue_at_Risk", "sum")
).round(2)
summary["Recommended_Action"] = summary.index.map(lambda s: strategy_map[s]["action"])
print(summary.sort_values("total_revenue_at_risk", ascending=False))

df.to_csv("data_processed/retention_strategy_final.csv", index=False)
print("\nSaved to data_processed/retention_strategy_final.csv")
```

---

## 15. `quantitative_tradeoff.py` — Quantitative Trade-Off (Fase 3.3)

```cmd
python quantitative_tradeoff.py
```

```python
import pandas as pd
import numpy as np

df = pd.read_csv("data_processed/revenue_at_risk_scored.csv")

# ASUMSI ILUSTRATIF (bukan data riil, wajib disebut eksplisit di README)
RETENTION_COST_PER_CUSTOMER = 15  # dalam $, asumsi

df_sorted = df.sort_values("Expected_Revenue_at_Risk", ascending=False).reset_index(drop=True)

total_customers = len(df_sorted)
total_revenue_at_risk = df_sorted["Expected_Revenue_at_Risk"].sum()

def evaluate_strategy(name, subset_df):
    n = len(subset_df)
    reach_pct = n / total_customers * 100
    revenue_targeted = subset_df["Expected_Revenue_at_Risk"].sum()
    revenue_targeted_pct = revenue_targeted / total_revenue_at_risk * 100
    total_cost = n * RETENTION_COST_PER_CUSTOMER
    cost_per_customer = RETENTION_COST_PER_CUSTOMER
    cost_per_dollar_addressed = total_cost / revenue_targeted if revenue_targeted > 0 else np.nan

    return {
        "Strategy": name,
        "Reach_%": round(reach_pct, 1),
        "N_Customers": n,
        "Revenue_at_Risk_Targeted_$": round(revenue_targeted, 2),
        "%_of_Total_Revenue_at_Risk": round(revenue_targeted_pct, 1),
        "Total_Cost_$": round(total_cost, 2),
        "Cost_per_Targeted_Customer_$": round(cost_per_customer, 2),
        "Cost_per_$_Revenue_at_Risk_Addressed": round(cost_per_dollar_addressed, 3)
    }

results = []

# Strategy: Mass (100%)
results.append(evaluate_strategy("Mass (100%)", df_sorted))

# Strategy: Top 10%, 20%, 30%
for pct in [10, 20, 30]:
    n = int(np.ceil(total_customers * pct / 100))
    subset = df_sorted.iloc[:n]
    results.append(evaluate_strategy(f"Top {pct}%", subset))

# Strategy: Hybrid = Top 20% model-prioritized + 10% random exploration sample
n_top20 = int(np.ceil(total_customers * 20 / 100))
top20 = df_sorted.iloc[:n_top20]
remaining = df_sorted.iloc[n_top20:]
n_exploration = int(np.ceil(total_customers * 10 / 100))
np.random.seed(42)
exploration_sample = remaining.sample(n=n_exploration, random_state=42)
hybrid = pd.concat([top20, exploration_sample])
results.append(evaluate_strategy("Hybrid (Top 20% + 10% random exploration)", hybrid))

results_df = pd.DataFrame(results)
print(f"Asumsi Retention Cost per Customer: ${RETENTION_COST_PER_CUSTOMER} (ILUSTRATIF, bukan data riil)\n")
print(results_df.to_string(index=False))

results_df.to_csv("data_processed/quantitative_tradeoff.csv", index=False)
print("\nSaved to data_processed/quantitative_tradeoff.csv")
```

---

## Ringkasan Output Akhir

Setelah semua script dijalankan berurutan, folder `data_processed/` akan berisi:

```
data_processed/
├── telco_customer_clean.csv           # hasil cleaning + feature engineering
├── model_predictions_test.csv         # probabilitas prediksi test set (2 model)
├── threshold_tuning_results.csv       # hasil eksperimen threshold
├── final_churn_scores.csv             # probabilitas setelah kalibrasi
├── customer_value_scored.csv          # skor value per customer
├── risk_value_segmented.csv           # hasil Risk x Value Matrix
├── revenue_at_risk_scored.csv         # Expected Revenue at Risk per customer
├── customer_retention_scoring.csv     # retention priority ranking final
├── lift_gain_table.csv                # tabel Lift & Cumulative Gain
├── churn_driver_coefficients.csv      # koefisien Logistic Regression
├── retention_strategy_final.csv       # strategi retensi per customer/segmen
├── quantitative_tradeoff.csv          # perbandingan strategi penargetan
└── calibration_curve.png              # visualisasi reliability curve
```

**Fase yang sudah selesai:** Fase 0 (Aturan Main), Fase 1 (Fondasi), Fase 2 (Keputusan Bisnis), Fase 3 (Strategi & Batasan).
**Belum dikerjakan:** Fase 4 (Power BI Dashboard + Dokumentasi tambahan).
