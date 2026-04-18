# Assignment 4 — Clustering and PageRank
**Student ID:** M25DE1047 | **Institute:** IIT Jodhpur

---

## How to Use This Notebook

### Before running
1. Upload **`Assignment_4-_datasets.zip`** to `/content/` using the **📁 folder icon** in the Colab left sidebar → Upload to session storage.
2. Run all cells top to bottom — **no runtime restart needed**.

> Google Colab ships with Java pre-installed. Only PySpark needs to be pip-installed. The notebook detects JAVA_HOME automatically using `readlink`, so it works regardless of whether Colab has Java 11, 17, or 21.

### Setup cell order
| Cell | Purpose |
|------|---------|
| Cell 1 | `!pip install pyspark -q` |
| Cell 2 | Detect `JAVA_HOME` dynamically → start `SparkContext` |
| Cell 3 | Unzip `Assignment_4-_datasets.zip` → `/content/assignment4/` |
| Cell 4 | Resolve and verify all dataset file paths |
| Cell 5+ | Parts 1, 2, 3 |

---

## What caused the crash loop (and why it is fixed)

The previous version used `os.kill(os.getpid(), signal.SIGKILL)` to restart the runtime after installing Java. This caused a **crash loop**:

1. Cell A runs → kills the kernel.
2. Colab auto-restarts the kernel.
3. If "Run All" is active, Cell A runs again → kills again.
4. Loop repeats indefinitely (visible in your logs: 3 restarts at 9:59, 10:00, 10:01).

**The fix:** Java is already installed on Colab. No `apt-get`, no `os.kill`, no restart needed. The only root cause of `JAVA_GATEWAY_EXITED` was a wrong hardcoded `JAVA_HOME` path (`java-11` on a runtime that has `java-21`). Setting `JAVA_HOME` dynamically via `readlink -f /usr/bin/java` resolves this correctly.

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

The Part 3 graph (`small.txt`, `whole.txt`) is **not in the zip**. It is downloaded automatically from GitHub inside the notebook.

---

## Part 1 — Clustering (40 Marks)

### Functions implemented
| Function | Description |
|---|---|
| `readVectorsSeq(filename)` | Reads `spambase.data` → list of PySpark `Vector` (dense) |
| `kcenter(P, k)` | Farthest-First Traversal — O(|P|·k) |
| `kmeansPP(P, k)` | k-means++ probabilistic seeding — O(|P|·k) |
| `kmeansObj(P, C)` | Average squared distance to nearest centre |

### Three-step experiment
1. `kcenter(P, k)` — prints running time
2. `kmeansPP(P, k)` → `kmeansObj(P, C)` — prints objective
3. `kcenter(P, k1)` → coreset X → `kmeansPP(X, k)` → `kmeansObj(P, C)` — prints objective

### Assumptions
- All 58 columns of `spambase.data` are feature dimensions (assignment: "4601 points with 58-dimensions").
- `kcenter` uses `P[0]` as the first centre.
- `kmeansPP` is non-deterministic by design.

---

## Part 2 — Web Search Engine (40 Marks)

### Class hierarchy
```
MySet → Position → WordEntry → PageIndex → PageEntry
     → MyHashTable → InvertedPageIndex → SearchEngine
```

### Processing rules (exhaustive, per assignment)
| Rule | Value |
|---|---|
| Lowercase | Always |
| Stop words | `{a, an, the, they, these, this, for, is, are, was, of, or, and, does, will, whose}` |
| Punctuation → space | `{}[]<>=(). ,;\'"?#!-:` |
| Plural → singular | `stacks→stack`, `structures→structure`, `applications→application` |
| Positions | 1-based; stop words count toward position index but are NOT stored |

### Key edge cases verified against answers.txt
| Case | Explanation |
|---|---|
| `C++` query | `+` is NOT in the punctuation list → one token `c++` |
| `magazines` | Not in plural map → stored as `magazines`, matches `stackmagazine` directly |
| Action ordering | DB is stateful; `stackoverflow` not yet added at action 10 → only `stack_cprogramming` returned |
| `"in"` | Not a stop word → indexed normally |

All 11 outputs match `answers.txt` exactly.

---

## Part 3 — PageRank on Spark (40 Marks)

$$r^{(t)} = \frac{1-\beta}{n}\mathbf{1} + \beta M r^{(t-1)}, \quad \beta=0.8, \quad 40 \text{ iterations}$$

- Validates on `small.txt` (top score ≈ 0.036).
- Reports top-5 and bottom-5 node IDs with scores for `whole.txt`.
- Edges deduplicated via `distinct()`.
- Rank sum ≈ 1.0 confirms correct normalisation.
