# AIRS Session Continuation — Phase 9.5 In Progress

## Date
Session ended: 2026-09-14 (evening, local time)
Session starts: 2026-09-15 (whenever you begin)

## Repo state
- Branch: main
- Last commit: docs: remove stale known-issue row from LIMITATIONS
- Working tree: clean
- Local == origin/main
- 10 commits pushed today (see "Today's commits" below)

## Today's commits (2026-09-14)

1. chore(deps): pin runtime dependencies from actual imports
2. chore: ignore one-off test script in gitignore
3. fix(llm): resolve Ollama model config drift, default to qwen2.5:7b
4a. fix(hypothesis): stop using beta as a directional signal
4b. feat(risk): surface market beta as a risk dimension
4c. docs: document magnitude vs direction separation (Decision 037)
--. added tomorrow.md, todo list for tomorrow in gitignore
4d. docs: unify version markers at v0.3.9
4e. docs: remove stale known-issue row from LIMITATIONS

Current declared version: v0.3.9

## What Phase 9 revealed (the audit)

Ran `python main.py --audit` on 2026-09-14 against 30 sessions
(all dated 2026-08-14, day 31 of the 30-day window). 23 scored,
7 skipped (startups, no ticker).

Overall average score: -0.09 (roughly coin-flip with slight negative edge)

By Uncertainty Level (THE KEY FINDING):
  Low       | 7 sessions  | avg -0.4431   <- WORST
  Moderate  | 15 sessions | avg +0.0815   <- BEST
  Elevated  | 1 session   | avg -0.2961

By Evidence Strength:
  Convicted | 23 sessions | avg -0.0945   <- ALL in one bucket, thresholds too low

By Sector (partial):
  cloud-infrastructure | 3 | +0.61
  cybersecurity        | 1 | +1.00
  consumer-tech        | 4 | -0.80
  l1-blockchain        | 3 | -1.00   <- BTC/ETH/SOL all called bearish, all rallied 22-32%
  ev-energy            | 1 | -0.98
  semiconductors       | 1 | -1.00

The Low-uncertainty inversion is the loudest signal in the data.
Hypothesis: same root cause as the magnitude/direction mixing bug.

## What we fixed today

Bug class identified: magnitude metrics were mixed with direction
metrics in the Hypothesis Engine's bullish/bearish claim buckets.

Direction metrics (should influence direction):
  trend, momentum, MACD, RSI extremes, returns, news signals

Magnitude metrics (should influence risk only):
  beta, risk_score, drawdown, volatility_regime, volatility, ATR

Fixed so far:
  - beta removed from hypothesis.py _assess_evidence()
  - beta added to agents/risk.py with thresholds:
      beta > 1.8 -> high risk
      beta > 1.3 -> medium warning
      beta < 0.5 -> medium warning (opportunity cost)
      0.5-1.3   -> no claim

Remaining in wrong buckets (Phase 9.5 priority 1):
  - risk_score    (currently bearish, strength 0.75)
  - drawdown      (currently bearish, strength 0.50)
  - volatility_regime (currently bearish, strength 0.45)
  - volatility    (currently neutral, strength 0.30)

These need the same treatment as beta.

## Priority 1 — Finish the magnitude/direction separation

Files: reports/hypothesis.py, agents/risk.py

Steps:
1. Read reports/hypothesis.py _assess_evidence()
2. Identify every magnitude metric still in bullish/bearish/neutral buckets
3. Remove each from _assess_evidence()
4. Add corresponding rules to agents/risk.py
5. Commit each metric separately (one commit per metric for clean history)

Expected commits:
  fix(hypothesis): stop using risk_score as a directional signal
  fix(hypothesis): stop using drawdown as a directional signal
  fix(hypothesis): stop using volatility_regime as a directional signal
  fix(hypothesis): stop using volatility as a directional signal
  feat(risk): add drawdown/volatility/risk_score risk rules

Test after each: python main.py --entity NVIDIA --ticker NVDA
Confirm the claim disappears from bull/bear case but appears in Risk.

## Priority 2 — Fix the Low-uncertainty inversion

After Priority 1 is complete and tested:
1. Re-run: python main.py --audit --force
2. Compare new "By Uncertainty Level" grouping against today's
3. If inversion persists, examine reports/hypothesis.py _compute_uncertainty()
4. Likely recalibration targets:
   - Scarcity weight (currently 0.35 max)
   - Conflict weight (currently 0.40 max)
   - Coverage weight (currently 0.25 max)
   - Level thresholds (Low <0.20, Moderate <0.40, etc.)

Do NOT re-run --audit --force until Priority 1 is committed and tested.

## Priority 3 — Re-bucket evidence strength

All 23 sessions landed in "Convicted". Current thresholds in data/audit.py:
  Speculative: total < 0.5
  Tentative:   0.5 <= total < 1.5
  Convicted:   total >= 1.5

Need to examine actual (bull_strength + bear_strength) distribution:
  python -c "import sqlite3; c=sqlite3.connect('airs.db'); c.row_factory=sqlite3.Row; [print(dict(r)) for r in c.execute('SELECT id, entity, bull_strength, bear_strength, (bull_strength+bear_strength) as total FROM research_sessions WHERE ticker IS NOT NULL ORDER BY total')]"

Recalibrate thresholds based on real distribution.
Files: data/audit.py compute_evidence_strength()

## Priority 4 — Document the baseline

Create docs/research/PHASE_9_BASELINE.md
Record:
  - 30 sessions, 23 scored, overall avg -0.09
  - By uncertainty, evidence strength, sector breakdown
  - Note it is the pre-Phase-9.5 baseline
  - Future audits will differ after calibration fixes

## Priority 5 (defer) — Business Agent non-determinism

Ran NVIDIA twice ~10 min apart on 2026-09-15:
  Run 1: BULLISH (net +0.18)
  Run 2: NEUTRAL (net +0.10)
Cause: live RSS feeds return different news between runs
Impact: same entity + same day -> different bias
Not urgent. Flag for Phase 10.

## Priority 6 (cosmetic) — Windows UTF-8 mojibake

Em dashes render as ΓÇö in PowerShell console output on Windows 10.
Likely console encoding, not file encoding. Verify with:
  python -c "print(open('reports/output/<latest>.md', encoding='utf-8').read()[500:600])"
Low priority.

## Key facts for next session

- Python 3.13.1
- DB file: airs.db at repo root (gitignored)
- Backup: airs.db.backup_20260914_193314 (gitignored)
- research_sessions has 30 rows (ids 7-36, gap from 1-6 deleted)
- research_outcomes has 23 rows (from today's --audit)
- LLM: Ollama qwen2.5:7b (localhost:11434), qwen3:4b also pulled but returns empty on JSON extraction prompts
- Hardware: GTX 1060 6GB, 16GB RAM, Windows 10
- Venv: activated in the PowerShell prompt as (venv)

## Design principles still in force

- Critic is 100% rule-based. No LLM halt decisions.
- Evidence Register is single source of truth.
- Financial analysis is deterministic. No LLM for math.
- Uncertainty is independent of directional conviction.
- Historical sessions (ids 7-36) must NOT be rewritten.
- Every output traceable to evidence.

## User workflow preferences

- Commit after every meaningful change (GitHub activity graph is a portfolio goal)
- Commit messages must be substantive, not "update" or "fix"
- NO git add . — always name files explicitly
- NO force-push, NO rewriting pushed history
- Small, atomic commits preferred over large batched ones
- Ask before multi-file refactors
- User values honest technical feedback over politeness
- User is building AIRS as a portfolio piece for job applications

## Recovery checklist for tomorrow

1. Run: git log --oneline -5
2. Run: git status
3. Confirm clean tree, synced with origin
4. Paste reports/hypothesis.py _assess_evidence() function
5. Start Priority 1: remove risk_score from directional claims

## First action tomorrow

Run:
  git log --oneline -5
  git status

Paste both. Confirm tree is clean and last commit is 4e
(remove stale known-issue row from LIMITATIONS). Then paste
reports/hypothesis.py so we can start Priority 1.

Do not start coding until I have confirmed the state.<img src="https://capsule-render.vercel.app/api?type=venom&height=220&color=0:000000,100:a371f7&text=Angad%20Khanal&fontSize=60&fontColor=FFFFFF&animation=fadeIn&fontAlignY=40&desc=Machine%20Learning%20Engineer%20%7C%20Data%20Analyst%20&descSize=22&descColor=FFFFFF&descAlignY=65" width="100%"/>

<p align="center">
  <a href="https://komarev.com/ghpvc/?username=deadpool-kha">
    <img src="https://komarev.com/ghpvc/?username=deadpool-kha&label=Profile%20views&color=00FFFF&style=flat-square" alt="deadpool-kha's profile views" />
  </a>
</p>

<img src="https://i.pinimg.com/originals/42/b4/22/42b4229a9ec3145edaa895b2415dd720.gif" alt="Banner" width="100%" />

## 📌 About Me
- 💻 Data Analyst & AI Engineer
- 🤖 Building ML, NLP & LLM-powered applications
- 📊 Turning messy data into actionable insights
- 🧠 Passionate about AI, Data Systems & Automation
- ⚙️ Python • SQL • Power BI • Scikit-learn
- 🚀 Exploring Agentic AI & End-to-End ML Pipelines
- 🌱 Always learning and building


## 🧠 My Focus Areas
- Artificial Intelligence
- Machine Learning
- Large Language Models (LLMs)
- Natural Language Processing (NLP)
- Data Analytics
- Data Engineering
- SQL & Database Systems
- ETL Pipelines
- Business Intelligence
- MLOps & ML Pipelines
- AI Agents & Automation


## 📊 GitHub Stats & Trophies
<p align="center">
  <a href="https://github.com/deadpool-kha">
    <img height="180em" src="https://github-readme-stats-eight-theta.vercel.app/api?username=deadpool-kha&cache_seconds=7200&layout=compact&theme=nightowl&border_radius=10" alt="deadpool-kha's GitHub Stats" />
  </a>
</p>
# AIRS Session Continuation — Phase 9.5 In Progress

## Date
Session ended: 2026-09-14 (evening, local time)
Session starts: 2026-09-15 (whenever you begin)

## Repo state
- Branch: main
- Last commit: docs: remove stale known-issue row from LIMITATIONS
- Working tree: clean
- Local == origin/main
- 10 commits pushed today (see "Today's commits" below)

## Today's commits (2026-09-14)

1. chore(deps): pin runtime dependencies from actual imports
2. chore: ignore one-off test script in gitignore
3. fix(llm): resolve Ollama model config drift, default to qwen2.5:7b
4a. fix(hypothesis): stop using beta as a directional signal
4b. feat(risk): surface market beta as a risk dimension
4c. docs: document magnitude vs direction separation (Decision 037)
--. added tomorrow.md, todo list for tomorrow in gitignore
4d. docs: unify version markers at v0.3.9
4e. docs: remove stale known-issue row from LIMITATIONS

Current declared version: v0.3.9

## What Phase 9 revealed (the audit)

Ran `python main.py --audit` on 2026-09-14 against 30 sessions
(all dated 2026-08-14, day 31 of the 30-day window). 23 scored,
7 skipped (startups, no ticker).

Overall average score: -0.09 (roughly coin-flip with slight negative edge)

By Uncertainty Level (THE KEY FINDING):
  Low       | 7 sessions  | avg -0.4431   <- WORST
  Moderate  | 15 sessions | avg +0.0815   <- BEST
  Elevated  | 1 session   | avg -0.2961

By Evidence Strength:
  Convicted | 23 sessions | avg -0.0945   <- ALL in one bucket, thresholds too low

By Sector (partial):
  cloud-infrastructure | 3 | +0.61
  cybersecurity        | 1 | +1.00
  consumer-tech        | 4 | -0.80
  l1-blockchain        | 3 | -1.00   <- BTC/ETH/SOL all called bearish, all rallied 22-32%
  ev-energy            | 1 | -0.98
  semiconductors       | 1 | -1.00

The Low-uncertainty inversion is the loudest signal in the data.
Hypothesis: same root cause as the magnitude/direction mixing bug.

## What we fixed today

Bug class identified: magnitude metrics were mixed with direction
metrics in the Hypothesis Engine's bullish/bearish claim buckets.

Direction metrics (should influence direction):
  trend, momentum, MACD, RSI extremes, returns, news signals

Magnitude metrics (should influence risk only):
  beta, risk_score, drawdown, volatility_regime, volatility, ATR

Fixed so far:
  - beta removed from hypothesis.py _assess_evidence()
  - beta added to agents/risk.py with thresholds:
      beta > 1.8 -> high risk
      beta > 1.3 -> medium warning
      beta < 0.5 -> medium warning (opportunity cost)
      0.5-1.3   -> no claim

Remaining in wrong buckets (Phase 9.5 priority 1):
  - risk_score    (currently bearish, strength 0.75)
  - drawdown      (currently bearish, strength 0.50)
  - volatility_regime (currently bearish, strength 0.45)
  - volatility    (currently neutral, strength 0.30)

These need the same treatment as beta.

## Priority 1 — Finish the magnitude/direction separation

Files: reports/hypothesis.py, agents/risk.py

Steps:
1. Read reports/hypothesis.py _assess_evidence()
2. Identify every magnitude metric still in bullish/bearish/neutral buckets
3. Remove each from _assess_evidence()
4. Add corresponding rules to agents/risk.py
5. Commit each metric separately (one commit per metric for clean history)

Expected commits:
  fix(hypothesis): stop using risk_score as a directional signal
  fix(hypothesis): stop using drawdown as a directional signal
  fix(hypothesis): stop using volatility_regime as a directional signal
  fix(hypothesis): stop using volatility as a directional signal
  feat(risk): add drawdown/volatility/risk_score risk rules

Test after each: python main.py --entity NVIDIA --ticker NVDA
Confirm the claim disappears from bull/bear case but appears in Risk.

## Priority 2 — Fix the Low-uncertainty inversion

After Priority 1 is complete and tested:
1. Re-run: python main.py --audit --force
2. Compare new "By Uncertainty Level" grouping against today's
3. If inversion persists, examine reports/hypothesis.py _compute_uncertainty()
4. Likely recalibration targets:
   - Scarcity weight (currently 0.35 max)
   - Conflict weight (currently 0.40 max)
   - Coverage weight (currently 0.25 max)
   - Level thresholds (Low <0.20, Moderate <0.40, etc.)

Do NOT re-run --audit --force until Priority 1 is committed and tested.

## Priority 3 — Re-bucket evidence strength

All 23 sessions landed in "Convicted". Current thresholds in data/audit.py:
  Speculative: total < 0.5
  Tentative:   0.5 <= total < 1.5
  Convicted:   total >= 1.5

Need to examine actual (bull_strength + bear_strength) distribution:
  python -c "import sqlite3; c=sqlite3.connect('airs.db'); c.row_factory=sqlite3.Row; [print(dict(r)) for r in c.execute('SELECT id, entity, bull_strength, bear_strength, (bull_strength+bear_strength) as total FROM research_sessions WHERE ticker IS NOT NULL ORDER BY total')]"

Recalibrate thresholds based on real distribution.
Files: data/audit.py compute_evidence_strength()

## Priority 4 — Document the baseline

Create docs/research/PHASE_9_BASELINE.md
Record:
  - 30 sessions, 23 scored, overall avg -0.09
  - By uncertainty, evidence strength, sector breakdown
  - Note it is the pre-Phase-9.5 baseline
  - Future audits will differ after calibration fixes

## Priority 5 (defer) — Business Agent non-determinism

Ran NVIDIA twice ~10 min apart on 2026-09-15:
  Run 1: BULLISH (net +0.18)
  Run 2: NEUTRAL (net +0.10)
Cause: live RSS feeds return different news between runs
Impact: same entity + same day -> different bias
Not urgent. Flag for Phase 10.

## Priority 6 (cosmetic) — Windows UTF-8 mojibake

Em dashes render as ΓÇö in PowerShell console output on Windows 10.
Likely console encoding, not file encoding. Verify with:
  python -c "print(open('reports/output/<latest>.md', encoding='utf-8').read()[500:600])"
Low priority.

## Key facts for next session

- Python 3.13.1
- DB file: airs.db at repo root (gitignored)
- Backup: airs.db.backup_20260914_193314 (gitignored)
- research_sessions has 30 rows (ids 7-36, gap from 1-6 deleted)
- research_outcomes has 23 rows (from today's --audit)
- LLM: Ollama qwen2.5:7b (localhost:11434), qwen3:4b also pulled but returns empty on JSON extraction prompts
- Hardware: GTX 1060 6GB, 16GB RAM, Windows 10
- Venv: activated in the PowerShell prompt as (venv)

## Design principles still in force

- Critic is 100% rule-based. No LLM halt decisions.
- Evidence Register is single source of truth.
- Financial analysis is deterministic. No LLM for math.
- Uncertainty is independent of directional conviction.
- Historical sessions (ids 7-36) must NOT be rewritten.
- Every output traceable to evidence.

## User workflow preferences

- Commit after every meaningful change (GitHub activity graph is a portfolio goal)
- Commit messages must be substantive, not "update" or "fix"
- NO git add . — always name files explicitly
- NO force-push, NO rewriting pushed history
- Small, atomic commits preferred over large batched ones
- Ask before multi-file refactors
- User values honest technical feedback over politeness
- User is building AIRS as a portfolio piece for job applications

## Recovery checklist for tomorrow

1. Run: git log --oneline -5
2. Run: git status
3. Confirm clean tree, synced with origin
4. Paste reports/hypothesis.py _assess_evidence() function
5. Start Priority 1: remove risk_score from directional claims

## First action tomorrow

Run:
  git log --oneline -5
  git status

Paste both. Confirm tree is clean and last commit is 4e
(remove stale known-issue row from LIMITATIONS). Then paste
reports/hypothesis.py so we can start Priority 1.

Do not start coding until I have confirmed the state.


## 🛠️ Languages & Tools

<h3 align="center">Programming Languages</h3>
<p align="center">
  <img src="https://raw.githubusercontent.com/devicons/devicon/master/icons/javascript/javascript-original.svg" alt="JavaScript" width="40" />&nbsp;&nbsp;
  <img src="https://raw.githubusercontent.com/devicons/devicon/master/icons/python/python-original.svg" alt="Python" width="40" />&nbsp;&nbsp;
  <img src="https://raw.githubusercontent.com/devicons/devicon/master/icons/cplusplus/cplusplus-original.svg" alt="C++" width="40" />

</p>

<h3 align="center">Frontend</h3>
<p align="center">
  <img src="https://raw.githubusercontent.com/devicons/devicon/master/icons/react/react-original.svg" alt="React" width="40" />

</p>

<h3 align="center">Backend</h3>
<p align="center">
  <img src="https://raw.githubusercontent.com/devicons/devicon/master/icons/nodejs/nodejs-original.svg" alt="Node.js" width="40" />&nbsp;&nbsp;
  <img src="https://cdn.worldvectorlogo.com/logos/django.svg" alt="Django" width="40" />&nbsp;&nbsp;
  <img src="https://www.vectorlogo.zone/logos/palletsprojects_flask/palletsprojects_flask-ar21.svg" alt="Flask" width="40" />

</p>

<h3 align="center">Database</h3>
<p align="center">
  <img src="https://raw.githubusercontent.com/devicons/devicon/master/icons/mysql/mysql-original.svg" alt="MySQL" width="40" />&nbsp;&nbsp;
  <img src="https://raw.githubusercontent.com/devicons/devicon/master/icons/postgresql/postgresql-original.svg" alt="PostgreSQL" width="40" />

</p>

<h3 align="center">DevOps & Cloud</h3>
<p align="center">
  <img src="https://www.vectorlogo.zone/logos/kubernetes/kubernetes-icon.svg" alt="Kubernetes" width="40" />&nbsp;&nbsp;
  <img src="https://www.vectorlogo.zone/logos/amazon_aws/amazon_aws-icon.svg" alt="AWS" width="40" />

</p>

<h3 align="center">Tools</h3>
<p align="center">
  <img src="https://www.vectorlogo.zone/logos/git-scm/git-scm-icon.svg" alt="Git" width="40" />&nbsp;&nbsp;
  <img src="https://www.vectorlogo.zone/logos/visualstudio_code/visualstudio_code-icon.svg" alt="VS Code" width="40" />&nbsp;&nbsp;
  <img src="https://www.vectorlogo.zone/logos/getpostman/getpostman-icon.svg" alt="Postman" width="40" />

</p>

<p align="center">
  <a href="https://github.com/deadpool-kha">
    <img height="180em" src="https://github-readme-stats-eight-theta.vercel.app/api/top-langs/?username=deadpool-kha&langs_count=8&layout=compact&theme=nightowl&border_radius=10" alt="Top Languages" />
  </a>
</p>

## 🔗 Connect with Me
<p align="center">
  <a href="https://www.linkedin.com/in/angadkhanal/" target="_blank">
    <img align="center" src="https://img.shields.io/badge/LinkedIn-%230077B5.svg?style=for-the-badge&logo=linkedin&logoColor=white&color=00FFFF" alt="Angad Khanal's LinkedIn"/>
  </a>&nbsp;&nbsp;
  
  <a href="mailto:khanalak07@gmail.com">
    <img align="center" src="https://img.shields.io/badge/Gmail-D14836?style=for-the-badge&logo=gmail&logoColor=white&color=00FFFF" alt="Angad Khanal's Email"/>
  </a>
</p>

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/tobiasmeyhoefer/tobiasmeyhoefer/output/github-snake-dark.svg" />
  <source media="(prefers-color-scheme: light)" srcset="https://raw.githubusercontent.com/tobiasmeyhoefer/tobiasmeyhoefer/output/github-snake.svg" />
  <img alt="github-snake" src="https://raw.githubusercontent.com/tobiasmeyhoefer/tobiasmeyhoefer/output/github-snake.svg" />
</picture>

<div align="center">
  <img src="https://user-images.githubusercontent.com/74038190/212284158-e840e285-664b-44d7-b79b-e264b5e54825.gif" alt="Bottom Line" width="100%" />
</div>

