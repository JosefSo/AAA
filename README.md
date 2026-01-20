# Procedural Fairness for Course Allocation (Draft + Improvement Phase)

This README explains how to add **procedural fairness** (fair “voice” in decisions) to a **course allocation** project that already uses:
- an initial **draft / snake** allocation (HBS-style), and
- an **improvement phase** with $N$ iterations (local search / swaps).

The goal is to discuss this idea with professors in a **simple, clear** way.

---

## 1) Motivation

Most course allocation systems measure fairness only by the **outcome**:
- who got top courses,
- total utility,
- inequality (e.g., Gini),
- envy.

But recent work on **procedural fairness** says:
> Fairness is also about the **process**: did each student have a fair chance to influence decisions?

In other words:
- Outcome fairness = *what you got*
- Procedural fairness = *how much your preferences actually mattered during decision making*

This is very relevant when we run:
- a greedy draft,
- then repeated improvements that might repeatedly benefit the same students.

---

## 2) Core Idea: “Voice” / Decision Share

### 2.1 What is Decision Share?

For each student $i$, define a set of **favorite options** (e.g., Top-1 or Top-$k$ courses still feasible).

**Decision Share** measures how often the algorithm gives the student something from their favorite set.

There are two practical definitions (choose one depending on your implementation):

#### A) Outcome-based (easy if you output final schedules only)
Each student receives $K_i$ courses.

Let `TopK(i)` = student's top-$k$ ranked courses.

$$
DS_i = \frac{\#\{c \in \text{Assigned}(i) \cap \text{TopK}(i)\}}{\min(K_i, k)}
$$

Interpretation:
- $DS_i = 1$: student got only top-$k$ courses (strong “voice”)
- $DS_i = 0$: student got none of their top-$k$ courses (weak “voice”)

#### B) Process-based (best if you have draft rounds / step logs)
At each step $t$, student $i$ has a favorite set among **currently available** courses.

$$
DS_i = \frac{1}{T_i}\sum_{t \in \text{steps of }i} \mathbf{1}\{\text{choice}_t(i) \in \text{Favorites}_t(i)\}
$$

Interpretation:
- measures whether the *process* consistently respects student’s top choices.

---

## 3) Why this is “New” for Course Allocation

Traditional metrics can look good while procedural fairness is bad.

Example:
- total utility improves,
- but the system repeatedly “listens” to the same group (high-priority students or lucky draft order),
- other students rarely get top feasible choices.

Procedural fairness gives an additional lens:
- **Are we optimizing welfare at the cost of student representation?**

---

## 4) Metrics to Add to the Project

### 4.1 Outcome metrics (standard)
- **Total Utility**:
  $$
  U_{\text{total}} = \sum_i U_i
  $$
- **Top-X rate**: % of students who got at least one Top-1 / Top-3 course
- **Gini of utilities**:
  - compute over $\{U_i\}$

### 4.2 Procedural metrics (new layer)
- **Decision Share per student**: $\{DS_i\}$
- **Gini(DS)**: inequality of “voice”
- **Min DS**: $\min_i DS_i$ (protects the worst-off)
- **DS Nash Welfare (DSNW)** (balances voice across students):
  $$
  DSNW = \prod_i (DS_i + \epsilon)
  $$
  where $\epsilon$ is a tiny constant to avoid multiplying by zero (only for the metric computation).

> DSNW is high only if **most** students have non-trivial voice.

### 4.3 Stability (optional but strong academically)
- **Swap-regret**: how many mutually beneficial swaps still exist after optimization?
- **Envy rate**: % of students who envy someone’s schedule

---

## 5) How to Integrate Procedural Fairness into the Algorithm

You already have:
1) **Phase A**: Draft allocation (snake)
2) **Phase B**: Improvement (N iterations)

Procedural fairness can be added mainly to Phase B.

---

## 6) Proposed Algorithm Variants

### Variant 0 (Baseline)
- Phase A only (draft), no improvements.

### Variant 1 (Welfare-only improvement)
- Phase A + Phase B local search
- Accept a move if it improves total utility:
  $$
  \Delta U_{\text{total}} > 0
  $$

### Variant 2 (Procedurally-aware improvement) ✅ (main proposal)
- Phase A + Phase B local search
- Accept a move only if it improves welfare **without harming voice**, e.g.:

**Rule A (hard constraint):**
$$
\Delta U_{\text{total}} > 0 \quad \text{and} \quad \min_i DS_i \ge \tau
$$
Meaning:
- improve welfare,
- but guarantee each student has at least $\tau$ voice (e.g., $\tau = 0.2$ for Top-3 matching).

**Rule B (multi-objective):**
$$
\text{accept if } \Delta \big(U_{\text{total}} + \lambda \cdot \log(DSNW)\big) > 0
$$
Meaning:
- tune $\lambda$ to trade off welfare vs procedural fairness.

**Rule C (tie-break):**
If two moves have similar welfare gain, choose the one with higher DSNW.

---

## 7) What “Moves” Can Phase B Use?

You can keep your current improvement logic and just add acceptance rules.
Typical moves:

1) **Single-course reassignment** (if capacity allows)
2) **2-swap**: swap one course between two students
3) **k-cycle**: A→B→C→A swaps (optional)

The main new part is not the move type — it’s the **procedural acceptance criteria**.

---

## 8) Data to Collect from Students

Minimum:
- ranking of courses (position 1..M)
- optional score/intensity per course (1..5)

For procedural fairness:
- define Top-$k$ for DS (e.g., $k=3$)

Optional (for a richer thesis):
- “avoid” list (1–2 people or courses)
- “friends” list (top-3 people)
- personal weight:
  - some students care more about friends,
  - some care more about courses.

---

## 9) Utility Model (Simple)

### 9.1 Course utility
Let $pos(i,c)$ be the rank (1 = best).
A simple decreasing function:

$$
U_{\text{course}}(i,c) = M - pos(i,c) + 1
$$

If you have a score $score(i,c)$, you can combine:

$$
U_{\text{course}}(i,c) = a \cdot score(i,c) + b \cdot (M - pos(i,c) + 1)
$$

### 9.2 Social utility (optional)
If student $i$ wants student $j$ in the same course:

$$
U_{\text{social}}(i) = \sum_{c \in \text{Assigned}(i)} \sum_{j \in \text{AssignedToCourse}(c)} w_{ij}
$$

Total:

$$
U_i = \sum_{c \in \text{Assigned}(i)} U_{\text{course}}(i,c) + \gamma \cdot U_{\text{social}}(i)
$$

---

## 10) Small Example (Toy)

- 4 students: A, B, C, D  
- 3 courses: X, Y, Z  
- Capacity: 2 each  
- Each student gets $K=1$

Suppose Top-2 per student:
- A: X, Y
- B: X, Y
- C: X, Z
- D: Y, Z

Outcome S1:
- A→X, B→X, C→Z, D→Y

Decision Shares (Top-2):
- A got X (Top-2) → DS=1
- B got X (Top-2) → DS=1
- C got Z (Top-2) → DS=1
- D got Y (Top-2) → DS=1

Outcome S2 (worse procedural):
- A→X, B→Y, C→Y, D→Z
If C’s Top-2 is {X,Z}, then C got Y → DS=0

Here total utility may look similar, but procedural fairness differs:
- S1: min DS = 1
- S2: min DS = 0

---

## 11) System Diagram

```mermaid
flowchart TD
  A[Input: course preferences<br/>rank/score + capacities + K] --> B[Phase A: Snake Draft Allocation]
  B --> C[Initial schedule S0]
  C --> D[Phase B: Improvement (N iterations)]
  D --> E{Proposed acceptance rule}
  E -->|Welfare-only| F[Accept if ΔU_total > 0]
  E -->|Procedurally-aware| G[Accept if improves welfare<br/>AND keeps DS fairness]
  F --> H[Final schedule S*]
  G --> H[Final schedule S*]
  H --> I[Report Metrics: Utility, Gini(U), DS, Gini(DS), DSNW, Stability]
```
