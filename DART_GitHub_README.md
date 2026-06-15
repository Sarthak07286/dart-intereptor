# DART — Digital Arrest Real-Time Interceptor

> **PSB Hackathon Series 2026** · Bank of Baroda · IIT Gandhinagar · Theme: Cybersecurity & Fraud — Identity Trust, Protection & Safety

---

## The Problem

India lost **₹54,000 crore** to digital arrest scams. In a digital arrest fraud:

1. A fraudster calls the victim impersonating CBI / ED / Police
2. The victim is placed on a live video call and told not to move or speak
3. Accused of money laundering, the victim is psychologically coerced
4. Under sustained terror, they transfer large sums — **repeatedly, over hours**
5. Every individual transaction is user-initiated, OTP-verified, and looks completely normal

**No existing bank system catches this.** OTP auth is bypassed. Fraud detection sees nothing unusual. Warning pop-ups are dismissed on the fraudster's instruction.

The fraud isn't in a single transaction. It's in the **pattern across time**.

---

## Our Solution

DART is a **3-layer behavioral detection engine** deployed server-side at the bank. It monitors transaction patterns in real-time and — when a digital arrest signature is detected — silently holds the transaction and triggers a bank callback to the customer.

The fraudster sees nothing. The victim receives a call from their bank and can cancel freely.

### Layer 1 — Transaction Pattern Engine
- 2+ large transfers (>₹50,000) to **new** beneficiaries within 6 hours
- Escalating amounts across consecutive transfers
- Transfers at atypical hours (midnight–6 AM) for senior account holders
- First-time RTGS/IMPS usage by the account holder

### Layer 2 — Account Risk Profiler
- Account holder age 60+ (from KYC demographics)
- Beneficiary account created within last 30 days
- Destination account flagged in RBI MuleHunter Registry
- Geographic mismatch between sender and beneficiary

### Layer 3 — Silent Intervention Module
- Transaction silently queued — app shows `Processing...`
- Fraudster sees **no alert, no warning, no indication**
- Bank initiates outbound Twilio call to customer's registered number
- Customer confirms or cancels — free from coercion

---

## System Architecture

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────────┐
│  Mobile Banking  │    │  API Gateway +   │    │   DART Detection    │
│   App (Flutter)  │───▶│   Kafka Bus      │───▶│   Engine (Python)   │
│  UPI/RTGS/IMPS  │    │  <50ms latency   │    │  Pattern ML + Score │
└─────────────────┘    └──────────────────┘    └──────────┬──────────┘
                                                            │
                              ┌─────────────────────────────▼──────────────┐
                              │           Supporting Data Sources           │
                              │  KYC · Transaction History · Beneficiary DB │
                              │  RBI MuleHunter Registry · Account Flags    │
                              └─────────────────────────────────────────────┘
                                                            │
                        ┌───────────────────────────────────▼──────────────┐
                        │           Decision Orchestrator                   │
                        │   Risk score thresholding (threshold: 0.72)      │
                        └───────────────────────────────────┬───────────────┘
                                                            │
                                               ┌────────────▼───────────┐
                                               │   Intervention Module  │
                                               │  Silent hold (15 min)  │
                                               │  Twilio callback IVR   │
                                               └────────────────────────┘
```

---

## Demo Scenario

A 68-year-old account holder is coerced into 3 transfers over 2 hours:

| Time | Event | Risk Score | Action |
|------|-------|-----------|--------|
| 10:05 AM | Transfer 1 — ₹80,000 to new account | 0.38 | Allowed |
| 11:42 AM | Transfer 2 — ₹2.4L to second new account | 0.61 | Watching |
| 12:17 PM | Transfer 3 — ₹5.0L attempted | **0.84** | **DART TRIGGERS** |
| 12:17 PM | Silent hold activated | — | App shows `Processing...` |
| 12:19 PM | Bank callback to victim | — | She cancels. Fraud stopped. |

🔗 **Live Demo:** [dart-demo.vercel.app](https://dart-demo.vercel.app)

---

## Technology Stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| **Mobile App** | Flutter 3.x + Dart | Cross-platform banking app with mock UPI/IMPS flow |
| **State Management** | Provider / Riverpod | Transaction state, hold state management |
| **API** | Python 3.11 + FastAPI | Async REST endpoints: `/score`, `/hold`, `/release` |
| **Event Streaming** | Apache Kafka | Real-time transaction event bus (<50ms) |
| **Database** | PostgreSQL 15 | Transaction history, account profiles, beneficiary DB |
| **Cache** | Redis | Session state, pattern window management |
| **ML Model** | scikit-learn (GBM) | Gradient Boosted Classifier, trained on synthetic data |
| **Mobile ML** | TensorFlow Lite | On-device inference for offline pattern detection |
| **Feature Eng.** | Pandas / NumPy | Transaction feature extraction and scoring |
| **Intervention** | Twilio Voice API | Outbound IVR callback to customer |
| **IVR** | DTMF tones | Press 1 confirm / Press 2 cancel — no app required |
| **Hosting** | Vercel + Railway | Demo frontend + backend deployment |
| **CI/CD** | GitHub Actions | Automated testing pipeline |

---

## Project Structure

```
dart-interceptor/
├── app/                        # Flutter mobile app
│   ├── lib/
│   │   ├── screens/            # Transaction, hold, confirm screens
│   │   ├── services/           # API client, transaction service
│   │   └── models/             # Transaction, risk score models
│   └── pubspec.yaml
│
├── backend/                    # Python FastAPI backend
│   ├── api/
│   │   ├── routes/
│   │   │   ├── transactions.py # POST /transaction, GET /status
│   │   │   ├── score.py        # POST /score — ML inference endpoint
│   │   │   └── intervention.py # POST /hold, POST /release
│   │   └── main.py
│   ├── ml/
│   │   ├── model.py            # GBM classifier wrapper
│   │   ├── features.py         # Feature extraction pipeline
│   │   └── train.py            # Training script
│   ├── data/
│   │   └── synthetic/          # 5,000+ simulated scenarios
│   ├── kafka/
│   │   └── consumer.py         # Kafka event stream consumer
│   └── requirements.txt
│
├── demo/                       # React web demo (Vercel)
│   ├── src/
│   │   ├── components/         # TransactionFlow, ScoreDisplay, Timeline
│   │   └── App.jsx
│   └── package.json
│
├── docs/
│   ├── architecture.md
│   └── api-reference.md
│
└── README.md
```

---

## Build Roadmap

### Phase 1: Jun 15 – Jun 30 — Detection Foundation
- [x] Rule-based pattern detector (Python)
- [ ] Synthetic dataset: 5,000 transaction scenarios
- [ ] Threshold scoring logic (L1 + L2)
- [ ] PostgreSQL schema + seed data

### Phase 2: Jul 1 – Jul 15 — ML Model + API
- [ ] Train Gradient Boosted Classifier
- [ ] FastAPI endpoints: `/score`, `/hold`, `/release`
- [ ] Kafka event stream integration
- [ ] Unit tests + accuracy benchmarks (target: >92% F1)

### Phase 3: Jul 16 – Jul 26 — App + Live Demo
- [ ] Flutter banking app with mock UPI/IMPS flow
- [ ] Silent hold: `Processing...` state with backend queue
- [ ] Twilio callback integration
- [ ] Bank officer dashboard (React)
- [ ] Demo video

---

## Why This Matters

- **₹54,000 crore** lost annually to digital arrest fraud in India
- **70%+** of victims are elderly (60+)
- **+33%** spike in incidents in 2025 alone
- **0** existing real-time interventions in any Indian bank today

DART is the phone call that frees the victim.

---

## Regulatory Alignment

- RBI's 2026 directive: PSPs must monitor cybersecurity incidents in real-time
- MHA's Digital Arrest Task Force — formal recognition of the threat
- Integrates with **RBI MuleHunter.AI** — feeds intercepted account data to national mule registry

---

## Team

> Submitted for PSB Hackathon Series 2026 · Bank of Baroda Edition
> IIT Gandhinagar Innovation & Entrepreneurship Center (IIEC)
> Department of Financial Services, Government of India

**Sarthak Sapdhare** — EXTC Engineering, SIES Graduate School of Technology, Mumbai University

---

## Links

- 🔗 **Demo:** [dart-demo.vercel.app](https://dart-demo.vercel.app)
- 📧 **Contact:** sarthaksapdhare01@gmail.com
- 🏛️ **Hackathon:** [events.iitgn.ac.in/2026/bobhackathon](https://events.iitgn.ac.in/2026/bobhackathon/)
