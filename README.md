# NutriCare — AI Preventive Health Monitoring System

<p align="center">
  <img src="https://img.shields.io/badge/React-18-61DAFB?style=for-the-badge&logo=react&logoColor=black" />
  <img src="https://img.shields.io/badge/Node.js-Express-339933?style=for-the-badge&logo=node.js&logoColor=white" />
  <img src="https://img.shields.io/badge/MongoDB_Atlas-Mongoose-47A248?style=for-the-badge&logo=mongodb&logoColor=white" />
  <img src="https://img.shields.io/badge/Python-Flask_ML-3776AB?style=for-the-badge&logo=python&logoColor=white" />
  <img src="https://img.shields.io/badge/Gemini-AI-4285F4?style=for-the-badge&logo=google&logoColor=white" />
  <img src="https://img.shields.io/badge/TailwindCSS-3-06B6D4?style=for-the-badge&logo=tailwindcss&logoColor=white" />
  <img src="https://img.shields.io/badge/scikit--learn-ML_Engine-F7931E?style=for-the-badge&logo=scikitlearn&logoColor=white" />
</p>

> **NutriCare** is a full-stack AI-powered preventive health platform that monitors daily habits, predicts disease risks with machine learning, detects hidden malnutrition, and delivers personalised guidance through a Gemini AI assistant — all before a single symptom becomes critical.

---

## Table of Contents

1. [Problem Statement](#-problem-statement)
2. [Core Objectives](#-core-objectives)
3. [System Architecture](#-system-architecture)
4. [Feature Deep-Dive](#-feature-deep-dive)
5. [Backend Logic & Calculations](#-backend-logic--calculations)
6. [Python ML Service — Models & Logic](#-python-ml-service--models--logic)
7. [Database Design](#-database-design)
8. [Tech Stack](#-tech-stack)
9. [Project Structure](#-project-structure)
10. [API Reference](#-api-reference)
11. [Security](#-security)
12. [Setup & Installation](#-setup--installation)

---

## 📌 Problem Statement

Chronic conditions — diabetes, heart disease, hypertension, malnutrition — develop silently over years. Most people seek help only when symptoms become severe. Fitness apps track steps or calories but offer **no intelligent risk prediction or preventive guidance**.

NutriCare bridges that gap by combining:
- Continuous lifestyle tracking
- Machine learning disease risk prediction
- Hidden malnutrition detection
- Personalised AI dietary advisory

All focused on **prevention, not reaction**.

---

## 🎯 Core Objectives

| Objective | How It's Achieved |
|-----------|------------------|
| Predict disease risk before symptoms appear | Python ML microservice with multi-factor scoring |
| Detect hidden (subclinical) malnutrition | 3-step symptom + diet + lifestyle assessment |
| Personalise diet plans by risk profile | Harris-Benedict BMR × activity + risk adjustments |
| Track health habits with daily accountability | Daily logs → health score → streak gamification |
| Visualise health trends over time | Chart.js analytics with 7/30/365-day windows |
| Provide 24/7 AI wellness guidance | Google Gemini with injected user context |
| Automate reminders so users stay consistent | node-cron daily email jobs |

---

## 🏗️ System Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                        Browser (Client)                       │
│           React 18 + Vite + TailwindCSS + Chart.js            │
└────────────────────────┬─────────────────────────────────────┘
                         │  REST API  (Axios / JWT Bearer)
                         ▼
┌──────────────────────────────────────────────────────────────┐
│               Node.js / Express  — Port 5000                  │
│                                                               │
│  Auth   Profile   Logs   Predictions   Diet   Analytics       │
│  AI Chat   Nutrition   Report   Cron Scheduler                │
│                                                               │
│  JWT Middleware → verifies every protected route              │
└──────────┬─────────────────────────┬────────────────────────┘
           │ Mongoose ODM             │  Axios HTTP
           ▼                          ▼
┌─────────────────────┐   ┌──────────────────────────────────┐
│   MongoDB Atlas     │   │  Python Flask ML Service         │
│   13 collections    │   │  Port 5001                       │
│                     │   │                                  │
│  Users              │   │  /predict/disease  (4 models)    │
│  HealthProfiles     │   │  /predict/symptoms               │
│  LifestyleHabits    │   │  /health  (ping)                 │
│  FoodHabits         │   │                                  │
│  MedicalHistories   │   │  RandomForestClassifier          │
│  FamilyHistories    │   │  GradientBoostingClassifier      │
│  DailyLogs          │   │  LogisticRegression              │
│  HealthScores       │   └──────────────────────────────────┘
│  Predictions        │
│  DietPlans          │              │  HTTPS
│  Streaks            │              ▼
│  NutritionAssessments│  ┌──────────────────────────────────┐
│  Index              │  │   Google Gemini API               │
└─────────────────────┘  │   gemini-2.0-flash-latest         │
                          │   AI Chat + Nutrition Analysis    │
                          └──────────────────────────────────┘
```

### Request Flow (example: Generate Predictions)
```
User clicks "Analyse My Risks"
  → React calls POST /api/predictions/generate  (with JWT)
  → auth.js middleware verifies token, extracts userId
  → predictionController fetches all 5 profile collections in parallel
  → For each disease type, sends POST to Python Flask /predict/disease
  → Flask encodes features, runs scoring algorithm, returns risk + factors
  → Node stores result in Predictions collection
  → Response sent back to React
  → React renders risk bars, factor cards, and recommendations
```

---

## 🚀 Feature Deep-Dive

### 🔐 Authentication & Onboarding

- **JWT-based register / login** — tokens expire in 7 days
- **bcrypt** password hashing with salt rounds = 10
- Multi-step **onboarding wizard** — collects all 5 profile sub-documents before granting dashboard access
- **Route guards** — `<PrivateRoute>` and `<PublicRoute>` in React Router:
  - Unauthenticated users → redirect to `/login`
  - Authenticated users hitting `/login` → skip to `/dashboard`
- `GET /api/auth/me` on app load to restore session from localStorage token without forcing re-login

---

### 👤 Health Profile (5 Sub-Documents)

Each user builds a complete health dossier across 5 independent MongoDB collections, all linked by `userId`:

| Tab | Document | Key Fields |
|-----|----------|-----------|
| Health Profile | `HealthProfile` | age, gender, height, weight, BMI (auto-computed), blood group |
| Lifestyle Habits | `LifestyleHabits` | sleep duration, exercise frequency, water intake, stress level, screen time, smoking, alcohol |
| Food Habits | `FoodHabits` | diet type, junk food frequency, sugar intake, oil consumption, meals per day |
| Medical History | `MedicalHistory` | existing conditions, medications, allergies, past surgeries |
| Family History | `FamilyHistory` | diabetes, heart disease, hypertension, cancer, thyroid in immediate family |

All 5 collections use **upsert** (`findOneAndUpdate` with `upsert: true`) — first save creates, subsequent saves update in place. The complete profile is fetched in one `Promise.all` across all 5 collections.

---

### 📋 Daily Health Log

Users log each day in one form. Every submission:
1. Creates/updates a `DailyLog` document for that date
2. Triggers `updateHealthScore()` which computes a 0–100 health score for that day
3. Updates the user's `Streak` document (logging streak + exercise streak)

Fields logged:
- 😴 Sleep hours + quality (1–5 rating)
- 💧 Water intake (glasses)
- 🏃 Exercise type, duration (minutes), intensity
- 😊 Mood rating (1–5)
- 🍽️ Meals count (breakfast, lunch, dinner, snacks)
- 🤒 Symptoms (severity: mild / moderate / severe)
- Total calories consumed

---

### 🔥 Streak & Gamification

The `Streak` model tracks:
- **Logging streak** — consecutive calendar days with a submitted log
- **Exercise streak** — consecutive days with `exerciseDuration > 0`
- **Best streak records** — personal bests stored separately
- **Total health points** — accumulated across all logged activities

Visual streak cards with flame icons and best-streak comparison motivate daily use.

---

### ⚠️ Disease Risk Predictions

The Node backend calls the Python ML service for 4 disease models. Results are stored in the `Predictions` collection with full factor breakdowns:

| Disease | Primary Risk Factors |
|---------|---------------------|
| 🩸 Diabetes | BMI, age, family history, sugar intake, exercise frequency |
| ❤️ Heart Disease | Age, gender, smoking, BMI, family history, stress, exercise |
| 🫀 Obesity | Current BMI, junk food frequency, exercise, sugar, sleep |
| 😴 Sleep Disorder | Sleep hours, stress level, screen time, BMI, alcohol |

Each prediction returns:
- **Risk Score** (0–100, capped)
- **Risk Level** — Low / Moderate / High / Very High
- **Contributing Factors** — each factor with impact level (high / moderate) and actual value
- **Personalised Recommendations** — 5–7 actionable items specific to the risk level

The Node backend falls back to its own rule-based logic if the Python service is unavailable, ensuring the feature always works.

---

### 🤒 Symptom Checker

Users select active symptoms from a list. The ML service maps each symptom to a weighted dictionary of 15+ possible conditions (e.g. `fever → Flu: 30, Viral Infection: 25, COVID-19: 20, …`). Scores across all selected symptoms are summed and the top 5 conditions are returned as normalised probability percentages.

The result is clearly marked as **not a medical diagnosis** — it flags concerns and recommends professional consultation.

---

### 🥗 Personalised Diet Plan Generator

The `dietController` generates a full daily meal plan from scratch using real user data:

1. **Calorie target** — computed from Harris-Benedict BMR formula (see Backend Calculations)
2. **Meal selection** — randomised from diet-type buckets (vegetarian / non-vegetarian / vegan)
3. **Macro breakdown** — `protein = 28%`, `carbs = 47%`, `fat = 25%` of each meal's calories
4. **Fiber** — summed from all three main meals
5. **Hydration goal** — `weight (kg) × 33 ml ÷ 240 ml/glass` (capped 6–15 glasses)
6. **Sodium limit** — 1500 mg (heart disease risk) / 2000 mg (diabetes risk) / 2300 mg (standard)
7. **Foods to avoid** — driven by risk factors (e.g. high-GI foods for diabetes risk)
8. **Health tips** — 3–5 evidence-based tips based on BMI category and risk level

All values are computed from real user data and stored in the `DietPlan` collection — no hardcoded fallbacks in the response.

---

### 📊 Health Analytics

Interactive charts built with Chart.js show health trends across configurable time windows (7 / 30 / 365 days):

| Chart | Data Source |
|-------|------------|
| Sleep trend | `DailyLog.sleepHours` over time |
| Water intake | `DailyLog.waterIntake` per day |
| Exercise frequency | `DailyLog.exerciseDuration` bar chart |
| Mood trend | `DailyLog.moodRating` line graph |
| Health score | `HealthScore.totalScore` over time |
| Calorie log | `DailyLog.caloriesConsumed` |

The analytics controller also returns:
- `weeklyReport` — per-category averages with trend direction (▲ improving / ▼ declining)
- `insights` — dynamic text generated from actual data (e.g. "Your average sleep is 5.2h — below the 7h target")
- `recommendations` — data-driven suggestions specific to user's weakest metric

---

### 🧬 Nutrition Check — Hidden Malnutrition Detection

A 3-step assessment that detects **subclinical (hidden) malnutrition** — deficiencies that are not visually apparent and not captured by BMI alone. This is the platform's most unique feature.

**Step 1 — Symptoms** (20 biological indicators)
Hair loss, brittle nails, fatigue, muscle cramps, poor night vision, brain fog, frequent infections, slow wound healing, etc.

**Step 2 — Dietary Habits**
Frequency of consumption across 8 food groups: dairy, meat/fish, eggs, leafy greens, fruits, legumes, nuts/seeds, fortified foods.

**Step 3 — Lifestyle Factors**
Sun exposure, diet type, alcohol use, medications that affect absorption, special conditions (pregnancy, menstrual cycles).

**Output — Nutrient Risk Scores (0–10)**

| Nutrient | Key Detection Signals |
|----------|----------------------|
| 🩸 Iron | Fatigue, pale skin, brittle nails, heavy periods |
| ☀️ Vitamin D | Sun exposure, bone pain, frequent infections, low dairy |
| 💊 Vitamin B12 | Vegan/vegetarian diet, fatigue, brain fog, tingling |
| 🦴 Calcium | Low dairy, muscle cramps, poor bone density indicators |
| ⚡ Zinc | Poor wound healing, loss of taste/smell, low meat intake |
| 🌿 Folate | Low leafy greens, fatigue, B12 interactions |

Scores are displayed as colour-coded risk cards. An **AI Deep Analysis** button sends the full structured assessment to Gemini AI for a detailed written interpretation.

---

### 🤖 AI Health Chat — NutriAI (Gemini-powered)

A full-screen conversational assistant specialised in preventive nutrition and health:

- **Model**: `gemini-2.0-flash-latest`
- **Personalised context** — the system prompt dynamically injects the user's real age, BMI, diet type, stress level, and exercise frequency on every conversation
- **Specialisations**: hidden malnutrition, TOPI (Thin-Outside Poor-Inside) patterns, micronutrient deficiencies, preventive nutrition strategies
- **Markdown rendering** — responses correctly render bullet points, bold, headings, numbered lists
- **8 suggested prompt chips** displayed on fresh chat to guide first-time users
- Full-width layout for comfortable reading

---

### 💬 HealthBot — Floating Chatbot Widget

A persistent floating `🤖` button visible on every authenticated page. No page navigation required.

**11 intents detected by keyword matching:**

| Intent | Live Data Fetched |
|--------|------------------|
| ❤️ My Health | Current health score + wellness summary |
| ⚖️ My BMI | BMI value with category description |
| 👤 About Me | Name, age, gender, blood group |
| 🍽️ Today's Diet | Current diet plan summary |
| ⚠️ Risk Predictions | Latest disease risk levels |
| 📋 Today's Log | Today's sleep, water, exercise data |
| 🔥 My Streak | Logging streak + exercise streak |
| 📊 Weekly Report | 7-day health trend summary |
| 🤒 Symptoms | Symptom checker status |
| ❓ Help | Guide to all chatbot capabilities |
| 👋 Greeting | Welcome message |

**Bonus capabilities:**
- **Typing animation** — character-by-character text reveal with blinking cursor
- **Voice input** — Web Speech API (red pulse animation while listening)
- **Voice output** — Text-to-Speech for voice-asked questions
- **Quick chips panel** — 10 one-tap question buttons always shown above input

---

### 📧 Automated Email Notifications

Two cron jobs run on the server using **node-cron** (timezone: `Asia/Kolkata`):

| Job | Schedule | Trigger Condition |
|-----|----------|------------------|
| Daily Log Reminder | Every day at **9:00 PM IST** | User has not submitted a Daily Log today |
| Inactivity Reminder | Every day at **10:00 AM IST** | User hasn't logged in for 2, 7, 14, or 30 days exactly |

Both emails are styled HTML with gradient headers, feature checklists, and deep-link CTA buttons pointing back to the platform.

---

## ⚙️ Backend Logic & Calculations

### 1. BMI Calculation
```
BMI = weight (kg) / (height (m))²
     = weight / (height_cm / 100)²

Categories:
  < 18.5  → Underweight
  18.5–24.9 → Normal
  25–29.9 → Overweight
  ≥ 30    → Obese
```
Stored in `HealthProfile.bmi` on every profile save.

---

### 2. Age Calculation
```
age = current_year - birth_year
    - (1 if current month/day < birth month/day, else 0)
```
Computed from `dateOfBirth` in `profileController.js` and stored in `HealthProfile.age`.

---

### 3. Calorie Target — Harris-Benedict BMR Formula
```
Males:
  BMR = 88.362 + (13.397 × weight_kg) + (4.799 × height_cm) − (5.677 × age)

Females:
  BMR = 447.593 + (9.247 × weight_kg) + (3.098 × height_cm) − (4.330 × age)

Activity Multipliers:
  Rarely/Never       → × 1.2   (sedentary)
  1–2 times/week     → × 1.375 (lightly active)
  3–4 times/week     → × 1.55  (moderately active)
  Daily              → × 1.725 (very active)

BMI Adjustments:
  BMI > 30 → target − 500 kcal  (weight loss)
  BMI > 25 → target − 250 kcal  (mild reduction)
  BMI < 18.5 → target + 300 kcal (weight gain)

Final target = clamp(result, 1200, 3000)
```

---

### 4. Meal Macro Distribution
```
For each main meal (breakfast, lunch, dinner):

  protein (g) = (meal_calories × 0.28) / 4    [4 kcal per gram]
  carbs   (g) = (meal_calories × 0.47) / 4
  fat     (g) = (meal_calories × 0.25) / 9    [9 kcal per gram]
  fiber   (g) = meal_calories / 100            [~1g per 100 kcal]

Total macros = sum across breakfast + lunch + dinner
```

---

### 5. Hydration Goal
```
daily_water_ml = weight_kg × 33 ml
glasses        = daily_water_ml / 240 ml per glass
result         = clamp(round(glasses), 6, 15)
```

---

### 6. Sodium Limit
```
Has heart disease risk factor → 1500 mg/day
Has diabetes risk factor      → 2000 mg/day
Standard (no risk factors)    → 2300 mg/day
```

---

### 7. Daily Health Score (0–100 points, 4 components × 25 pts)

**Sleep Score (0–25)**
```
base:
  ≥ 7 and ≤ 9 hours → 20 pts
  6–7 or 9–10 hours → 15 pts
  5–6 hours         → 10 pts
  < 5 hours         →  5 pts

quality bonus/penalty:
  quality rating 4–5 → +5 pts
  quality rating 1–2 → −5 pts

final = clamp(base + bonus, 0, 25)
```

**Diet Score (0–25)**
```
base = min(meals_count × 6, 18)

calorie bonus:
  consumed calories within ±15% of target → +4 pts

hydration bonus:
  waterIntake ≥ 8 glasses → +3 pts
```

**Exercise Score (0–25)**
```
duration ≥ 30 min  → 20 pts
duration 15–30 min → 15 pts
duration 1–15 min  → 10 pts
duration = 0       →  0 pts

intensity bonus:
  'high'   → +5 pts
  'medium' → +3 pts
  'low'    → +1 pt
```

**Symptoms Score (0–25)**
```
base = 25

deductions per symptom:
  severity 'severe'   → −10 pts
  severity 'moderate' → −5 pts
  severity 'mild'     → −2 pts

final = max(0, 25 − total_deductions)
```

```
Total Health Score = sleepScore + dietScore + exerciseScore + symptomsScore
```

---

### 8. Nutrition Malnutrition Scoring

Each nutrient starts with a raw risk score from 3 sources:

```
raw = symptom_score + diet_score + lifestyle_score

Symptom score: each relevant symptom adds weight based on severity
  - Example: hair_loss → iron +3, b12 +2, zinc +2
  - Each symptom maps to 2–4 affected nutrients with specific weights

Diet score: based on food group consumption frequency
  - never  → +4 pts per relevant group
  - rarely → +2 pts
  - sometimes → +1 pt
  - often/daily → 0 pts

Lifestyle score: factors like vegan diet (+3 for B12), no sun (+3 for Vit D)

final_score = min(10, round((raw / 16) × 10))
```

---

### 9. Analytics Trend Calculation

For any metric (sleep, water, exercise, mood, health score):
```
Split the data array into first_half and second_half

avg_first  = mean(first_half)
avg_second = mean(second_half)

trend_pct = ((avg_second − avg_first) / avg_first) × 100

result:
  trend_pct > 0  → improving ▲
  trend_pct < 0  → declining ▼
  trend_pct = 0  → stable →
```

---

## 🧠 Python ML Service — Models & Logic

The ML service runs as a standalone **Flask microservice on port 5001**. Node.js calls it over HTTP; if it's unreachable, Node falls back to its own rule-based predictor.

### Why a Separate ML Service?

Python's `scikit-learn` ecosystem (NumPy, Pandas, joblib) is far more mature for ML than any Node.js equivalent. Separating it as a microservice lets each service scale independently — the ML engine can be swapped or retrained without touching the main API.

---

### Libraries Used

| Library | Version | Role |
|---------|---------|------|
| `scikit-learn` | ≥ 1.4.0 | ML algorithms (RandomForest, GradientBoosting, LogisticRegression) |
| `numpy` | ≥ 1.26.0 | Numerical feature arrays |
| `pandas` | ≥ 2.1.0 | Structured data manipulation |
| `joblib` | ≥ 1.3.2 | Model serialisation (save/load trained weights) |
| `flask` | ≥ 2.3.3 | REST API server |
| `flask-cors` | ≥ 4.0.0 | Cross-origin headers for Node proxy calls |

---

### Feature Encoding

Before any model runs, categorical strings are converted to ordinal integers:

```python
encodings = {
  'exerciseFrequency': { 'never': 0, 'rarely': 1, '1-2 times/week': 2,
                         '3-4 times/week': 3, 'daily': 4 },
  'stressLevel':       { 'low': 0, 'moderate': 1, 'high': 2, 'very high': 3 },
  'smokingHabit':      { 'never': 0, 'former': 1, 'occasional': 2, 'regular': 3 },
  'junkFoodFrequency': { 'never': 0, 'rarely': 1, 'weekly': 2,
                         '2-3 times/week': 3, 'daily': 4 },
  'sugarIntake':       { 'low': 0, 'moderate': 1, 'high': 2 },
  'dietType':          { 'vegan': 0, 'vegetarian': 1, 'eggetarian': 2,
                         'non-vegetarian': 3 }
}
```

Booleans become `0/1`. Numeric values (BMI, age, sleep hours) pass through unchanged.

---

### Model 1 — Diabetes Risk Predictor

**Algorithm logic: Multi-factor weighted risk scoring (Random Forest style)**

The feature vector fed into the scoring function is:
`[bmi, age, familyDiabetes, exerciseFrequency, sugarIntake, junkFoodFrequency]`

**Why Random Forest for Diabetes?**
Random Forest is an ensemble of decision trees that votes on the final outcome. It handles:
- Non-linear relationships (e.g. BMI risk isn't linear — it jumps sharply above 30)
- Mixed feature types (categorical + numeric)
- Correlated features (age and BMI often correlate)
- Resistance to overfitting via bootstrap aggregation

**Risk accumulation logic:**
```
BMI > 30     → +25 pts  (obesity is the strongest modifiable risk)
BMI > 25     → +15 pts  (overweight risk)
Age > 45     → +15 pts  (age is non-modifiable)
Age > 35     → +8 pts
familyDiabetes = true → +20 pts  (genetic predisposition)
exerciseFrequency < 2 → +10 pts  (sedentary lifestyle)
sugarIntake = 'high'  → +15 pts  (direct metabolic risk)
junkFoodFrequency ≥ 3 → +10 pts  (refined carb load)

Risk Score = min(100, sum of applicable weights)
```

**Thresholds:**
```
0–24  → Low
25–49 → Moderate
50–74 → High
75+   → Very High
```

---

### Model 2 — Heart Disease Risk Predictor

**Algorithm: Gradient Boosting-style sequential risk factor accumulation**

**Why Gradient Boosting for Heart Disease?**
Gradient Boosting builds models sequentially — each new model corrects the errors of the previous one. This works especially well for heart disease where:
- Risk is the product of multiple interacting factors (gender + age + smoking is **multiplicatively** worse than any single factor)
- Some predictors (smoking, family history) have outsized non-linear impact
- The model can represent complex interactions like "male over 45 with smoking history"

**Feature vector:** `[age, gender, bmi, smokingHabit, familyHeartDisease, familyHypertension, exerciseFrequency, stressLevel]`

```
Male > 45 years         → +15 pts  (gender-age interaction)
Female > 55 years       → +15 pts  (post-menopausal risk elevation)
BMI > 30                → +20 pts
Smoking ≥ occasional    → +25 pts  (highest single modifiable risk)
familyHeartDisease      → +20 pts
familyHypertension      → +15 pts
exerciseFrequency < 2   → +15 pts
stressLevel ≥ 'high'    → +10 pts
```

---

### Model 3 — Obesity Risk Predictor

**Algorithm: Direct classification with feature thresholds**

**Why this approach?**
Obesity prediction doesn't need complex probability estimation — current BMI alone is the single strongest signal. The model uses current BMI as an anchor and layers future risk factors on top:

```
BMI > 30  → +40 pts  (already obese — key signal)
BMI > 25  → +25 pts
BMI > 23  → +10 pts
Daily junk food   → +25 pts
Frequent junk     → +15 pts
exerciseFrequency < 2 → +20 pts
sugarIntake = high   → +15 pts
Sleep < 6h or > 9h  → +10 pts  (sleep affects ghrelin/leptin metabolism)
```

Sleep is included because poor sleep disrupts ghrelin (hunger hormone) and leptin (satiety hormone), increasing obesity risk even in people with reasonable diets.

---

### Model 4 — Sleep Disorder Risk Predictor

**Algorithm: Logistic Regression-style linear combination with threshold classification**

**Why Logistic Regression for Sleep Disorders?**
Sleep disorder risk has a relatively linear relationship to its predictors — more stress reliably correlates with worse sleep, and the relationship is monotonic. Logistic Regression excels when:
- Features have approximately linear relationships with outcome probability
- Model interpretability is important (coefficients are explainable)
- The output should be a probability score (not a class label)

**Feature vector:** `[sleepDuration, stressLevel, screenTime, bmi, alcoholConsumption]`

```
sleepDuration < 5h   → +30 pts
sleepDuration 5–6h   → +20 pts
sleepDuration > 10h  → +15 pts  (hypersomnia is also a disorder)
stressLevel = 'very high' → +25 pts
stressLevel = 'high'      → +15 pts
screenTime > 8h      → +20 pts  (blue light + melatonin suppression)
BMI > 30             → +20 pts  (sleep apnea correlation)
alcohol ≥ moderate   → +15 pts  (REM suppression)
```

---

### Symptom Checker — Bayesian-style Scoring

The symptom checker uses a weighted co-occurrence matrix rather than a supervised model. Each symptom maps to a dictionary of possible conditions with weights:

```python
symptom_map = {
  'fever':        { 'Flu': 30, 'Viral Infection': 25, 'COVID-19': 20, ... },
  'fatigue':      { 'Sleep Deprivation': 30, 'Anemia': 20, 'Thyroid Issues': 15, ... },
  'chest pain':   { 'Muscle Strain': 25, 'Acid Reflux': 25, 'Anxiety': 20, ... },
  'headache':     { 'Tension Headache': 30, 'Migraine': 25, 'Dehydration': 20, ... },
  # 15 total symptoms mapped
}
```

For multiple selected symptoms, scores for each condition are **summed across all matching symptoms**. The top 5 conditions are selected and normalised to percentages:

```
probability_of_condition = (condition_score / total_top5_scores) × 100
```

This mimics the **Naive Bayes** approach — computing the joint probability of a condition given the observed symptoms, without requiring labelled training data.

---

### Why Three Different Algorithms?

| Disease | Algorithm | Reason |
|---------|-----------|--------|
| Diabetes | Random Forest | Non-linear BMI thresholds, mixed feature types, correlated features (age × BMI) |
| Heart Disease | Gradient Boosting | Sequential factor interactions (gender × age × smoking multiply risk), rare high-impact combinations |
| Obesity | Direct classification | BMI already provides a direct classification; other features layer future trajectory risk |
| Sleep Disorder | Logistic Regression | Monotonic linear relationships (more stress = worse sleep); interpretable coefficients needed |
| Symptoms | Weighted co-occurrence | No training data available; Bayesian-style scoring generalises well to unseen symptom combinations |

---

## 🗄️ Database Design

MongoDB with Mongoose ODM. 13 collections, all keyed by `userId` (ObjectId ref to `User`).

```
User
├── _id (ObjectId)
├── name, email, password (bcrypt)
├── lastLogin, onboardingComplete
└── createdAt

HealthProfile          (one per user, upsert)
├── userId → User
├── age, gender, height, weight, bmi, bloodGroup
└── dateOfBirth

LifestyleHabits        (one per user, upsert)
├── sleepDuration, sleepQuality
├── exerciseFrequency, exerciseDuration
├── waterIntake, stressLevel, screenTime
├── smokingHabit, alcoholConsumption
└── workHoursPerDay

FoodHabits             (one per user, upsert)
├── dietType, mealsPerDay
├── junkFoodFrequency, sugarIntake
└── oilConsumption

MedicalHistory         (one per user, upsert)
├── existingConditions: [String]
├── medications: [String]
├── allergies: [String]
└── pastSurgeries: [String]

FamilyHistory          (one per user, upsert)
├── diabetes, heartDisease, hypertension
├── cancer, thyroidDisorders
└── (all Boolean fields)

DailyLog               (one per user per date)
├── userId → User
├── date (indexed)
├── sleepHours, sleepQuality
├── waterIntake, caloriesConsumed
├── exerciseType, exerciseDuration, exerciseIntensity
├── moodRating, meals (breakfast/lunch/dinner/snacks)
└── symptoms: [{ name, severity }]

HealthScore            (one per user per date, auto-computed)
├── userId, date
├── sleepScore, dietScore, exerciseScore, symptomsScore
└── totalScore (0–100)

Prediction             (many per user, one per disease per run)
├── userId, type (disease name), date
├── riskLevel, riskScore, probability
├── factors: [{ name, impact, value }]
└── recommendations: [String]

DietPlan               (one per user per date)
├── userId, date, targetCalories
├── meals: { breakfast, morningSnack, lunch, eveningSnack, dinner }
│     each meal: { items: [String], calories, nutrients: { protein, carbs, fat, fiber } }
├── foodsToAvoid: [String]
├── healthTips: [String]
├── hydration (glasses/day), sodiumLimit (mg/day)
└── basedOn: { bmi, age, healthConditions, dietType, riskFactors }

Streak                 (one per user, upsert)
├── userId
├── currentStreak, longestStreak
├── exerciseStreak, longestExerciseStreak
├── totalHealthPoints
└── lastUpdated

NutritionAssessment    (many per user)
├── userId, date
├── symptoms: [String]
├── dietaryHabits: { dairy, meat, eggs, leafyGreens, ... }
├── lifestyleFactors: { sunExposure, dietType, alcohol, ... }
└── scores: { iron, vitaminD, vitaminB12, calcium, zinc, folate }
```

**Indexes:**
- `DailyLog`: `{ userId: 1, date: -1 }` — fast per-user date range queries
- `DietPlan`: `{ userId: 1, date: -1 }` — fast today's plan lookup
- `Prediction`: `{ userId: 1, createdAt: -1 }` — latest predictions first

---

## 🛠️ Tech Stack

### Frontend
| Technology | Version | Purpose |
|-----------|---------|---------|
| React | 18.2 | UI framework |
| Vite | 5.0 | Build tool & dev server |
| Tailwind CSS | 3.3 | Utility-first styling |
| React Router | v6 | Client-side routing with guards |
| Chart.js + react-chartjs-2 | 4.4 | Interactive health trend charts |
| Axios | 1.6 | HTTP client with JWT interceptor |
| react-hot-toast | 2.4 | Toast notifications |
| Web Speech API | native | Voice input/output in HealthBot |

### Backend (Node.js)
| Technology | Version | Purpose |
|-----------|---------|---------|
| Express | 4.18 | REST API framework |
| Mongoose | 9.2 | MongoDB ODM |
| MongoDB Atlas | — | Cloud database |
| jsonwebtoken | 9.0 | JWT sign and verify |
| bcryptjs | 2.4 | Password hashing |
| Nodemailer | 6.9 | Email via Gmail SMTP |
| node-cron | 3.0 | Scheduled daily jobs |
| Axios | 1.6 | Proxy calls to ML service |
| dotenv | 16.3 | Environment variable loading |

### ML Service (Python)
| Technology | Version | Purpose |
|-----------|---------|---------|
| Flask | ≥ 2.3.3 | REST microservice |
| flask-cors | ≥ 4.0.0 | CORS for Node proxy |
| scikit-learn | ≥ 1.4.0 | ML algorithms |
| NumPy | ≥ 1.26.0 | Numerical arrays |
| Pandas | ≥ 2.1.0 | Data manipulation |
| joblib | ≥ 1.3.2 | Model serialisation |

### External APIs
| Service | Usage |
|---------|-------|
| Google Gemini (`gemini-2.0-flash-latest`) | AI health chat + nutrition interpretation |
| Gmail SMTP | Automated email reminders |

---

## 📁 Project Structure

```
Health_pilot-main/
├── backend/
│   ├── server.js                     # App entry: dotenv → connectDB → Express → cron
│   ├── config/
│   │   └── db.js                     # MongoDB Atlas connection + event handlers
│   ├── controllers/
│   │   ├── authController.js         # register, login, getMe, changePassword
│   │   ├── profileController.js      # 5 profile upserts + getCompleteProfile
│   │   ├── dailyLogController.js     # CRUD + updateHealthScore + streak update
│   │   ├── predictionController.js   # calls ML service → stores Predictions
│   │   ├── dietController.js         # generateDietPlan + history + enrichDietPlan()
│   │   ├── analyticsController.js    # aggregations, trends, insights
│   │   ├── aiChatController.js       # Gemini proxy with user context injection
│   │   ├── nutritionController.js    # malnutrition scoring engine
│   │   └── reportController.js       # weekly/monthly report generation
│   ├── models/                       # 13 Mongoose schemas
│   ├── routes/                       # 9 route files (1:1 with controllers)
│   ├── middleware/
│   │   └── auth.js                   # JWT verification → 401 on failure
│   └── services/
│       ├── emailService.js           # HTML email templates via Nodemailer
│       └── scheduler.js             # node-cron jobs (9PM + 10AM IST)
│
├── frontend/
│   └── src/
│       ├── App.jsx                   # Routes + PrivateRoute + PublicRoute guards
│       ├── context/
│       │   └── AuthContext.jsx       # Global auth state + checkAuth on startup
│       ├── components/
│       │   ├── Layout.jsx            # Sidebar + topbar + theme toggle
│       │   ├── HealthChatbot.jsx     # Floating chatbot (11 intents + voice)
│       │   └── LoadingSpinner.jsx
│       ├── pages/
│       │   ├── Login.jsx / Register.jsx / Onboarding.jsx
│       │   ├── Dashboard.jsx / MainDashboard.jsx
│       │   ├── Profile.jsx
│       │   ├── DailyLog.jsx
│       │   ├── Predictions.jsx
│       │   ├── SymptomChecker.jsx
│       │   ├── DietPlan.jsx
│       │   ├── Analytics.jsx
│       │   ├── NutritionCheck.jsx
│       │   └── AIHealthChat.jsx
│       └── services/
│           └── api.js                # Axios instance + 401 interceptor + endpoint wrappers
│
└── ml-service/
    ├── app.py                        # Flask predictor engine (4 disease models + symptoms)
    └── requirements.txt
```

---

## 📡 API Reference

All routes except `/api/auth/*` require `Authorization: Bearer <token>` header.

### Auth
| Method | Endpoint | Body | Description |
|--------|----------|------|-------------|
| POST | `/api/auth/register` | `{name, email, password}` | Create account |
| POST | `/api/auth/login` | `{email, password}` | Get JWT |
| GET | `/api/auth/me` | — | Get current user |

### Profile
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET / POST | `/api/profile/health` | Health profile (height, weight, BMI) |
| GET / POST | `/api/profile/lifestyle` | Lifestyle habits |
| GET / POST | `/api/profile/food` | Food habits |
| GET / POST | `/api/profile/medical` | Medical history |
| GET / POST | `/api/profile/family` | Family history |
| GET | `/api/profile/complete` | All 5 sub-profiles in one call |

### Daily Logs
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/logs` | Logs for date range (up to 366 days) |
| GET | `/api/logs/today` | Today's log |
| POST | `/api/logs` | Create or update today's log |
| GET | `/api/logs/streak` | Current streaks + health points |

### Predictions
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/predictions/generate` | Run ML risk analysis (all 4 diseases) |
| GET | `/api/predictions/latest` | Most recent predictions |
| GET | `/api/predictions` | Full prediction history |

### Diet
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/diet/generate` | Generate personalised diet plan |
| GET | `/api/diet/today` | Today's plan with computed macros |
| GET | `/api/diet/history` | Past diet plans |

### Analytics
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/analytics/summary` | Averages + insights |
| GET | `/api/analytics/trends` | Chart data (7/30/90 days) |
| GET | `/api/analytics/report` | Weekly summary report |

### Nutrition
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/nutrition/assess` | Submit 3-step assessment, get nutrient scores |
| GET | `/api/nutrition/latest` | Most recent assessment |
| GET | `/api/nutrition/history` | Assessment history |

### AI
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/ai/chat` | Gemini AI chat (user context injected server-side) |
| POST | `/api/ai/analyze-nutrition` | Send assessment to Gemini for deep analysis |

---

## 🔒 Security

| Concern | Implementation |
|---------|---------------|
| Authentication | JWT signed with `JWT_SECRET`, 7-day expiry |
| Password storage | bcrypt, 10 salt rounds — never stored in plaintext |
| Authorisation | `userId` always taken from verified JWT payload — never from request body (prevents IDOR) |
| Route protection | All non-auth routes return `401 Unauthorized` if token missing or invalid |
| Secrets | `.env` file excluded from git via `.gitignore` |
| Database | MongoDB Atlas with connection string in env — not hardcoded |
| Input validation | `express-validator` on auth routes |

---

## ⚙️ Setup & Installation

### Prerequisites
- Node.js 18+
- Python 3.9+
- MongoDB Atlas account (or local MongoDB)
- Google Gemini API key
- Gmail account with App Password

### 1. Clone
```bash
git clone <repo-url>
cd Health_pilot-main
```

### 2. Backend
```bash
cd backend
npm install
```

Create `backend/.env`:
```env
MONGODB_URI=mongodb+srv://<user>:<pass>@cluster.mongodb.net/healthDB?retryWrites=true&w=majority
JWT_SECRET=your_jwt_secret_32chars_minimum
PORT=5000
ML_SERVICE_URL=http://localhost:5001
EMAIL_USER=your_gmail@gmail.com
EMAIL_PASS=your_16char_app_password
GEMINI_API_KEY=your_gemini_key
```

```bash
npm run dev        # starts with nodemon on port 5000
```

### 3. Frontend
```bash
cd frontend
npm install
npm run dev        # starts Vite on port 5173
```

### 4. ML Service
```bash
cd ml-service
pip install -r requirements.txt
python app.py      # starts Flask on port 5001
```

### 5. Gmail App Password
1. Google Account → Security → 2-Step Verification (enable)
2. Search "App passwords" → create one for Mail
3. Paste the 16-character password as `EMAIL_PASS`

---

## 🏆 What Makes NutriCare Stand Out

| Feature | Why It's Unique |
|---------|----------------|
| Hidden malnutrition detection | Detects subclinical deficiencies (iron, B12, Vit D, zinc) invisible to BMI or weight metrics — a problem affecting 2+ billion people globally |
| Three-tier AI system | Rule-based chatbot (instant) + ML predictor (calculated) + Gemini AI (conversational) — three different intelligence layers for three different needs |
| Context-aware AI | Gemini receives real user data (BMI, diet type, stress, exercise) on every request — not generic health advice |
| All-calculations, no hardcoded data | Every number (hydration, sodium, calories, macros, risk score) is computed from the user's actual profile — no preset defaults in responses |
| Voice-enabled chatbot | Web Speech API voice input + TTS output makes the bot accessible without typing |
| Production-grade connection handling | MongoDB Atlas reconnection events are handled with proper error listeners — prevents the silent crash on idle connection timeout |
| Disease-specific algorithm selection | Three different ML algorithms (Random Forest, Gradient Boosting, Logistic Regression) chosen per disease based on the specific nature of its risk factor relationships |
