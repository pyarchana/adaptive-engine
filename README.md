# Adaptive Diagnostic Engine

A **1-Dimension Adaptive Testing** system built for the Ed-Tech internship assignment. 
(Note: Take Home assignment that explicitly required integrating AI tools into the workflow)
Dynamically selects GRE-style questions based on a student's real-time estimated ability using
**Item Response Theory (Rasch Model)** and generates a personalized **AI-powered study plan**
via the Anthropic Claude API.

---

## Getting Started

### 1. Clone & install dependencies
```bash
git clone https://github.com/pyarchana/adaptive-engine.git
cd adaptive-engine
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements.txt
```

### 2. Configure environment
```bash
cp .env.example .env
# Edit .env and add your MONGO_URI and ANTHROPIC_API_KEY
```

| Variable | Where to get it |
|---|---|
| `MONGO_URI` | [MongoDB Atlas](https://cloud.mongodb.com) → Connect → Drivers |
| `ANTHROPIC_API_KEY` | [console.anthropic.com](https://console.anthropic.com) |

### 3. Seed the database
```bash
python scripts/seed.py
# ✅ Seeded 20 questions into 'adaptive_engine.questions'
```

### 4. Run the server
```bash
uvicorn app.main:app --reload
# Server running at http://localhost:8000
# Interactive docs at http://localhost:8000/docs
```

---

## API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/api/session/start?student_name=Alice` | Start a new adaptive session |
| `POST` | `/api/submit-answer` | Submit an answer, get next question + updated ability |
| `GET` | `/api/session/{session_id}` | Retrieve full session state |
| `GET` | `/api/next-question/{session_id}` | Get the current unanswered question |
| `GET` | `/api/insights/{session_id}` | Get AI-generated study plan (session must be complete) |
| `GET` | `/api/questions` | List all questions (admin/debug) |

### Example flow
```bash
# 1. Start session
curl -X POST "http://localhost:8000/api/session/start?student_name=Alice"
# → Returns session_id and first question

# 2. Submit answer
curl -X POST "http://localhost:8000/api/submit-answer" \
  -H "Content-Type: application/json" \
  -d '{"session_id": "<id>", "question_id": "<id>", "answer": "B"}'
# → Returns is_correct, updated_ability_score, next_question

# 3. After 10 questions, get study plan
curl "http://localhost:8000/api/insights/<session_id>"
# → Returns AI-generated 3-step personalized study plan
```

---

## Adaptive Algorithm — Rasch Model (IRT)

### The Core Formula

The **Rasch Model** (1-Parameter IRT) estimates the probability that a student with
ability **θ** (theta) answers a question of difficulty **b** correctly:

```
P(correct | θ, b) = exp(θ - b) / (1 + exp(θ - b))
```

### How Ability Updates

After every response, we update the ability estimate using a gradient step:

```
θ_new = θ_old + η × (observed - expected)

where:
  observed = 1 if correct, 0 if incorrect
  expected = P(correct | θ_old, b)    ← Rasch probability above
  η        = 0.5 (learning rate)
```

This is a simplified **MLE (Maximum Likelihood Estimation)** update — the same
mathematical foundation used in real adaptive testing systems like the GRE.

### Question Selection

After each update, the system targets the next question at difficulty ≈ θ
(the student's current estimated ability). This places the student at the
"zone of proximal development" — hard enough to be informative, not so hard
as to be discouraging.

### Why Not Just ±0.1?

A naive approach (if correct → difficulty + 0.1) ignores *how* correct.
A student with θ = 0.9 answering a difficulty-0.1 question correctly gives
almost zero new information. The Rasch model naturally weights responses by
how surprising they are.

---

## AI Log (How I Used AI Tools)

### What AI helped with:
- **Boilerplate generation**: FastAPI route structure, Pydantic model schemas, and
  Motor async patterns were generated in Cursor with targeted prompts, saving ~2 hours.
- **IRT implementation**: Used Claude to verify the Rasch probability formula and
  sanity-check the logit ↔ probability mapping functions.
- **Question dataset**: Used Claude to generate 20 diverse GRE-style questions
  spanning Algebra, Vocabulary, Geometry, and Data Analysis, then manually verified
  each question and difficulty calibration.
- **LLM prompt engineering**: Iterated on the study-plan prompt with Claude to ensure
  the output was specific, actionable, and tied to the student's actual weak topics.

### What AI couldn't solve:
- **Difficulty band tuning**: The ±0.25 difficulty window for question selection required
  manual testing to prevent the algorithm from getting stuck at the extremes.
- **Motor async lifecycle**: FastAPI's `on_event("startup")` with Motor's async client
  had a subtle scoping issue (global variable not accessible in route functions) that
  required reading Motor's docs directly.
- **Edge case in IRT**: When ability approaches 0.95 or 0.05, the logit conversion
  becomes numerically unstable. Claude suggested clamping, but determining the right
  clamp values required empirical testing.

---

## Project Structure

```
adaptive-engine/
├── app/
│   ├── main.py              # FastAPI app & middleware
│   ├── core/
│   │   └── database.py      # MongoDB async connection (Motor)
│   ├── models/
│   │   └── schemas.py       # Pydantic models (Question, Session, etc.)
│   ├── routes/
│   │   ├── questions.py     # GET /questions, GET /next-question
│   │   ├── sessions.py      # POST /session/start, POST /submit-answer
│   │   └── insights.py      # GET /insights (AI study plan)
│   └── services/
│       ├── irt.py           # Rasch Model — ability update & question selection
│       └── llm.py           # Anthropic API — study plan generation
├── scripts/
│   └── seed.py              # Seeds 20 GRE questions into MongoDB
├── requirements.txt
├── .env.example
└── README.md
```
