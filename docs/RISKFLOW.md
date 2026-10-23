# RiskFlow — ML-First Loan Performance Intelligence Engine

> **AntBox Relevance:** The direct architectural blueprint for AntBox's AI Readiness Engine — calibrated scoring, explainability, drift monitoring, and human-in-the-loop evaluation.

---

## Executive Summary

RiskFlow is an end-to-end, ML-first performance intelligence engine that predicts multiple outcomes (delinquency, default, prepayment), isolates competing risks, stress-tests portfolios against macroeconomic shocks, and provides deep explainability. It culminates in a live Streamlit Command Center and a RAG-grounded LLM Copilot that acts as an AI assistant to human reviewers.

**Source:** [github.com/Subhash-Jetty/RiskFlow](https://github.com/Subhash-Jetty/RiskFlow)

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                       DATA INGESTION                            │
│  Raw loan performance data → Feature Engineering Pipeline       │
│  Rolling averages, ratio computations, .shift(1) temporal guard │
└─────────────────────────┬───────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│                    TIME-AWARE SPLITTER                           │
│  Chronological 70/15/15 split — zero future leakage            │
│  Target encoders fitted ONLY on training set                    │
└──────────┬─────────────────────────────────┬────────────────────┘
           │                                 │
           ▼                                 ▼
┌─────────────────────┐           ┌─────────────────────────┐
│  XGBoost Classifier │           │  CoxPH Survival Model   │
│  + Calibration Layer│           │  (Lifelines)            │
│                     │           │                         │
│  Binary: Platt      │           │  Competing Risks:       │
│  Multiclass: Isotonic│          │  Default vs Prepayment  │
│                     │           │                         │
│  ROC-AUC: 0.998     │           │  C-Index: 0.703         │
│  Macro-F1: 0.936    │           │  Time-to-event curves   │
└──────────┬──────────┘           └──────────┬──────────────┘
           │                                 │
           └────────────┬────────────────────┘
                        │
           ┌────────────▼────────────┐
           │   ANOMALY DETECTION     │
           │  Isolation Forest       │
           │  + Business Rule Engine │
           │  (Hybrid Ensemble)      │
           └────────────┬────────────┘
                        │
           ┌────────────▼────────────┐
           │   STRESS TESTING        │
           │  Monte Carlo Simulation │
           │  Clustered Feature      │
           │  Shock Propagation      │
           │  Value at Risk (VaR)    │
           └────────────┬────────────┘
                        │
           ┌────────────▼────────────┐
           │   EXPLAINABILITY        │
           │  SHAP Feature Waterfalls│
           │  Counterfactual Actions │
           │  Demographic Parity     │
           └────────────┬────────────┘
                        │
           ┌────────────▼────────────┐
           │   HUMAN-IN-THE-LOOP     │
           │  30-70% Uncertainty →   │
           │  Route to Human Reviewer│
           │  Log corrections for    │
           │  model retraining       │
           └────────────┬────────────┘
                        │
           ┌────────────▼────────────┐
           │   LLM COPILOT (RAG)     │
           │  FAISS vector search    │
           │  Domain-grounded only   │
           │                         │
           │  Tier 1: Gemini 3.6     │
           │  Tier 2: Groq Qwen 27B │
           │  Tier 3: Offline Python │
           └────────────┬────────────┘
                        │
           ┌────────────▼────────────┐
           │   STREAMLIT DASHBOARD   │
           │  Real-time PSI tracking │
           │  Model health metrics   │
           │  Drift alerting         │
           └─────────────────────────┘
```

---

## Core Tech Stack

| Component | Technology |
|:---|:---|
| Data Processing | Python, Pandas, NumPy |
| Predictive Modeling | XGBoost, Scikit-Learn, CalibratedClassifierCV |
| Survival Analysis | Lifelines (CoxPH) |
| Explainability | SHAP (SHapley Additive exPlanations) |
| LLM Copilot | Gemini / Groq APIs with FAISS RAG |
| Anomaly Detection | Isolation Forest + Business Rules |
| Frontend UI | Streamlit |
| Deployment | Git, GitHub, Streamlit Community Cloud |

---

## 9 Advanced Engineering Features

### 1. Time-Aware Splitting (Zero Data Leakage)

**What:** Split train/validation/test sets based on origination time, not random sampling.

**Why:** Random K-Fold in temporal data = model trains on 2025 data to "predict" 2023 outcomes. It looks amazing in validation. It fails in production.

**Implementation:**
- Chronological 70/15/15 split along origination vintage and month
- Target encoders fitted **strictly** on training set
- Rolling features use `.shift(1)` to exclude current month's target
- No information from the future ever leaks into the past

**AntBox parallel:** If the Readiness Engine trains on data that includes how students performed in later sprints, it produces inflated backtesting metrics that collapse with real students. Time-aware splitting prevents this.

---

### 2. Calibrated Probability Models

**What:** Wrap XGBoost outputs in CalibratedClassifierCV to produce statistically faithful probabilities.

**Why:** Raw XGBoost outputs aren't true probabilities. A score of 0.8 might actually correspond to a 65% true rate. Calibration fixes this.

**Implementation:**
| Target Type | Calibration Method | Why |
|:---|:---|:---|
| Binary (default/prepayment) | **Platt Scaling** (Sigmoid) | Binary log-odds assumption holds for 2-class problems |
| Multiclass (state transitions) | **Isotonic Regression** | Non-parametric — no distributional assumption. Handles multiclass better than sigmoid. |

**Validation:** Brier Score confirms statistical fidelity.

**AntBox parallel:** When AntBox tells a corporate partner "this candidate is 88% ready," calibration ensures it's actually 88%. Not 72%, not 95%. This is the trust layer that makes the Readiness Score sellable.

---

### 3. Cause-Specific CoxPH Survival Models

**What:** Competing-risk survival analysis using Lifelines library.

**Why:** Standard binary classifiers ask "will this loan default?" Survival models ask "**WHEN** will it default?" But a loan can't default if it prepays first. These are competing risks that contaminate each other's curves.

**Implementation:**
- Separate CoxPH models for Default risk and Prepayment risk
- Competing-risk framework isolates each curve
- Concordance Index: 0.703

**AntBox parallel:** "When will a student reach Advanced proficiency?" isn't a binary question. Students can drop out (competing risk), plateau, or accelerate. Survival analysis models the *trajectory*, not just the endpoint.

---

### 4. Monte Carlo Portfolio Stress Testing

**What:** Simulate thousands of economic scenarios and propagate shocks through the portfolio.

**Why:** Shifting one variable in isolation is unrealistic. A housing price crash correlates with unemployment spikes, which correlate with interest rate changes.

**Implementation:**
- Hierarchical clustering groups correlated features
- Monte Carlo sampling generates shock scenarios
- Shocks propagate through feature clusters, not independently
- Output: Value at Risk (VaR) under severe distress

**AntBox parallel:** When a major university changes its curriculum, how does that affect the readiness distribution of an entire cohort? Stress testing answers this.

---

### 5. SHAP Explainability + Counterfactuals

**What:** Game-theoretic feature attributions with actionable "what-if" scenarios.

**SHAP provides three levels:**
1. **Global importance** — across all predictions, which features matter most
2. **Local attributions** — for this specific prediction, what pushed the score up/down
3. **Counterfactuals** — what's the minimum change needed to cross a threshold

**Example counterfactual:**
> "If you reduce the balance ratio by 5%, the default probability drops from 62% to 47% — below the danger zone."

**AntBox parallel:** SHAP-powered radar charts for each student: "Sprint velocity contributed +12% to your score, but communication quality brought it down -8%. Improving response time in code reviews would push you above the 90th percentile."

---

### 6. Hybrid Anomaly Detection

**What:** Combine Isolation Forest (unsupervised ML) with deterministic business rules.

**Why each alone isn't enough:**
| Approach | Catches | Misses |
|:---|:---|:---|
| Business rules | Negative balances, impossible dates | Complex statistical outliers |
| Isolation Forest | Multi-variate distributional anomalies | Simple logic errors |
| **Hybrid (both)** | Everything | — |

**AntBox parallel:** Business rules catch "student submitted a blank file." Isolation Forest catches "student submitted code that passed all tests but has suspiciously similar structure to 47 other submissions."

---

### 7. Human-in-the-Loop Active Learning

**What:** Route uncertain predictions (30-70% confidence) to human reviewers. Log corrections. Retrain.

**Why:** AI shouldn't make career-altering decisions on edge cases. The model handles the 80% of clear cases efficiently; humans handle the 20% that require judgment.

**Feedback loop:**
```
Model predicts 55% confidence → routed to human reviewer
    │
    Human says "actually default" → correction logged
    │
    Next retraining cycle → model learns from correction
    │
    Over time → 55% uncertainty zone shrinks to 45-55%
```

**AntBox parallel:** The student who wrote mediocre Sprint 1 code but showed exceptional growth in Sprint 3. The AI is uncertain. A mentor reviews and says "high potential — PPO candidate." That correction trains the next model.

---

### 8. RAG-Grounded LLM Copilot

**What:** LLM reviewer grounded in project documentation via FAISS vector search.

**3-Tier Fallback:**
```
Tier 1: Gemini 3.6 Flash (primary)
    │ if Google API fails
    ▼
Tier 2: Groq Qwen 27B (backup)
    │ if no internet
    ▼
Tier 3: Offline Python template
         (injects SHAP values + FAISS contexts deterministically)
```

**Why RAG grounding matters:** LLMs hallucinate financial numbers badly. By restricting the context window to the project's data dictionary, SHAP drivers, and validation rules via FAISS similarity search, the copilot produces accurate, auditable summaries.

**AntBox parallel:** An AI mentor that summarizes a student's sprint performance — must be grounded in actual sprint data, not hallucinated feedback. RAG ensures accuracy.

---

### 9. Streamlit Command Center (PSI Drift Monitoring)

**What:** Live dashboard tracking Population Stability Index and model health.

**Why:** Models degrade silently. If incoming data distributions shift (new university cohorts, different baseline skills), the model's predictions become unreliable. PSI catches this before it causes damage.

**AntBox parallel:** When AntBox onboards 500 students from a new university, the skill distribution may differ from the training cohort. PSI monitoring alerts the team to retrain before scores become meaningless.

---

## Pipeline Architecture

```python
# pipeline.py — Master Orchestrator

def _safe(step_name, fn, *args, **kwargs):
    """Execute a pipeline step with fault isolation.
    If one step fails, the rest keep running."""
    try:
        return fn(*args, **kwargs)
    except Exception as e:
        log.error(f"[{step_name}] Failed: {e}")
        return None

# Pipeline execution
features   = _safe("features",   run_feature_engineering, raw_data)
models     = _safe("train",      run_training,            features)
survival   = _safe("survival",   run_survival_analysis,   features)
anomalies  = _safe("anomaly",    run_anomaly_detection,   features)
stress     = _safe("scenario",   run_stress_testing,      features, models)
explain    = _safe("explain",    run_explainability,      features, models)
copilot    = _safe("copilot",    run_llm_review,          explain)
```

If anomaly detection crashes, training, explainability, and the copilot still complete. No single point of failure.

---

## Performance Metrics

| Model | Metric | Value |
|:---|:---|:---|
| XGBoost (12-month Default) | ROC-AUC | **0.998** |
| XGBoost (Next State Multiclass) | Macro-F1 | **0.936** |
| CoxPH Survival | Concordance Index | **0.703** |
| Top Risk Driver (SHAP) | — | `days_past_due` |

---

*RiskFlow isn't just a credit scoring model — it's a complete scoring, explainability, monitoring, and decision-support architecture. The same patterns power AntBox's AI Readiness Engine.*
