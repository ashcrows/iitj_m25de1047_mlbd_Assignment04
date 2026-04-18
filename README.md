# Assignment 4 — Clustering and PageRank

**Name:** Ashish Sinha
**Roll No:** M25DE1047
**Institute:** IIT Jodhpur | MTech Data Engineering
**Platform:** Google Colab | PySpark 4.0.2 | Java 17.0.18

---

## How to Run

### Step 1 — Upload the dataset zip
Upload **`Assignment_4-_datasets.zip`** to `/content/` using the **📁 folder icon** in the Colab left sidebar → *Upload to session storage*.

### Step 2 — Run all cells top to bottom
No runtime restart is needed. Google Colab ships with Java pre-installed. The notebook detects `JAVA_HOME` automatically using `readlink -f /usr/bin/java`, so it works regardless of whether Colab has Java 11, 17, or 21.

### Step 3 — Part 3 graph files
The PageRank graph files (`small.txt`, `whole.txt`) are **not in the zip**. They are downloaded automatically from GitHub inside the notebook:
```
https://github.com/pnijhara/PySpark-PageRank/tree/main/graphs
```

---

## Dataset Layout (inside the zip)

```
Assignment_4-_datasets.zip
└── Assignment 4- datasets/
    ├── Q1- UCI Spam clustering/
    │   ├── spambase.data          ← Part 1
    │   ├── spambase.DOCUMENTATION
    │   └── spambase.names
    └── Q2- webSearch/
        ├── actions.txt            ← Part 2 queries
        ├── answers.txt            ← Part 2 expected outputs
        └── webpages/
            ├── stack_datastructure_wiki
            ├── stack_cprogramming
            ├── stack_oracle
            ├── stackoverflow
            ├── stacklighting
            ├── stackmagazine
            └── references
```

---

## Part 1 — Clustering (40 Marks)

### Functions implemented

| Function | Signature | Description |
|---|---|---|
| `readVectorsSeq` | `readVectorsSeq(filename)` | Reads `spambase.data` → list of PySpark `Vectors.dense(...)` |
| `kcenter` | `kcenter(P, k)` | Farthest-First Traversal — O(\|P\|·k) |
| `kmeansPP` | `kmeansPP(P, k)` | k-means++ probabilistic D² seeding — O(\|P\|·k) |
| `kmeansObj` | `kmeansObj(P, C)` | Average squared distance to nearest centre |

### Three-step experiment (k = 10, k1 = 50)

| Step | Method | Result |
|---|---|---|
| 1 | `kcenter(P, k=10)` | Running time: **0.1695 s** |
| 2 | `kmeansPP(P, k=10)` → `kmeansObj(P, C)` | Objective: **29,799.338126** |
| 3 | `kcenter(P, k1=50)` → `kmeansPP(X, k=10)` → `kmeansObj(P, C)` | Objective: **420,676.616459** |

Step 2 achieves a lower objective than Step 3 because k-means++ runs directly on all 4601 points. Step 3 first compresses P to a 50-point Farthest-First coreset, then seeds k-means++ on it — faster but higher error when evaluated on full P. As k1 increases, Step 3 converges toward Step 2.

### Assumptions
- All 58 columns of `spambase.data` are used as feature dimensions (assignment: "4601 points with 58-dimensions").
- `kcenter` uses `P[0]` as the first centre.
- `kmeansPP` is non-deterministic by design (no fixed random seed).
- `Vectors.sqdist(a, b)` was removed in PySpark 4.x — replaced with `a.squared_distance(b)`.

---

## Part 2 — Web Search Engine (40 Marks)

### Class hierarchy
```
MySet → Position → WordEntry → PageIndex → PageEntry
     → MyHashTable → InvertedPageIndex → SearchEngine
```

### Text processing rules (exhaustive, per assignment)

| Rule | Specification |
|---|---|
| Lowercase | Every token lowercased before any processing |
| Stop words | `{a, an, the, they, these, this, for, is, are, was, of, or, and, does, will, whose}` — not stored; positions still count |
| Punctuation → space | `{}[]<>=(). ,;'"?#!-:` — note: `+` is **not** in this list |
| Plural → singular | `stacks→stack`, `structures→structure`, `applications→application` (exhaustive) |
| Positions | 1-based; stop words count toward position index but are NOT stored in the index |

### All 11 queries match `answers.txt` — ALL CORRECT: YES

| Query | Output |
|---|---|
| `queryFindPagesWhichContainWord delhi` | `No webpage contains word delhi` |
| `queryFindPagesWhichContainWord stack` | `stack_datastructure_wiki` |
| `queryFindPagesWhichContainWord wikipedia` | `stack_datastructure_wiki` |
| `queryFindPositionsOfWordInAPage magazines stack_datastructure_wiki` | `Webpage stack_datastructure_wiki does not contain word magazines` |
| `queryFindPagesWhichContainWord allain` (before stack_cprogramming added) | `No webpage contains word allain` |
| `queryFindPagesWhichContainWord allain` (after stack_cprogramming added) | `stack_cprogramming` |
| `queryFindPagesWhichContainWord C` | `stack_cprogramming` |
| `queryFindPagesWhichContainWord C++` | `stack_cprogramming` |
| `queryFindPagesWhichContainWord jdk` | `stack_oracle` |
| `queryFindPagesWhichContainWord function` | `stack_cprogramming, stack_datastructure_wiki, stackoverflow` |
| `queryFindPagesWhichContainWord magazines` | `stackmagazine` |

### Key edge cases

| Case | Explanation |
|---|---|
| `C++` query | `+` is NOT in the punctuation list → tokenised as one token `c++` |
| `magazines` | Not in the plural map → stored as `magazines`, matches `stackmagazine` directly |
| Action ordering | DB is stateful; `stackoverflow` not yet added at action 10 → only `stack_cprogramming` returned for `C++` |
| `"in"` | Not a stop word (list is exhaustive) → indexed and counted in positions normally |
| `stack` TF-IDF = 0 | Appears in all 6 indexed pages → IDF = log(6/6) = 0 |

### Assumptions
- Positions are 1-based, consistent with the assignment example.
- Stop word and plural-map lists are exhaustive — no entries added beyond those specified.
- TF denominator counts all tokens including stop words (standard definition).
- Output page names sorted alphabetically for deterministic ordering.

---

## Part 3 — PageRank on Spark (40 Marks)

$$r^{(t)} = \frac{1-\beta}{n}\mathbf{1} + \beta M r^{(t-1)}, \quad \beta=0.8, \quad 40 \text{ iterations}$$

### Dataset

Downloaded from GitHub inside the notebook (NOT in the zip):
```
https://github.com/pnijhara/PySpark-PageRank/tree/main/graphs
```

| File | Size | Edges | Description |
|---|---|---|---|
| `small.txt` | 5,978 bytes | 1,024 | 53-node graph for validation |
| `whole.txt` | 63,780 bytes | 8,192 | 1000-node main experiment graph |

### Results

**Validation — small.txt (53 nodes):**
Top node score ≈ 0.036 → **PASS**

**Main experiment — whole.txt (β = 0.8, 40 iterations):**

| Category | Rank | Node ID | Score |
|---|---|---|---|
| Top 5 | 1 | 694 | 0.00208821 |
| | 2 | 965 | 0.00192502 |
| | 3 | 95 | 0.00188745 |
| | 4 | 12 | 0.00182002 |
| | 5 | 460 | 0.00179724 |
| Bottom 5 | 996 | 876 | 0.00037020 |
| | 997 | 795 | 0.00037007 |
| | 998 | 478 | 0.00035269 |
| | 999 | 193 | 0.00035070 |
| | 1000 | 837 | 0.00032577 |

**Sum of all ranks: 1.000000** — correct normalisation confirmed.

### Spark RDD design
- Graph stored as `RDD[(src, [dst_list])]`.
- Edges deduplicated via `distinct()` before `groupByKey` (multiple edges between same pair = single edge, per assignment).
- Each iteration: `join` adjacency list with ranks → `flatMap` contributions → `reduceByKey` sum → add teleportation term.
- No dangling nodes — all nodes have positive out-degree (per assignment).

### Assumptions
- Graphs downloaded directly from the GitHub URL specified in the assignment.
- Multiple directed edges treated as a single edge via `distinct()`.
- 40 iterations with β = 0.8 provides practical convergence (residual < 10⁻⁶).
