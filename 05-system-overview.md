# INNUTRIRE — System Overview Diagram

> End-to-end flow from a clinician making a prediction request to the result being displayed on the ICU bedside app. Spans all four repositories.

---

## Full System Architecture

```mermaid
flowchart TD
    subgraph CLINICIAN["👤 Clinician — ICU Bedside"]
        TAB["Windows Tablet / PC\nRunning frontend_innutrire.exe"]
    end

    subgraph FRONTEND["frontend_innutrire — PySide6 Desktop App"]
        F_LIST["Patient List Screen"]
        F_DASH["Patient Dashboard\n(name, bed, recent prediction, insights)"]
        F_WIZARD["REE Calculation Wizard\n(5-step form)"]
        F_THREAD["ApiWorker QThread\n(async HTTP calls)"]
        F_RESULT["Result Panel\nAI REE kcal/day\nEnergy Recommendation kcal/day"]
    end

    subgraph IAC["hospital-server-deployment-iac\nUbuntu Server — 192.168.8.200"]
        UFW["UFW Firewall\nALLOW: 22, 80, 443, 8000\nDENY: all other inbound"]
        subgraph DOCKER["Docker Compose — app_net bridge"]
            WEB["web container\nGunicorn 3 workers\n0.0.0.0:8000"]
            PGDB["db container\nPostgreSQL 16-alpine\nport 5432 (internal)"]
        end
        VPN["Tailscale VPN\n(optional secure remote access)"]
    end

    subgraph BACKEND["django_backend_innutrire — Django 6 REST API"]
        B_URLS["urls.py dispatcher"]
        B_PV["prediction_views.py"]
        B_GV["gold_standard_views.py"]
        B_PAT["patient_views.py"]
        subgraph B_ML["ML Inference — ml_model_service.py"]
            B_PREP["prepare_data()\n• Average 3 vitals readings\n• Encode gender, BMI class, Temp class\n→ 14-column DataFrame"]
            B_SVR["StandardScaler → SVR.predict()\n→ raw REE kcal/day"]
            B_INJ["injury_factor.py\nrec = raw_ree × max(trauma, burns)"]
        end
        B_XAI["explainable_ai_views.py\nSHAP + PFI on demand"]
        B_AGENT["agent/ — Moniteer proxy\nserver-health, patients, predictions"]
        B_DB["PostgreSQL\nPatient, Prediction, GoldStandard"]
    end

    subgraph MLREPO["Regression_models_innutrire — ML Research (offline)"]
        R_NB["87 Jupyter Notebooks\n10+ algorithms, 5-fold CV"]
        R_BEST["Best Model: SVR\nRMSE=314 kcal, R²=0.42"]
        R_SHAP["SHAP + PFI Analysis"]
        R_EXPORT["retrain_and_export_models.py\n→ model_dict.joblib"]
    end

    MLREPO -->|"Manual: copy .joblib\nto innutrire/model/"| BACKEND

    TAB --> F_LIST --> F_DASH --> F_WIZARD
    F_WIZARD -->|"All 5 forms complete"| F_THREAD
    F_THREAD -->|"POST /api/predictions\n(28 clinical fields)"| UFW
    UFW --> WEB --> B_URLS --> B_PV
    B_PV --> B_PREP --> B_SVR --> B_INJ
    B_INJ --> B_DB
    B_DB -->|"{prediction_id, value, recommendation}"| B_PV
    B_PV --> WEB --> UFW --> F_THREAD

    F_THREAD -->|"POST /api/gold-standard\n(prediction_id + IC measurements)"| UFW
    UFW --> WEB --> B_URLS --> B_GV --> B_DB

    F_THREAD -->|"finish_calculation_signal"| F_RESULT
    F_RESULT --> TAB

    WEB --> PGDB
```

---

## Prediction Request: Step-by-Step Sequence

```mermaid
sequenceDiagram
    actor C as Clinician
    participant APP as Desktop App (PySide6)
    participant W as ApiWorker (QThread)
    participant FW as UFW Firewall
    participant GUN as Gunicorn (web container)
    participant DJ as Django (prediction_views)
    participant ML as ModelService (SVR)
    participant DB as PostgreSQL (db container)

    C->>APP: Selects patient & opens REE Wizard
    C->>APP: Completes Form 0 — Gold Standard (IC measurements)
    C->>APP: Completes Form 1 — Demographics (ABW auto-calculates BMI/IBW/BSA)
    C->>APP: Completes Form 2 — Severity Scores (APACHE/SOFA/SAPS2/NUTRIC)
    C->>APP: Completes Form 3 — Vitals (Temp/RR/HR/TV ×3, MV)
    C->>APP: Completes Form 4 — Injury Details (Trauma + Burns)
    C->>APP: Clicks "Calculate"

    APP->>W: Spawn ApiWorker thread
    W->>FW: POST /api/predictions {28 fields}
    FW->>GUN: Allow (port 8000 open)
    GUN->>DJ: Route to prediction_views.py

    DJ->>ML: ModelService.predict(clinical_data)
    ML->>ML: prepare_data() — feature engineering (14 columns)
    ML->>ML: StandardScaler.transform(X)
    ML->>ML: SVR.predict(X_scaled) → raw REE
    ML->>ML: raw_ree × max(trauma_factor, burns_factor)
    ML-->>DJ: {raw_ree, energy_recommendation}

    DJ->>DB: Prediction.objects.create(patient_id, value, recommendation, ...28 fields)
    DB-->>DJ: prediction record (id assigned)
    DJ-->>GUN: JsonResponse 201 {prediction_id, value, energy_recommendation}
    GUN-->>FW: response
    FW-->>W: response

    W->>FW: POST /api/gold-standard {prediction_id, RQ, HR, VO2, VCO2, ref_ree, ...}
    FW->>GUN: Allow
    GUN->>DJ: Route to gold_standard_views.py
    DJ->>DB: GoldStandard.objects.create(prediction_id, ...)
    DB-->>DJ: saved
    DJ-->>GUN: 201
    GUN-->>FW: response
    FW-->>W: response

    W-->>APP: finished signal {prediction, recommendation}
    APP->>C: Display: AI REE = X kcal/day | Recommendation = Y kcal/day
```

---

## Repository Roles Summary

```mermaid
flowchart LR
    subgraph R1["Regression_models_innutrire"]
        RL["Research & Training\n(offline / data science)"]
    end

    subgraph R2["hospital-server-deployment-iac"]
        RL2["Infrastructure Provisioning\n(Ansible — run once per server)"]
    end

    subgraph R3["django_backend_innutrire"]
        RL3["Business Logic + ML Serving\n(always running on server)"]
    end

    subgraph R4["frontend_innutrire"]
        RL4["Clinician UI\n(desktop .exe on ICU tablet)"]
    end

    R1 -->|"Export .joblib model artifact"| R3
    R2 -->|"Provision server + deploy Docker"| R3
    R4 -->|"HTTP REST API calls"| R3
```

---

## Data Flow: From Patient Vitals to REE Prediction

```mermaid
flowchart TD
    A["Clinician enters:\n• ABW, Height, Age, Gender\n• APACHE II, SOFA, SAPS2, NUTRIC\n• Temp ×3, RR ×3, HR ×3, TV ×3\n• MV (auto-calc)\n• Trauma type + severity\n• Burns type + severity"]

    A --> B["Frontend: FilledInFormSignal\ncollects all 28 attributes"]
    B --> C["ApiWorker: POST /api/predictions"]
    C --> D["Django: json.loads(request.body)"]

    D --> E["prepare_data():\n• avg(TV1,TV2,TV3) → mean_tidal_volume\n• avg(RR1,RR2,RR3) → mean_resp_rate\n• avg(HR1,HR2,HR3) → mean_heart_rate\n• avg(Temp1,Temp2,Temp3) → mean_temp\n• gender: M→1, F→0\n• BMI → category (one-hot 4 cols)\n• Temp → category (one-hot 3 cols)\n→ 14-column DataFrame"]

    E --> F["StandardScaler.transform(X_14col)"]
    F --> G["SVR.predict(X_scaled)\n→ raw_ree (kcal/day)"]
    G --> H["energy_recommendation =\nraw_ree × max(trauma_factor, burns_factor)"]

    H --> I["Save to DB: Prediction record"]
    I --> J["Response: {prediction_id, value, recommendation}"]
    J --> K["Desktop app displays result\nto clinician at bedside"]
```

---

## Infrastructure Topology

```mermaid
flowchart TB
    subgraph LAN["Hospital LAN"]
        T1["ICU Tablet 1\n(frontend_innutrire.exe)"]
        T2["ICU Tablet 2\n(frontend_innutrire.exe)"]
    end

    subgraph SERVER["On-Premise Ubuntu Server\n192.168.8.200"]
        UFW["UFW Firewall"]
        GUN["Gunicorn :8000"]
        PG["PostgreSQL :5432\n(internal only)"]
        TS["Tailscale VPN"]
    end

    subgraph REMOTE["Remote (via Tailscale)"]
        DEV["Developer / Admin\n(SSH + monitoring)"]
    end

    subgraph CLOUD["GitHub"]
        REPO1["django_backend_innutrire"]
        REPO2["hospital-server-deployment-iac"]
    end

    T1 -->|"HTTP :8000"| UFW
    T2 -->|"HTTP :8000"| UFW
    UFW --> GUN
    GUN --> PG
    DEV -->|"Tailscale mesh"| TS
    TS --> UFW
    REPO1 -->|"git clone (Ansible)"| SERVER
    REPO2 -->|"Ansible playbook\n(run on server)"| SERVER
```
