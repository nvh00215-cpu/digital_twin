# GlycoTwin: A Type 2 Diabetes Digital Twin

**GlycoTwin joins a clinic record to a 14-day wearable trace and tells a doctor which patient is likely to reach 180 mg/dL glucose in the next 2 hours.**


> **The data is synthetic and the model is not clinically validated.** Patients come from Synthea. The wearable series is simulated, not recorded from a real watch or lab-grade CGM. The screen never shows or recommends a drug dose.

| | |
|---|---|
| Live app | https://digitaltwin-neon-phi.vercel.app/login |
| Backend| `https://digital-twin-dkt6.onrender.com/` |
| Demo sign-in | username `doctor` / password `VXpEhqS4gz6iOCoZPc` |
| GitHub | https://github.com/nvh00215-cpu/digital_twin.git |
| Team | `Aeternum`, `Naveen V Hiremath` (`Dr. Ambedkar Institute Of Technology`) |
| Phone Number | 7676191308 |
| Email id | vhnaveen5@gmail.com |

The free Render API sleeps when idle. Open the health-check link first and wait for `"status":"ok"`. The first wake-up can take about a minute.

### At a glance

- **Task:** predict whether glucose reaches 180 mg/dL within 2 hours, and forecast glucose at 30, 60, 90 and 120 minutes.
- **Input:** the past 6 hours of wearable data plus a clinic record, for 339 synthetic patients (76 diabetes, 263 prediabetes).
- **Result:** AUROC **0.986** on 52 held-out patients, against 0.751 for current glucose alone. Split by person.
- **Key finding:** at the same current glucose (140 to 160 mg/dL), the 2-hour spike rate is **0.8%** with HbA1c under 7 and **29.7%** with HbA1c 7 or higher.
- **Product:** a doctor-only web app with a risk-sorted patient list, forecast charts and what-if sliders. Live, with a public repository.

---

## Problem Statement

A glucose reading can look identical for two very different patients. A doctor reviewing a panel of Type 2 diabetes patients needs to know **who is heading for a spike soon**, and why, using the recent glucose trace and the patient's clinical record together. A reading alone does not carry that information.

**Who has the problem:** clinicians who review many patients and have limited time per patient.
**What is missing today:** a view that ranks a panel by near-term risk and explains the risk using both the record and the recent trace.

---

## Solution

GlycoTwin is a doctor-facing dashboard. For each patient it joins a clinic record to a 14-day wearable trace and estimates the chance that glucose reaches 180 mg/dL within the next 2 hours.

| Need | What the doctor gets |
|---|---|
| Who needs attention first? | Patient list **sorted by 2-hour risk**, with High, Moderate and Low filters and search by patient id |
| What is happening now? | Glucose history, heart rate and steps |
| What is likely next? | Four forecast points (30, 60, 90, 120 minutes) against a dashed **180 mg/dL** line, plus the alert |
| Why is this patient different? | The clinic record on the same page |
| What if sleep or activity changes? | Sleep and step sliders that rescore the patient. Results are displayed only and nothing is saved. |
| Is it safe to show? | Doctor-only sign-in, no drug dose anywhere, and a synthetic-data notice on every screen |

---

## Architecture

```
Browser -> Next.js server (sign-in, proxy) -> FastAPI + LightGBM -> CSV data + joblib models
```

- The browser talks **only** to the Next.js server.
- The Next.js server checks a signed session cookie, then proxies `/api/*` calls to the FastAPI service.
- Sign-in uses a bcrypt-hashed password, a signed JWT cookie and rate limiting.
- The API loads the trained models and the data at startup and scores patients on request.

| Layer | Technology | Role |
|---|---|---|
| UI | Next.js, React, Tailwind CSS, Recharts | Patient list, charts, sliders |
| Server | Next.js route handlers | Sign-in, session cookie, proxy to the API |
| API | FastAPI, Uvicorn | `/patients`, `/patients/{id}`, `/patients/{id}/series`, `/patients/{id}/predict`, `/metrics`, `/health` |
| Models | LightGBM, joblib | One spike classifier, four curve regressors |
| Data | Synthea CSVs, simulated wearables | Clinic record and 5-minute series |

More in [`GlycoTwin_ppt.ppt`](GlycoTwin_ppt.ppt) and [`GlycoTwin_Architecture.pdf`](GlycoTwin_Architecture.pdf).

---

## AI / Technical Implementation

### What the model sees

At each decision time the model sees **only the past 6 hours of wearable data plus the clinic record**. Nothing from the future leaks into the features. Labels use the following 2 hours only. Decision times are every 30 minutes.

### Models

| Task | Model | Output |
|---|---|---|
| Spike alert | LightGBM classifier | Probability that glucose goes above 180 mg/dL within 2 hours. The alert turns on at **60%**. |
| Forecast curve | Four LightGBM regressors | Glucose at **+30, +60, +90, +120 minutes** |

The curve is shown to the doctor but does **not** decide the alert.

### Data

| | |
|---|---|
| Clinic record | Synthea: age, sex, BMI, HbA1c, diabetes duration, blood pressure, hypertension, kidney disease, and whether metformin, a sulfonylurea or insulin is on the record |
| Wearables | Simulated 5-minute series, **seed 2026**: glucose, heart rate, steps, one sleep value per night, 14 days |
| Patients | 339 total: **76 with diabetes, 263 with prediabetes** |
| Join | The same `patient_id` links both tables |
| Split | **By person**: 70% train / 15% validation / 15% test, stratified by diabetes and prediabetes. The same person is never in both train and test. |


### Evaluation (52 held-out patients, 34,112 rows, seed 2026)

Positive rate in the test set is 11.95%.

| Model | AUROC |
|---|---|
| **GlycoTwin (record + wearables)** | **0.986** |
| Current glucose only | 0.751 |
| 30-minute slope only | 0.619 |

| Metric | Value |
|---|---|
| AUPRC | 0.912 |
| False alarms at the 60% line | **0.27 per patient-day** |
| Forecast error, MAE at +30 / +60 / +90 / +120 min | 5.5 / 5.9 / 6.4 / 7.6 mg/dL |

Most useful features by model gain: current glucose, hour of day, HbA1c, 6-hour glucose variability, 2-hour mean glucose, 30-minute slope, share of time above 140 mg/dL, and steps in the last 2 hours.

### Reproducibility and checks

- Fixed seed 2026 for the data and the split.
- `validate_data.py` resimulates the first two patients and checks every glucose and sleep value against `signals.csv`, then recomputes their labels. A mismatch exits with status 1.
- Every step from data to metrics is a script in this repository (see "Reproduce the data and models" below).

### How to read these scores

The glucose series is generated by a formula (HbA1c sets the baseline, with meals, steps, sleep and insulin on top). A high AUROC shows the model recovers the structure of that simulation. It is **not** evidence of clinical accuracy.

---

## Innovation and Creativity

- **A twin built from two different sources.** A clinic record (who the patient is) is joined to a wearable trace (what is happening now), by the same patient id. Many glucose forecasters use the trace alone.
- **A test showing the record changes the call.** Take rows where current glucose is already 140 to 160 mg/dL (4,005 test rows), so the recent trace looks the same:

  | HbA1c | 2-hour spike rate |
  |---|---|
  | Under 7 | **0.8%** |
  | 7 or higher | **29.7%** |

- **Alert and forecast are separate on purpose.** The alert comes from a dedicated probability model. The curve is for the doctor to inspect and does not trigger anything.
- **What-if rescoring.** Sleep and step sliders rerun the same simulator and model, so a doctor can see how last night's sleep or daytime activity moves the risk. Nothing is saved.
- **Safety by design.** Doctor-only sign-in, no dose shown anywhere, and a synthetic-data notice on every screen.

---

## Impact and Scalability

### Impact

About **101 million people in India (roughly 11% of the population)** were estimated to have diabetes in the ICMR-INDIAB study ([Gulf News report](https://gulfnews.com/world/asia/india/india-study-estimates-11-of-population-or-101-million-is-diabetic-1.96302385), [Morung Express report](https://morungexpress.com/315-mn-indians-suffer-from-hypertension-101-mn-are-diabetic-icmr-study)). With that many patients and limited clinician time, a tool that ranks a panel by near-term risk could help a clinic spend review time on the patients who need it first.

GlycoTwin is a research prototype. The claims here are about the design, not clinical benefit.

| Who | Benefit |
|---|---|
| Doctors and clinics | A risk-sorted panel and a forecast, with the reason visible in the record |
| Patients | Earlier attention for those heading toward a spike |
| Health systems | A cheap-to-run scoring service that works from data clinics already collect |

### Path to real use

1. **Replace simulated inputs** with real CGM and wearable feeds and real EHR records, joined by patient id.
2. **Retrain and validate prospectively** on real patients, with calibration checks and subgroup analysis, before anyone relies on an alert.
3. **Keep a clinician in the loop.** The tool surfaces risk and shows the forecast. The doctor decides what to do.
4. **Fit the clinic workflow** through the risk-sorted panel.

### Scaling

### Why it scales

The model needs only the **last 6 hours of data (72 readings) plus the clinic record** for each patient, not the full 14-day history. The serving state per patient is therefore small. Scoring is five small LightGBM predictions per patient (one spike classifier and four curve models), which is cheap when batched.

### Roadmap

| Stage | Size | What changes |
|---|---|---|
| **Prototype (now)** | 339 synthetic patients | CSV files loaded into memory, one API process, one demo doctor account |
| **One clinic** | Hundreds to thousands of patients | Database for records and readings, API loads only the latest 6 hours per request, per-doctor accounts and panels, API protected by a shared secret, 2 or more API instances behind a load balancer |
| **Hospital network** | Tens of thousands of patients | Streaming ingest of device and EHR data (for example through FHIR), rolling 6-hour window per patient in a fast store, scheduled batch scoring, risk-sorted list read from stored scores |
| **Multi-site** | Many hospitals | Per-site calibration and validation, model registry with versioning, drift and false-alarm monitoring, audit logs, region-level data storage |

### What changes from the prototype

- **Risk-sorted list:** it needs a score for every patient, so at scale the scores are computed on a schedule and stored instead of computed on each page load.
- **Data loading:** the API currently loads all history into memory at startup. At scale it loads only the latest window per patient, and older history stays in cold storage for charts.

### Beyond one disease: multi-disease support

GlycoTwin targets Type 2 diabetes, but the design is not specific to glucose. The clinic record is shared across diseases (it already includes hypertension, kidney disease and blood pressure), and heart rate, steps and sleep are already in the trace. Only the glucose label, the glucose features and the 180 mg/dL threshold are diabetes-specific.

A new disease would be added as a module with five parts:

| Part | Diabetes today | A new disease |
|---|---|---|
| Signal | Glucose | Its own signal, such as blood pressure or weight |
| Event to predict | Glucose above 180 mg/dL within 2 hours | A clinician-defined event and time window |
| Features | 6-hour glucose window plus the record | Its own window plus the shared record |
| Model | LightGBM classifier and curve models | The same model family, trained per disease |
| Alert rule | 60% probability | A threshold set from that disease's false-alarm rate |

The patient page would show one risk card per disease, built on the same shared clinic record, so a doctor sees one twin per patient with several risk scores.

### Limits of these plans

- The roadmap and the multi-disease modules are **design plans, not built features and not measured benchmarks.**
- Each new disease needs its own signals, clinician-defined labels, and separate validation and false-alarm checks. A good result for diabetes says nothing about another disease.
- Several simultaneous alerts need to be designed to avoid alert fatigue.
- Real deployment needs prospective validation on real patients, plus regulatory and privacy review.

---

## Functionality and Demo

The app is complete and deployed. There is no demo video, so use the live app or the screenshots.

### Try the live app (2 minutes)

1. Open the health-check link above and wait for `"status":"ok"`.
2. Open the live app and sign in with the demo credentials.
3. Use the **High / Moderate / Low** filters and search by patient id.
4. Open the highest-risk patient and check the glucose history, forecast points, 180 mg/dL line, heart rate, steps, record and alert.
5. Move the **sleep** and **step** sliders and watch the risk rescore.
6. Open **Model notes** and read the synthetic-data statement.

### Screenshots

| Risk-sorted patient list | Patient twin | What-if sliders |
|---|---|---|
| ![List](docs/screenshots/list.png) | ![Patient](docs/screenshots/patient.png) | ![What-if](docs/screenshots/whatif.png) |

### Feature checklist

| Feature | Status |
|---|---|
| Doctor-only sign-in | Done |
| Patient list sorted by 2-hour risk, with filters and search | Done |
| Glucose history, four forecast points, dashed 180 mg/dL line | Done |
| Heart rate, steps, clinic record, alert | Done |
| Sleep and step what-if sliders (display only) | Done |
| Model notes page and synthetic-data footer | Done |
| Public repository, reproducible scripts, live deployment | Done |

---

## Run it locally

**Backend** (from the repository root):

```powershell
py -3 -m pip install -r backend/requirements.txt
py -3 -m uvicorn backend.src.api:app --port 8000
```

**Frontend:**

```powershell
cd frontend
npm install
copy .env.example .env.local
npm run dev
```

Open http://localhost:3000.

| Frontend variable | Purpose |
|---|---|
| `BACKEND_URL` | Address of the FastAPI service, no trailing slash |
| `SESSION_SECRET` | Long random string that signs the login cookie |
| `DOCTOR_USERNAME` | The doctor id that can sign in |
| `DOCTOR_PASSWORD_HASH` | bcrypt hash. Create with `node scripts/hash-password.mjs <password>` |

In `.env.local`, write each `$` in the hash as `\$`. In Vercel, paste it as printed. The backend needs no environment variables.

### Reproduce the data and models

```powershell
py -3 -m pip install -r backend/requirements.txt
py -3 data/generate_signals.py
py -3 backend/src/make_dataset.py
py -3 backend/src/train_baseline.py
py -3 backend/src/validate_data.py
py -3 backend/src/write_docs.py
```

### Deployment

- **Backend:** Render web service from the repository root. Build: `pip install -r backend/requirements.txt`. Start: `uvicorn backend.src.api:app --host 0.0.0.0 --port $PORT`.
- **Frontend:** Vercel project with Root Directory `frontend` and the four variables above.

## Limitations

- Glucose traces are **simulated from a formula**, so scores describe the simulation, not real patients.
- Many Synthea HbA1c values were implausible for the stated diagnosis, so the value used for simulation is sometimes **imputed from the diagnosis**. The `hba1c_source` column in `patients_baseline.csv` records which. See `data/DATA_NOTES.txt`.
- The held-out set is 52 patients. There is no clinical validation and no regulatory review.

## Third-party technologies

| Technology | Use |
|---|---|
| [Synthea](https://github.com/synthetichealth/synthea) (Walonoski et al., JAMIA, 2018) | Synthetic patient records |
| [LightGBM](https://github.com/microsoft/LightGBM) | Spike classifier and curve models |
| pandas, NumPy, joblib | Data handling and model storage |
| [FastAPI](https://fastapi.tiangolo.com/), Uvicorn | Prediction API |
| [Next.js](https://nextjs.org/), React, Tailwind CSS | Doctor UI |
| Recharts, TanStack Query | Charts and data fetching |
| Radix UI, lucide-react | UI components and icons |
| jose, bcryptjs, zod | Session cookie, password hashing, input validation |
| Render, Vercel | Hosting |

All are used under their own licenses. No generative AI model runs in the product. The AI component is gradient-boosted trees (LightGBM).

## Repository layout

- `backend/` FastAPI service, training code, `models/baseline.joblib`, `outputs/metrics.json`
- `frontend/` Next.js doctor UI (`app.py` is an earlier Streamlit view, kept for reference)
- `data/` Synthea sample, `patients_baseline.csv`, `signals.csv`, processed rows, `DATA_NOTES.txt`
- `docs/` architecture and presentation PDFs

## License

MIT. See [`LICENSE`](LICENSE).
