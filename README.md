# INNUTRIRE — System Diagrams

Mermaid flow diagrams documenting the architecture and internal workflows of the INNUTRIRE hospital nutrition management system.

## Diagrams

| File | Repo | What it covers |
|------|------|----------------|
| [01-frontend-innutrire.md](01-frontend-innutrire.md) | `frontend_innutrire` | PySide6 desktop app — screen navigation, 5-step REE wizard, async API threading, state management |
| [02-django-backend-innutrire.md](02-django-backend-innutrire.md) | `django_backend_innutrire` | Django REST API — routing, ML inference pipeline, XAI (SHAP + PFI), data models, Moniteer agent proxy |
| [03-hospital-server-deployment-iac.md](03-hospital-server-deployment-iac.md) | `hospital-server-deployment-iac` | Ansible provisioning — 7-phase playbook, Docker Compose topology, network/firewall/VPN architecture |
| [04-regression-models-innutrire.md](04-regression-models-innutrire.md) | `Regression_models_innutrire` | ML research — data pipeline, 10+ algorithms, evaluation, SHAP interpretability, model export workflow |
| [05-system-overview.md](05-system-overview.md) | **All repos** | End-to-end: clinician enters vitals → prediction result displayed; full sequence diagram + topology |

## System at a Glance

```
┌─────────────────────────────────────────────────────────────┐
│  Regression_models_innutrire  (offline ML research)         │
│  Train → evaluate → export SVR model as .joblib             │
└──────────────────────────┬──────────────────────────────────┘
                           │ deploy .joblib artifact
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  hospital-server-deployment-iac  (Ansible — run once)       │
│  Provision Ubuntu server → Docker Compose: web + db         │
└──────────────────────────┬──────────────────────────────────┘
                           │ provisions & deploys
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  django_backend_innutrire  (always running on server :8000) │
│  REST API — patients, predictions, SHAP/PFI, agent proxy    │
└──────────────────────────┬──────────────────────────────────┘
                           │ HTTP REST API
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  frontend_innutrire  (Windows .exe on ICU bedside tablet)   │
│  PySide6 desktop — patient list → REE wizard → result       │
└─────────────────────────────────────────────────────────────┘
```

## Viewing Diagrams

GitHub renders Mermaid diagrams natively in Markdown files. Open any `.md` file above to see the rendered diagram.

For local preview: use [Mermaid Live Editor](https://mermaid.live) or a VS Code extension with Mermaid support.
