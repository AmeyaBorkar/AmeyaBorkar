# Personal Health Assistant

> Flask + React wellness app: log meals, track macros, receive personalized coaching from the Cerebras LLM, and get AI-generated weekly summaries.

**Repository:** [`AmeyaBorkar/PersonalHealthAssistant`](https://github.com/AmeyaBorkar/PersonalHealthAssistant)  
**Category:** Web App / AI  
**Visibility:** Public  
**Primary language:** Python  
**Default branch:** `main`  
**License:** MIT License  
**Created:** 2026-03-02  
**Last pushed:** 2026-03-02  
**Metadata updated:** 2026-03-02  
**Size (GitHub reported):** 71 KB  

---

## What it is (one-paragraph version)

Flask + React wellness app: log meals, track macros, receive personalized coaching from the Cerebras LLM, and get AI-generated weekly summaries.

## Language breakdown

| Language | Bytes | Share |
|----------|------:|------:|
| Python | 91,889 | 58.2% |
| JavaScript | 53,133 | 33.7% |
| CSS | 12,130 | 7.7% |
| HTML | 615 | 0.4% |

## File tree

- Total entries indexed: **46** (40 files, 6 directories)

```
.env.example  (666 B)
.gitignore  (305 B)
LICENSE  (1 KB)
README.md  (5 KB)
app.py  (3 KB)
cerebras_service.py  (21 KB)
config.py  (13 KB)
database.py  (12 KB)
requirements.txt  (159 B)
run.py  (588 B)
utils.py  (8 KB)
frontend/    [21 files]
  frontend/.gitignore
  frontend/README.md
  frontend/eslint.config.js
  frontend/index.html
  frontend/package-lock.json
  frontend/package.json
  frontend/public/vite.svg
  frontend/src/App.jsx
  frontend/src/AuthContext.jsx
  frontend/src/api.js
  frontend/src/assets/react.svg
  frontend/src/index.css
  frontend/src/main.jsx
  frontend/src/pages/Chat.jsx
  frontend/src/pages/Dashboard.jsx
  ... and 6 more under frontend/
routes/    [8 files]
  routes/__init__.py
  routes/ai.py
  routes/analytics.py
  routes/auth.py
  routes/goals.py
  routes/meals.py
  routes/tracking.py
  routes/user.py
```

## README (verbatim)

# 🏋️ Personal Health Assistant

An AI-powered personal health and fitness assistant with nutrition tracking, goal setting, and intelligent wellness coaching — built with **Flask** (Python) and **React** (Vite).

![License](https://img.shields.io/badge/license-MIT-blue.svg)
![Python](https://img.shields.io/badge/python-3.9+-blue.svg)
![React](https://img.shields.io/badge/react-19-61DAFB.svg)

## ✨ Features

| Feature | Description |
|---------|-------------|
| 🤖 **AI Nutrition Analysis** | Automatically extracts calories, protein, carbs, and fats from food descriptions using LLM |
| 💬 **AI Health Chat** | Personalized fitness & nutrition advice with full user context |
| 🍽️ **Meal Tracking** | Log meals with AI-generated feedback on how each fits your daily goals |
| 🎯 **Goal Setting** | Set weight loss, muscle gain, or maintenance goals with auto-calculated macro targets |
| 💧 **Water Tracking** | Track daily water intake with progress visualization |
| 📊 **Dashboard** | Real-time daily progress with calorie/macro breakdown and streak tracking |
| 🏆 **Achievements** | Earn badges for consistency and milestones |
| 📈 **Weekly Summaries** | AI-generated weekly reports with personalized recommendations |

## 🏗️ Architecture

```
PersonalHealthAssistant/
├── app.py                 # Flask app factory (Blueprints)
├── run.py                 # Entry point
├── config.py              # Environment-based configuration
├── database.py            # SQLAlchemy models (SQLite/MySQL)
├── cerebras_service.py    # LLM integration (Cerebras)
├── utils.py               # Helpers (auth, calculations, validation)
├── routes/                # Modular API blueprints
│   ├── auth.py            #   Registration, login, JWT
│   ├── user.py            #   Profile management
│   ├── meals.py           #   Meal logging & tracking
│   ├── ai.py              #   AI chat & summaries
│   ├── goals.py           #   Goal setting & progress
│   ├── tracking.py        #   Water & achievements
│   └── analytics.py       #   Dashboard data
├── frontend/              # React + Vite SPA
│   ├── src/
│   │   ├── pages/         #   Dashboard, Meals, Chat, Goals, Profile
│   │   ├── api.js         #   API client
│   │   ├── AuthContext.jsx #   Auth state management
│   │   └── App.jsx        #   Routing & layout
│   └── index.html
├── .env.example           # Environment variables template
├── requirements.txt       # Python dependencies
└── README.md
```

## 🚀 Quick Start

### Prerequisites
- Python 3.9+
- Node.js 18+
- A [Cerebras](https://cloud.cerebras.ai/) API key

### 1. Clone & configure

```bash
git clone https://github.com/AmeyaBorkar/PersonalHealthAssistant.git
cd PersonalHealthAssistant

# Copy environment template and add your API key
cp .env.example .env
# Edit .env → set CEREBRAS_API_KEY
```

### 2. Backend

```bash
pip install -r requirements.txt
python run.py
# → Server starts at http://localhost:8000
# → SQLite database auto-created (no MySQL needed!)
```

### 3. Frontend

```bash
cd frontend
npm install
npm run dev
# → Opens at http://localhost:5173
```

### 4. Use it

1. Open http://localhost:5173
2. Create an account
3. Set a fitness goal
4. Start logging meals — AI will analyze nutrition and give feedback
5. Chat with the AI assistant about diet, workouts, or supplements

## 🔌 API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/api/v1/auth/register` | Register new user |
| `POST` | `/api/v1/auth/login` | Login |
| `GET` | `/api/v1/user/profile` | Get profile |
| `PUT` | `/api/v1/user/profile` | Update profile |
| `POST` | `/api/v1/meals/log` | Log meal (AI nutrition extraction) |
| `GET` | `/api/v1/meals/today` | Today's meals + progress |
| `GET` | `/api/v1/meals/week` | Weekly breakdown |
| `POST` | `/api/v1/ai/chat` | Chat with AI assistant |
| `GET` | `/api/v1/ai/weekly-summary` | AI weekly report |
| `POST` | `/api/v1/goals/set` | Set fitness goal |
| `GET` | `/api/v1/goals/current` | Current goal + progress |
| `POST` | `/api/v1/tracking/water` | Log water intake |
| `GET` | `/api/v1/analytics/dashboard` | Dashboard data |

All authenticated endpoints require `Authorization: Bearer <token>` header.

## 🛠️ Tech Stack

**Backend:** Flask, SQLAlchemy, Flask-JWT-Extended, bcrypt, Cerebras LLM SDK

**Frontend:** React 19, Vite, React Router, Vanilla CSS (dark glassmorphism)

**Database:** SQLite (default, zero setup) or MySQL (production)

## 📄 License

MIT License — see [LICENSE](LICENSE) for details.

---

*Generated 2026-05-02 from GitHub API. Source of truth: https://github.com/AmeyaBorkar/PersonalHealthAssistant*
