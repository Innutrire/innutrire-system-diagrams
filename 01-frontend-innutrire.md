# frontend_innutrire — Flow Diagram

> **What it is:** A PySide6 desktop application (compiled to a Windows `.exe`) that runs on bedside tablets in the ICU. Clinicians use it to register patients, enter clinical measurements, and trigger REE (Resting Energy Expenditure) predictions from the Django backend.

---

## Application Navigation & Screen Flow

```mermaid
flowchart TD
    A([run.py — QApplication Launch]) --> B[MainWindow created]
    B --> C[Navigation: QStackedWidget initialised]
    C --> D[SELECT_PATIENT_PAGE]

    D --> E{User action}
    E -->|Click Add Patient| F[AddNewPatientPage]
    E -->|Click existing patient row| G[PatientDashboardPage]

    F --> F1["Fill form\n(name, RN, race, gender,\nDOB, bed_no, height)"]
    F1 --> F2["Save →\nPOST /api/patients"]
    F2 --> D

    G --> G1["GET /api/patients/{id}"]
    G --> G2["GET /api/patients/{id}/latest-prediction"]
    G --> G3["GET /api/patients/{id}/insights"]
    G1 & G2 & G3 --> H[Dashboard rendered]

    H --> I{User action}
    I -->|REE Calculation| J[REECalculationPage]
    I -->|History| K[REEHistoryPage]
    I -->|Predictive Equations| L[PredictiveEquationsPage]
    I -->|Edit| M[EditPatientPage]
    I -->|Patient Consent| N["POST /api/patients/{id}/consent"]
```

---

## REE Calculation Wizard (5-Step Form)

```mermaid
flowchart TD
    J([REECalculationPage]) --> J0

    J0["Form 0 — Gold Standard\n(Indirect Calorimetry)\nRQ, HR, VO2 variability,\nVCO2 variability, reference REE,\nbrand type, measurement timestamp"]
    J0 -->|Form complete| J1

    J1["Form 1 — Demographics\nInput: ABW\nAuto-calc: BMI, IBW, BSA\nRead-only: Height, Gender, Age"]
    J1 -->|Form complete| J2

    J2["Form 2 — Severity Scores\nAPACHE II (0–74)\nSOFA (0–24)\nSAPS2 (0–163)\nNUTRIC (0–40)\n↳ Each has optional calculator modal"]
    J2 -->|Form complete| J3

    J3["Form 3 — Vital Signs (×3 readings each)\nBody Temperature °C\nRespiratory Rate bpm\nHeart Rate bpm\nTidal Volume mL\nMinute Ventilation (auto-calc)"]
    J3 -->|Form complete| J4

    J4["Form 4 — Injury Details\nTrauma table: Mild / Moderate / Severe / N/A\nBurns table: Mild / Moderate / Severe / N/A\n(radio selection, one per table)"]
    J4 -->|Form complete| J5

    J5{All 5 forms complete?}
    J5 -->|No| J2
    J5 -->|Yes| J6[Calculate button enabled]
    J6 --> J7([User clicks Calculate])
```

---

## Prediction Submission & Result Display

```mermaid
sequenceDiagram
    participant UI as CalcResetButtons (UI)
    participant W as ApiWorker (QThread)
    participant PP as PredictionProvider
    participant GP as GoldStandardProvider
    participant API as Django Backend API
    participant R as ResultLayout (UI)

    UI->>W: Spawn ApiWorker thread
    W->>PP: make_prediction(**ree_calculation_data)
    PP->>API: POST /api/predictions
    Note right of API: patient_id, abw, height, gender,<br/>age, bmi, ibw, bsa, apache, sofa,<br/>saps, nutric, tv1-3, rr1-3, mv,<br/>hr1-3, temp1-3, trauma_key/value,<br/>burns_key/value
    API-->>PP: {prediction_id, value, energy_recommendation}

    W->>GP: add_new_gold_standard(prediction_id, **gold_standard_data)
    GP->>API: POST /api/gold-standard
    Note right of API: prediction_id, reference_ree, RQ,<br/>HR, VO2 variability, VCO2 variability,<br/>measurement_timestamp, brand_type
    API-->>GP: 201 Created

    W-->>UI: finished signal → {prediction, recommendation}
    UI->>R: finish_calculation_signal emitted
    R->>R: Display AI REE kcal/day
    R->>R: Display Total Energy Recommendation kcal/day
```

---

## State Management

```mermaid
flowchart LR
    subgraph Global["app_context (singleton)"]
        N[navigator: Navigation]
        T[theme: Theme]
        S[status_bar_widget]
    end

    subgraph FormState["FilledInFormSignal (singleton)"]
        RD["ree_calculation_data\n32 clinical attributes"]
        GD["gold_standard_data\n7 IC measurements"]
        FC["form_complete flags\n(5 booleans)"]
    end

    subgraph Signals
        S1[complete_fill_in_form_signal]
        S2[complete_fill_in_all_forms]
        S3[finish_calculation_signal]
        S4[enable_button_signal]
    end

    FC -->|all True| S2
    S2 --> CalcBtn[Calculate button enabled]
    S3 --> ResultPanel[Result panel updated]
    S4 --> SideButtons[History / History / PE buttons enabled]
```

---

## Threading Architecture

```mermaid
flowchart TD
    MT[Main Thread — Qt Event Loop] -->|spawns| WT[ApiWorker: QThread]
    WT -->|runs| F["_run_two_functions()"]
    F --> P1["PredictionProvider.make_prediction()"]
    P1 -->|HTTP| API1["POST /api/predictions"]
    API1 -->|response| F
    F --> P2["GoldStandardProvider.add_new_gold_standard()"]
    P2 -->|HTTP| API2["POST /api/gold-standard"]
    API2 -->|response| F
    F -->|result| WT
    WT -->|finished.emit| MT
    MT --> UI[UI update: show result]
    WT -->|error.emit on exception| MT
    MT -->|error| EH[Re-enable Calculate button]
```

---

## API Surface Consumed

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/api/patients` | List all patients |
| POST | `/api/patients` | Register new patient |
| GET | `/api/patients/{id}` | Patient detail |
| PUT | `/api/patients/{id}` | Update patient |
| DELETE | `/api/patients/{id}` | Soft-delete patient |
| POST | `/api/patients/{id}/consent` | Record consent |
| GET | `/api/patients/{id}/latest-prediction` | Most recent REE result |
| GET | `/api/patients/{id}/predictions` | All REE history |
| GET | `/api/patients/{id}/insights` | Latest vs previous delta |
| POST | `/api/predictions` | **Run ML inference → REE result** |
| POST | `/api/gold-standard` | Store IC calorimetry measurement |
