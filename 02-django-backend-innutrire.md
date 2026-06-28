# django_backend_innutrire — Flow Diagram

> **What it is:** A Django 6 / Django REST Framework backend exposing a JSON REST API. It handles patient records, runs ML inference for REE prediction via a trained SVR pipeline, provides explainable AI outputs (SHAP + PFI), stores gold-standard calorimetry measurements, and proxies monitoring data for the Moniteer agent.

---

## Request Routing Overview

```mermaid
flowchart TD
    REQ([HTTP Request]) --> URLRouter["urls.py\n(Django URL dispatcher)"]

    URLRouter -->|GET /api/| HV[health_views.py]
    URLRouter -->|/api/patients| PV[patient_views.py]
    URLRouter -->|/api/predictions| PDV[prediction_views.py]
    URLRouter -->|/api/gold-standard| GV[gold_standard_views.py]
    URLRouter -->|/api/explainable-ai| EV[explainable_ai_views.py]
    URLRouter -->|/api/agent/| AV[agent/api_views/]

    HV --> R1[Return 'OK']
```

---

## Patient Endpoint Flow

```mermaid
flowchart TD
    PV([patient_views.py]) --> PM{HTTP Method + Path}

    PM -->|"GET /api/patients"| L1["Filter: deleted_at IS NULL\n→ all active Patient rows"]
    PM -->|"POST /api/patients"| L2["json.loads(request.body)\nPatient.objects.create(...)"]
    PM -->|"GET /api/patients/{id}"| L3["Patient.objects.get(id=id)"]
    PM -->|"PUT /api/patients/{id}"| L4["Partial update fields\nPatient.save()"]
    PM -->|"DELETE /api/patients/{id}"| L5["Soft-delete:\npatient.deleted_at = now()\npatient.save()"]
    PM -->|"POST /api/patients/{id}/consent"| L6["patient.is_patient_consent = True/False\npatient.save()"]
    PM -->|"GET /api/patients/{id}/latest-prediction"| L7["Prediction.objects.filter(patient=id)\n.order_by('-created_at').first()"]
    PM -->|"GET /api/patients/{id}/insights"| L8["Compare latest vs previous prediction\nReturn kcal delta + direction"]

    L1 & L2 & L3 & L4 & L5 & L6 & L7 & L8 --> RESP([JsonResponse])
```

---

## Prediction & ML Inference Flow

```mermaid
flowchart TD
    PDV(["POST /api/predictions"]) --> PD1["json.loads(request.body)\nExtract 28 clinical fields"]

    PD1 --> MS[ml_model_service.py — ModelService]

    subgraph ML["ML Inference Pipeline"]
        MS --> MS1["prepare_data()\n• Average 3 vitals readings\n  (TV, RR, HR, Temp)\n• Map gender: M→1, F→0\n• BMI class: underweight/normal/\n  overweight/obesity\n• Temp class: Hypothermia/Normal/Fever\n→ 14-column DataFrame"]
        MS1 --> MS2["Load .joblib from innutrire/model/\n{pipeline: sklearn Pipeline,\n feature_columns: [...]}"]
        MS2 --> MS3["StandardScaler.transform(X)"]
        MS3 --> MS4["SVR.predict(X_scaled)\n→ raw REE kcal/day"]
    end

    MS4 --> IF[injury_factor.py]
    IF --> IF1["energy_recommendation =\nraw_ree × max(trauma_factor, burns_factor)"]

    IF1 --> SAVE["Prediction.objects.create(\n  patient_id, value=raw_ree,\n  energy_recommendation, ...28 fields\n)"]
    SAVE --> RESP["JsonResponse:\n{prediction_id, value, energy_recommendation}"]
```

---

## Explainable AI Flow

```mermaid
flowchart TD
    EV([explainable_ai_views.py]) --> EVM{Endpoint}

    EVM -->|"GET /api/explainable-ai/pfi"| PFI[PermutationFeatureImportanceService]
    PFI --> PFI1["Load training data from\ndata_8april.xlsx via DataService"]
    PFI1 --> PFI2["For each feature:\n  shuffle column → measure RMSE increase\n  restore column"]
    PFI2 --> PFI3["Sort features by RMSE increase\n(descending)"]
    PFI3 --> PFI4["Return: [{feature, importance_score}, ...]"]

    EVM -->|"POST /api/explainable-ai/shap"| SHAP[LocalShapAnalysisService]
    SHAP --> SHAP1["json.loads(request.body)\nExtract patient feature dict"]
    SHAP1 --> SHAP2["KernelExplainer(\n  model.predict,\n  background_50_samples\n)"]
    SHAP2 --> SHAP3["explainer.shap_values(X_patient)"]
    SHAP3 --> SHAP4["Return: {feature: shap_value, ...}"]
```

---

## Gold Standard Endpoint Flow

```mermaid
flowchart TD
    GV(["POST /api/gold-standard"]) --> GV1["json.loads(request.body)"]
    GV1 --> GV2["Resolve Prediction by prediction_id"]
    GV2 --> GV3["GoldStandard.objects.create(\n  prediction=prediction,\n  reference_ree,\n  respiratory_quotient,\n  heart_rate,\n  variability_in_oxygen_consumption,\n  variability_in_carbon_dioxide_consumption,\n  measurement_timestamp,\n  brand_type\n)"]
    GV3 --> GV4["JsonResponse 201"]
```

---

## Agent Proxy Flow (Moniteer)

```mermaid
flowchart TD
    AV(["/api/agent/* — Moniteer proxy"]) --> AVM{Endpoint}

    AVM -->|"GET /api/agent/ping"| AG1["Return: {status: pong}"]
    AVM -->|"GET /api/agent/server-health"| AG2["psutil:\n  cpu_percent, virtual_memory,\n  disk_usage, boot_time,\n  process count\n→ serialised health snapshot"]
    AVM -->|"GET /api/agent/patients"| AG3["Patient.objects.all()\n(no soft-delete filter)\n→ full serialisation for snapshot"]
    AVM -->|"GET /api/agent/predictions"| AG4["Prediction.objects.all()\nEmbed related GoldStandard\n→ full serialisation"]
    AVM -->|"GET /api/agent/gold-standard"| AG5["GoldStandard.objects.all()\n→ full serialisation"]
    AVM -->|"GET /api/agent/tablet-health"| AG6["Connectivity check\n→ {status, timestamp}"]
```

---

## Data Models

```mermaid
erDiagram
    Patient {
        int id PK
        string name
        string rn
        string bed_no
        string gender
        float height
        date date_of_birth
        string race
        date admission_date
        date discharge_date
        string status
        bool is_patient_consent
        datetime deleted_at
    }
    Prediction {
        int id PK
        int patient_id FK
        float value
        float energy_recommendation
        float abw
        float bmi
        float ibw
        float bsa
        int apache
        int sofa
        int saps
        int nutric
        float tv1
        float tv2
        float tv3
        float rr1
        float rr2
        float rr3
        float hr1
        float hr2
        float hr3
        float temp1
        float temp2
        float temp3
        float mv
        string trauma_key
        float trauma_value
        string burns_key
        float burns_value
        datetime deleted_at
    }
    GoldStandard {
        int id PK
        int prediction_id FK
        float reference_ree
        float respiratory_quotient
        float heart_rate
        float variability_in_oxygen_consumption
        float variability_in_carbon_dioxide_consumption
        datetime measurement_timestamp
        string brand_type
    }

    Patient ||--o{ Prediction : "has many"
    Prediction ||--o| GoldStandard : "has one"
```

---

## Request Lifecycle (Full Stack View)

```mermaid
sequenceDiagram
    participant C as Desktop Client
    participant G as Gunicorn (WSGI)
    participant D as Django (views)
    participant S as ML Service
    participant DB as PostgreSQL

    C->>G: POST /api/predictions {28 fields}
    G->>D: dispatch to prediction_views.py
    D->>S: ModelService.predict(data)
    S->>S: prepare_data() — feature engineering
    S->>S: StandardScaler → SVR.predict()
    S->>S: apply injury factor multiplier
    S-->>D: {raw_ree, recommendation}
    D->>DB: Prediction.objects.create(...)
    DB-->>D: prediction record (with id)
    D-->>G: JsonResponse 201
    G-->>C: {prediction_id, value, energy_recommendation}
```
