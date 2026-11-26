# StreamCount

**Counting unique visitors with probabilistic estimation.**

A Probability & Statistics college project that demonstrates **HyperLogLog** — a probabilistic algorithm that estimates the number of unique visitors to a website using a compact set of registers instead of storing every visitor ID.

> Live demo: open `index.html` in any browser (no build step, no internet required).

---

## Table of Contents

- [Project Overview](#project-overview)
- [Problem Statement](#problem-statement)
- [Solution](#solution)
- [Features](#features)
- [How It Works](#how-it-works)
- [Algorithm Details](#algorithm-details)
- [Probability & Statistics Connection](#probability--statistics-connection)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [How to Run Locally](#how-to-run-locally)
- [How to Use the Simulator](#how-to-use-the-simulator)
- [2–3 Minute Presentation Flow](#23-minute-presentation-flow)
- [Verification & Results](#verification--results)
- [Limitations](#limitations)
- [Future Scope](#future-scope)
- [Deploying Live on GitHub Pages](#deploying-live-on-github-pages)

---

## Project Overview

Traditional analytics store every visitor ID to count unique users exactly. That works, but memory grows linearly with the number of visitors.

**StreamCount** shows an alternative: hash each visitor ID, update one of **1,024 small registers**, and combine the registers to *estimate* the unique count — using only **≈ 1 KB** of memory, no matter how large the stream is.

The website is a single-page interactive dashboard that:

1. Explains the problem in one sentence
2. Lets you run a live visitor-stream simulation
3. Visualizes all 1,024 HyperLogLog registers
4. Connects the algorithm to Probability & Statistics
5. Compares exact vs. estimated counts honestly, including limitations

It is designed for a **2–3 minute live demonstration** in a classroom / viva setting.

---

## Problem Statement

> How can we estimate the number of unique visitors in a large stream of events without storing every visitor ID?

- Storing all IDs: memory = O(number of unique visitors)
- For millions of visitors, this becomes expensive and wasteful when an approximate answer is good enough

---

## Solution

Use **HyperLogLog (HLL)**, a probabilistic cardinality-estimation algorithm:

- Each visitor ID is hashed to a uniformly random 64-bit value
- The hash selects one register and updates it with the length of the longest run of zeros observed
- A mathematical estimator combines all registers into an estimated unique count
- Memory stays fixed at **1,024 registers × 1 byte = 1,024 bytes (≈ 1 KB)**

The project compares this estimate against the **exact count** computed with a `Set`, so the error is always visible and verifiable.

---

## Features

| Feature | Description |
|---|---|
| Interactive simulator | Generate a stream of 100 – 1,00,000 visitor events with repeat visitors |
| Exact count | Computed with a JavaScript `Set` (ground truth) |
| HLL estimate | Computed with a real HyperLogLog implementation (not faked) |
| Relative error | `\|estimate − exact\| / exact × 100` shown live |
| Register memory | Fixed 1 KB, displayed alongside the memory needed to store all IDs |
| Register visualization | All 1,024 actual registers as a 32 × 32 grid, with hover details |
| Worked example | Shows one real visitor ID → hash → register → ρ from the last run |
| Presets | 1,000 / 10,000 / 50,000 / **1,00,000** events for quick demos |
| Results comparison | Bar chart + summary table from the latest run |
| Responsive design | Works on projector, laptop, tablet, and mobile |
| Offline-ready | No external APIs, CDNs, fonts, or network calls |

---

## How It Works

### Step 1 — Hash
Each visitor ID is transformed into a binary hash.

```
visitor-0042  →  64-bit hash (uniformly random-looking bits)
```

### Step 2 — Register
The hash is used to select a register and determine the relevant pattern.

- Top **10 bits** → register index (0 – 1023)
- Next bits → count of leading zeros (**ρ**)
- The register keeps the **maximum ρ** seen so far

### Step 3 — Estimate
The information stored in registers is combined to estimate the total number of unique visitors.

```
Ê = αm · m² / Σ 2^(−M[j])
```

Registers with larger values contribute less — they signal a larger cardinality.

---

## Algorithm Details

### HyperLogLog configuration

| Parameter | Value |
|---|---|
| Registers (`m`) | **1,024** (2¹⁰) |
| Precision (`p`) | **10** |
| Register type | `Uint8Array` (1 byte per register) |
| Total memory | **1,024 bytes ≈ 1 KB** |
| Hash size | 64-bit |

### Hash function

A deterministic 64-bit hash implemented in JavaScript:

1. **FNV-1a 64-bit** over the visitor ID string
2. **splitmix64 finalizer** (avalanche) so similar IDs produce very different hashes

This gives uniformly distributed bits, which HyperLogLog requires. The hash is implemented with `BigInt` for correct 64-bit arithmetic.

### Per-event update

```
h   = hash64(visitorID)
idx = top 10 bits of h          → register index (0–1023)
w   = remaining bits, left-aligned
ρ   = 1 + leadingZeros(w)       → 1 if first remaining bit is 1, larger if longer zero-run
M[idx] = max(M[idx], ρ)
```

### Estimator

```
sum = Σ 2^(−M[j])   for all 1,024 registers
Ê   = αm · m² / sum
```

where `αm = 0.7213 / (1 + 1.079/m)` for `m ≥ 128`.

**Small-range correction (linear counting):** when `Ê ≤ 2.5·m` and some registers are still zero:

```
Ê = m · ln(m / V)     where V = number of zero registers
```

A large-range correction for 64-bit hashes is included for completeness (it does not trigger in this project's range).

### Exact count (for comparison)

```js
const exactSet = new Set(stream);   // all visitor IDs in the stream
const exact = exactSet.size;
```

---

## Probability & Statistics Connection

HyperLogLog works because of a simple probabilistic fact about random bit patterns:

- For a uniformly random hash, the probability that the first *k* bits are all zero is **(1/2)ᵏ**
- A run of 5 zeros is about **32× less likely** than a run of 0 zeros
- Each register stores the **maximum** ρ observed for its share of hashes
- The distribution of these maxima across 1,024 registers encodes how many distinct hashes were seen
- More distinct visitors → registers fill in and grow → the estimator inverts this relationship

### Relative standard error

```
σ ≈ 1.04 / √m
```

> Under the standard assumptions of HyperLogLog, the relative standard error is approximately 1.04 divided by the square root of the number of registers.

For **m = 1,024**:

```
σ ≈ 1.04 / √1024 ≈ 3.25%
```

This is a **standard error** (typical size of deviation), not an absolute guarantee or a strict maximum error. Individual runs can be somewhat better or worse.

---

## Tech Stack

### Libraries & frameworks

**None.** The project intentionally uses **zero external libraries, frameworks, or CDNs**.

| Category | Choice | Reason |
|---|---|---|
| Libraries | — (none) | No dependencies = runs offline, loads instantly, nothing to break during a live demo |
| Frameworks | — (none) | Single-page app; a framework would be overkill |
| Fonts | System font stack | No web-font download; consistent on any machine |
| Icons / images | Inline SVG only | No image assets to load |
| Backend / API | — (none) | All computation happens in the browser |
| Build tools | — (none) | No bundler, no compile step |

### Languages

| Language | Used for |
|---|---|
| **HTML5** | Page structure and semantics (single `index.html`) |
| **CSS3** | All styling — custom CSS with variables, flexbox/grid, responsive breakpoints. No CSS framework |
| **JavaScript (ES6)** | HyperLogLog implementation, simulator logic, register rendering, charts. Uses `BigInt` (64-bit hash), `Set` (exact count), `Uint8Array` (registers), `TextEncoder` (memory calculation) |

### Browser APIs used

- DOM API (rendering, events)
- `BigInt` arithmetic
- `Set`, `Uint8Array`, `TextEncoder`
- CSS Grid / Flexbox

---

## Project Structure

```
streamcount/
├── index.html    # Entire application: HTML + CSS + JavaScript in one file
└── README.md     # This file
```

Everything is in **one file** by design:

- Easier to present (one file to open)
- Easier to deploy (upload one file)
- No build step, no `node_modules`, no server required

---

## How to Run Locally

### Option 1 — Just open the file (simplest)

Double-click `index.html`, or drag it into Chrome / Edge / Firefox.

The simulator auto-runs on page load, so results are visible immediately.

### Option 2 — Local server (recommended for presentation)

```bash
# Python (pre-installed on most systems)
python3 -m http.server 8000
# then open http://localhost:8000
```

```bash
# Or with Node.js
npx serve .
```

> No `npm install` is needed — there are no dependencies to install.

---

## How to Use the Simulator

1. **Total visitor events** — slider or number box (100 – 1,00,000)
2. **Repeat visitors** — percentage of events that are returning visitors (0–90%)
   - At 40%, about 60% of events introduce a new unique visitor
3. **Presets** — one-click sizes: `1,000` · `10,000` · `50,000` · `1,00,000`
4. Click **▶ Run Simulation**

The dashboard then shows:

| Metric | Meaning |
|---|---|
| Total Events | Events in the generated stream |
| Exact Unique Visitors | Counted with a `Set` |
| Estimated Unique Visitors | HyperLogLog estimate |
| Relative Error | `\|estimate − exact\| / exact` |
| Register Memory | 1,024 bytes (1.00 KB) — always fixed |

Each run generates a **fresh random stream** (unique visitors + shuffled repeats), so the estimate varies slightly every time — exactly as a real probabilistic algorithm should.

---

## 2–3 Minute Presentation Flow

1. **Open the website** — explain StreamCount in one sentence (hero section)
2. **Click Run Simulation** — show total events, exact count, estimate, and error
3. **Show the registers changing** — scroll to the 32 × 32 grid
4. **Explain hashing and registers** — use the three-step section
5. **Point to the formula** — `σ ≈ 1.04/√m ≈ 3.25%` in the Probability & Statistics section
6. **Finish with memory + limitations** — 1 KB vs. storing all IDs; estimate ≠ exact count

---

## Verification & Results

The implementation was tested in Node.js before finalizing:

| Unique visitors | Estimate | Error |
|---|---|---|
| 100 | 101 | 0.80% |
| 1,000 | 1,045 | 4.52% |
| 6,000 | 6,015 | 0.26% |
| 10,000 | 9,717 | 2.83% |
| 30,000 | 30,606 | 2.02% |
| 1,00,000 | 1,04,296 | 4.30% |

- A full **1,00,000-event run takes ~0.2 seconds** in the browser
- Errors are consistent with the theoretical ~3.25% standard error
- Exact count, error %, and memory calculations were verified against the implementation

---

## Limitations

- It provides an **estimate**, not an exact count
- It **cannot identify individual visitors**
- Accuracy depends on the algorithm's assumptions and configuration
- It is **not suitable** where an exact count is required

---

## Future Scope

- Configurable register count (`m`) to show the error/memory trade-off live
- Multiple hash seeds / repeated trials with average error
- Export results to CSV for lab-report tables
- Compare HyperLogLog with other sketches (e.g., Bloom filter, Count-Min Sketch)

---

## Deploying Live on GitHub Pages

This project is a static site, so it deploys free on **GitHub Pages** in a few minutes.

### Step 1 — Create the repository

1. Go to [github.com/new](https://github.com/new)
2. Repository name: `streamcount`
3. Choose **Public**
4. Do **not** check "Add a README" (we already have one)
5. Click **Create repository**

### Step 2 — Push this project

From inside the `streamcount` folder on your computer:

```bash
git init
git add index.html README.md
git commit -m "StreamCount: HyperLogLog unique-visitor estimator"
git branch -M main
git remote add origin https://github.com/YOUR-USERNAME/streamcount.git
git push -u origin main
```

> Replace `YOUR-USERNAME` with your GitHub username.

### Step 3 — Enable GitHub Pages

1. Open your repo → **Settings** → **Pages** (left sidebar)
2. Under **Build and deployment**:
   - Source: **Deploy from a branch**
   - Branch: `main` / `(root)`
3. Click **Save**

### Step 4 — Open your live site

After 1–2 minutes, your site is live at:

```
https://YOUR-USERNAME.github.io/streamcount/
```

Share that link with your professor — it works on any device, no installation needed.

---

## Academic Note

StreamCount is a **Probability & Statistics Project** built for educational demonstration. All numbers on the page are computed live in the browser by the actual HyperLogLog implementation — no pre-filled or fabricated results.

---

*StreamCount · Probability & Statistics Project · JG University*
