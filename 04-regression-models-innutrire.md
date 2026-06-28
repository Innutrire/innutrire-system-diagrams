# Regression_models_innutrire — Flow Diagram

> **What it is:** The ML research repository for INNUTRIRE. It contains 87 Jupyter notebooks that systematically explore data, compare regression algorithms, and converge on the best model for predicting ICU patient Resting Energy Expenditure (REE) in kcal/day. The winning model is exported as a `.joblib` artifact and deployed into the Django backend.

---

## Overall Research Workflow

```mermaid
flowchart TD
    A(["Raw Data — Excel Files"]) --> B

    subgraph DATA["Data Sources"]
        B["(n=315) 12august2025.xlsx\nPrimary: 29 columns, 315 ICU records\nTargets: IC Cosmed (n=176), IC GE (n=275)"]
        B2["(n=324) serum_urea_albumin.xlsx\n+ lab values, head/chest temp"]
        B3["(n=38) Extended severity score"]
        B4["new_300_updated data_with PE.xlsx\n+ Pulmonary Embolism feature"]
    end

    B --> EDA
    B2 & B3 & B4 --> EDA

    subgraph EDA["Exploratory Data Analysis"]
        EDA1["eda.ipynb — ydata_profiling\n37-column statistical report"]
        EDA2["compare_cosmed_ge/comparison.ipynb\nBias between two calorimeter brands"]
        EDA3["head_chest_temp/head_chest_eda.ipynb\nTemperature measurement subgroup n=73"]
        EDA4["serum_urea_albumin/eda.ipynb\nLab values subgroup n=236"]
    end

    EDA --> PREP

    subgraph PREP["Preprocessing Pipeline"]
        PREP1["Filter outliers: 800 ≤ IC ≤ 3000 kcal"]
        PREP1 --> PREP2["BMI categories\nunderweight / normal / overweight / obesity"]
        PREP2 --> PREP3["Temperature categories\nHypothermia / Normal / Fever"]
        PREP3 --> PREP4["One-hot encode:\nGender, Race, Admission Category, Temp class"]
        PREP4 --> PREP5["Impute missing: SimpleImputer(strategy=mean)"]
    end

    PREP --> TGT

    subgraph TGT["Target Variable Strategy"]
        T1["IC Cosmed only — n=176"]
        T2["IC GE only — n=275"]
        T3["Combined: IC = Cosmed.fillna(GE) — n=315"]
        T4["Augmented: duplicate rows where both exist — n≤442"]
    end

    TGT --> FS

    subgraph FS["Feature Selection"]
        FS1["RFECV (feature_selection/rfcve_stack_model.ipynb)\n30 features → 22 optimal"]
        FS2["Manual clinical knowledge selection\n14 features for deployment"]
    end

    FS --> TRAIN
```

---

## Model Training & Evaluation

```mermaid
flowchart TD
    TRAIN(["Features ready"]) --> CV["5-Fold KFold\n(shuffle=True, random_state=42)"]

    CV --> MODELS

    subgraph MODELS["Algorithms Explored (87 notebooks total)"]
        M1["Linear Regression\n(8 variants: Cosmed, GE, combined, SOFA, SAPS2, NUTRIC, NUTRIC, original)"]
        M2["Ridge / Lasso / Elastic Net\n(regularised linear)"]
        M3["Random Forest Regressor\n(GridSearchCV + 5-fold)"]
        M4["Gradient Boosting Regressor\n(GridSearchCV, quantile variant, LR-correction)"]
        M5["Support Vector Regressor ★\n(StandardScaler + SVR — DEPLOYED)"]
        M6["KNN Regressor\n(k=5/7)"]
        M7["XGBoost"]
        M8["CatBoost\n(tuned in best_model_catboost.ipynb)"]
        M9["AutoGluon TabularPredictor\n(weighted ensemble of 10 base models)"]
        M10["Stacking: RFR + GBR + Ridge → LinearRegression"]
        M11["KMeans(k=3) + per-cluster LR"]
        M12["Deep Learning: SVM + RFR + Dense NN"]
    end

    MODELS --> EVAL

    subgraph EVAL["Evaluation Metrics"]
        E1["RMSE — primary metric"]
        E2["R² — explained variance"]
        E3["MAPE, MAE"]
        E4["±10% and ±20% accuracy bands"]
        E5["Benchmark against 9 published Predictive Equations\n(Harris-Benedict 1919, Penn State,\nMifflin-St Jeor, Ireton-Jones, TAH, etc.)"]
    end

    EVAL --> RESULTS

    subgraph RESULTS["Key Results (model_comparison.ipynb, IC GE test set)"]
        R1["SVR         — RMSE=314 kcal  R²=0.42 ★ BEST ML"]
        R2["Random Forest — RMSE=317 kcal R²=0.41"]
        R3["H-B 1919 PE — RMSE=327 kcal  R²=0.37  (best PE)"]
        R4["Linear Reg  — RMSE=348 kcal  R²=0.29"]
        R5["XGBoost     — RMSE=354 kcal  R²=0.26"]
        R6["TAH Eq PE   — RMSE=1099 kcal (worst)"]
    end
```

---

## Data Augmentation Experiments

```mermaid
flowchart TD
    AUG(["Augmentation strategies"])

    AUG --> A1["Duplication (augmentation/gbr_aug.ipynb)\nWhen both IC Cosmed & IC GE exist\n→ create 2 rows (one per target)\nn: 315 → up to 442"]

    AUG --> A2["CTGAN Synthetic Data\n(augmentation/gbr_ctgan.ipynb)\nConditional GAN generates synthetic patients\n→ compare GBR performance on real vs synthetic"]

    AUG --> A3["Group-based Splits\n(augmentation/gbr_group_split.ipynb)\nSplit by patient group to avoid data leakage"]
```

---

## Interpretability Analysis

```mermaid
flowchart TD
    BEST["Best Model: SVR (StandardScaler + SVR)"] --> XAI

    subgraph XAI["Explainability (14-apr_shap_analysis/)"]
        X1["SHAP — KernelExplainer\n50-sample background subset\nInput: patient feature vector\nOutput: per-feature SHAP value"]
        X2["Permutation Feature Importance\nShuffle each feature → measure RMSE increase\nOutput: ranked feature list"]
        X3["DiCE Counterfactuals\nWhat-if analysis\n'Change X by Y to get Z kcal'"]
    end

    XAI --> TOPF["Top Features by RMSE-increase (PFI)\n1. ABW (Actual Body Weight) ~80 kcal\n2. Age\n3. MV (Minute Ventilation)\n4. Gender"]
```

---

## Model Export for Deployment

```mermaid
flowchart TD
    EXP(["retrain_and_export_models.py\n(src/14-apr_shap_analysis/)"])

    EXP --> E1["Load full dataset\n(n=315) 12august2025.xlsx"]
    E1 --> E2["Apply same preprocessing pipeline\n(BMI class, Temp class, one-hot, impute)"]
    E2 --> E3["Train SVR on full dataset\n(no train/test split — max data)"]
    E3 --> E4["Wrap in sklearn Pipeline:\nstep 1: StandardScaler\nstep 2: SVR"]
    E4 --> E5["Build export dict:\n{pipeline: Pipeline, feature_columns: [...14 cols...]}"]
    E5 --> E6["joblib.dump(model_dict, 'model.joblib')"]
    E6 --> E7["exported_models/model.joblib"]

    E7 -->|"Manual copy"| DEPLOY["django_backend_innutrire/\ninnutrire/model/model.joblib"]

    DEPLOY --> SERVE["Django: ModelService loads on\nfirst prediction request"]

    WARNING["⚠️ Critical constraint:\nfeature_columns order and preprocessing steps\nMUST match ml_model_service.py:prepare_data()\nMismatch → silent prediction errors in production"]
    E5 -.-> WARNING
    DEPLOY -.-> WARNING
```

---

## Repository Structure at a Glance

```mermaid
mindmap
  root((Regression_models))
    Data
      315 records primary
      324 + lab values
      38 ICU severity subset
    EDA
      eda.ipynb
      compare_cosmed_ge
      head_chest_temp
      serum_urea_albumin
    Preprocessing
      BMI classification
      Temp classification
      One-hot encoding
      RFECV feature selection
    Models
      Linear Regression ×8
      Ridge / Lasso / ElasticNet
      Random Forest ×8
      GBR ×8
      SVR ×8 DEPLOYED
      KNN ×4
      XGBoost
      CatBoost
      AutoGluon
      Stacking ensembles
    Augmentation
      Duplication
      CTGAN synthetic
      Group splits
    Interpretability
      SHAP KernelExplainer
      Permutation Importance
      DiCE Counterfactuals
    Export
      retrain_and_export_models.py
      exported_models/model.joblib
```
