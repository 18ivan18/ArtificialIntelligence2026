# k-Nearest Neighbours and Model Evaluation

Last week ended on a number we had no right to trust: one accuracy, from one split, of 114
examples. This week is about two things — the simplest possible learning algorithm, and how to tell
whether any algorithm is actually working.

They belong together. kNN has essentially one dial to turn, so it is the cleanest place to watch
what happens when you tune a model badly, and to see why measuring is harder than it looks.

## Basic Concepts

- Eager and lazy learning
- The kNN algorithm
- Distance metrics and the Minkowski family
- Why scaling decides everything for kNN
- Choosing k
- kd-trees
- The curse of dimensionality
- Bias and variance
- Train, validation and test
- k-fold and stratified cross-validation
- The confusion matrix, precision, recall and F1

## Two ways to learn

**Eager learning** builds a model from the training data first and then throws the data away. The
work happens up front; prediction is cheap. Linear and logistic regression from last week are eager
— after fitting, all that survives is a weight vector, and it does not matter whether there were
400 examples or 400 million.

**Lazy, or instance-based, learning** does no work at fitting time. It stores the examples and
answers each query by consulting them. There is no model in the usual sense: the data _is_ the
model.

|                   | Eager                     | Lazy                              |
| ----------------- | ------------------------- | --------------------------------- |
| Training cost     | high                      | none — just store the data        |
| Prediction cost   | low                       | high, grows with the training set |
| Memory            | the parameters only       | every example, forever            |
| Adding new data   | refit                     | append it                         |
| Decision boundary | fixed by the model family | whatever the data implies         |

kNN is the classic lazy learner, and its last row is its real attraction: it assumes nothing about
the shape of the boundary. Logistic regression could only ever draw a straight line. kNN will draw
whatever the examples support, including the XOR pattern that defeated week 6.

## The algorithm

To classify a new point:

1. Compute the distance from it to every training example.
2. Take the `k` closest.
3. Return the majority label among them.

That is all of it. For regression, return the mean of the `k` neighbours' values instead of the
majority.

Two refinements worth knowing:

- **Distance weighting.** Let nearer neighbours count for more, typically with weight `1/d`. This
  softens the choice of `k`, because distant neighbours contribute little regardless.
- **Ties.** With two classes and even `k` a vote can tie. Use odd `k` for two classes, or break
  ties by the nearest neighbour among the tied classes. With more than two classes an odd `k`
  guarantees nothing, so an explicit rule is needed either way.

Training is `O(1)`. Predicting one point by brute force is `O(Nn)` — every example, every feature.

## Distance metrics

Everything above depends on what "closest" means. The standard family is the **Minkowski distance**:

    D(x, y) = (Σᵢ |xᵢ - yᵢ|^p)^(1/p)

Three values of `p` have names:

| p   | name          | formula          | shape of the unit ball |
| --- | ------------- | ---------------- | ---------------------- | --- | --------- |
| 1   | Manhattan, L1 | `Σ               | xᵢ - yᵢ                | `   | a diamond |
| 2   | Euclidean, L2 | `√(Σ(xᵢ - yᵢ)²)` | a circle               |
| ∞   | Chebyshev, L∞ | `maxᵢ            | xᵢ - yᵢ                | `   | a square  |

Euclidean is the default and usually right for continuous measurements. Manhattan is more robust
when a single feature can be wildly off, since it does not square the discrepancy. Chebyshev cares
only about the worst single feature.

For anything that is not a vector of numbers — text, sets, sequences — the metric is where the
domain knowledge goes, and choosing it matters more than choosing `k`.

## Scaling is not optional here

Distance sums over features, so a feature measured in larger units contributes more, purely because
of its units. This is not a mild effect but a fatal one.

The wine data in the notebook has thirteen measurements. `proline` has a standard deviation of 314;
`nonflavanoid_phenols` has 0.12. That is a factor of **2,530**. Euclidean distance on the raw
numbers is, to three decimal places, the distance in `proline` alone — the other twelve
measurements are noise on top of it.

| features     | test accuracy, k = 5 |
| ------------ | -------------------: |
| raw          |                0.722 |
| standardised |            **1.000** |

Same algorithm, same data, same `k`. This is the single most important practical fact about kNN,
and it is why "do I need to normalise?" is a question worth being able to answer rather than guess.

The answer is not always yes. If every feature is already in the same unit and the differences
between them are meaningful, scaling can destroy information rather than rescue it. The question to
ask is whether a unit of feature A should really count the same as a unit of feature B.

## Choosing k

`k` controls how much the model smooths.

- **k = 1** — every training point is its own island. Training accuracy is exactly 100%, because the
  nearest neighbour of a training point is itself. This is the purest example of overfitting you
  will see, and the perfect training score is the tell.
- **large k** — every query averages over a large neighbourhood, and the boundary flattens. Push it
  to `k = N` and every prediction is the majority class.

So `k` is the bias-variance dial: small `k` is low bias and high variance, large `k` the reverse.

The right `k` depends on the data, and the honest way to choose it is cross-validation on the
training set — never the test set, for reasons below.

## kd-trees

Brute force costs `O(N)` distance computations per query: fine for a few thousand examples,
hopeless for millions. A **kd-tree** (a _k_-dimensional tree) organises the training points so that
most of them never get measured at all. It changes how the neighbours are found, never which ones:
a correct kd-tree returns exactly the neighbours brute force does.

### The structure

A binary tree in which every node holds one training point and splits space with it along one
feature. Everything in the node's left subtree has a value `≤` the node's value in that feature,
everything in the right subtree `≥`. Each node therefore owns an axis-aligned box of space, its two
children split that box in two, and the leaves are the smallest boxes.

### Building it

    build(points):
        if there are no points: return nothing
        a ← the feature to split on
        m ← the median point along a
        node ← m, splitting on a
        node.left  ← build(points with value ≤ m's in a, except m)
        node.right ← build(points with value ≥ m's in a, except m)

Two choices to be able to defend:

- **The median.** Each level halves the points, so the tree is balanced, about `log₂ N` deep: seven
  levels for 120 iris flowers. Finding the median is linear (`nth_element` in C++),
  though sorting each level as the notebook does is simpler and costs `O(N log N)` overall.
- **The feature.** Either cycle through them with the depth (`depth mod n`), the textbook version,
  or take the feature with the widest spread among the points at hand, which makes squarer boxes
  when features have different ranges. Both are correct; the second usually prunes better.

### Searching for the k nearest

Keep the best `k` candidates found so far in a **max-heap** ordered by distance, so the worst of them
is on top and is the one to drop when something better turns up. Then:

    search(node, q):
        if node is empty: return
        measure the distance from q to node's point; push it; if more than k, pop the worst
        diff ← q[a] − node's value in a                (a: the node's feature)
        near, far ← the child on q's side of the split, and the other one
        search(near, q)
        if fewer than k candidates  or  |diff| ≤ the worst candidate's distance:
            search(far, q)

Going to the query's own side first matters: it finds good candidates early, and good candidates
are what make the pruning test succeed.

**Why skipping the far side is safe.** Every point beyond the split differs from `q` by at least
`|diff|` in that one feature, and a Minkowski distance with `p ≥ 1` is never smaller than the
difference in any single feature. So if `|diff|` is already worse than the current `k`-th best,
nothing over there can get in, and the whole subtree is discarded unmeasured. It is the same
bound-and-prune argument as alpha-beta in week 5: a cheap bound proves a branch cannot matter, so
it is never explored.

**Ties.** Use `≤`, not `<`. If brute force breaks ties between equally distant points by index (as a
sorted list of `(distance, index)` pairs does), a far-side point at exactly the `k`-th distance but
with a smaller index must still be visited, or the tree returns a different, equally near
neighbour. Iris has duplicate measurements, so this happens.

### By hand

Six points: A(2,3) B(5,4) C(9,6) D(4,7) E(8,1) F(7,2), splitting on the widest feature.

    F (7,2) splits on x = 7          x spreads 7, y spreads 6
    ├─ left:  B (5,4) splits on y = 4
    │   ├─ left:  A (2,3)
    │   └─ right: D (4,7)
    └─ right: C (9,6) splits on y = 6
        └─ left:  E (8,1)

The nearest neighbour of `q = (3,4)`:

1. F: distance 4.47, the best so far. `q` is left of `x = 7`, so go left.
2. B: distance 2.00, the best so far. `q` is on the line `y = 4`; take the left side first.
3. A: distance **1.41**, the best so far.
4. Back at B: the split is 0 away, within 1.41, so D must be checked: 3.16, no better.
5. Back at F: the split is |3 − 7| = 4 away, more than 1.41. **C and E are never measured.**

The answer is A, after four distances instead of six. The notebook builds this tree and prints the
same trace.

### What it costs, measured

Distances measured per query with `k = 10`, from the notebook. First more points, in two
dimensions:

|  points | distances measured |
| ------: | -----------------: |
|   1,000 |                 41 |
|  10,000 |                 48 |
| 100,000 |                 53 |

Then more features, with 20,000 points:

| features | share of points measured |
| -------: | -----------------------: |
|        2 |                     0.2% |
|        5 |                     2.4% |
|        8 |                      18% |
|       10 |                      69% |
|       12 |                      95% |
|       16 |                    99.8% |

In two dimensions a hundred times more points cost twelve more distances: about `O(log N)` per
query, against brute force's `O(N)`. But the pruning relies on the splitting plane being farther
away than the current neighbours, and as features are added the neighbours themselves drift far
away in every direction, so no plane is far enough. With 20,000 points the tree measures almost
everything from about twelve features on. A rough rule: a kd-tree pays only while `N` is much
larger than `2^n`, since it takes about that many points to split every feature even once. The
wine data in the notebook, thirteen features and 142 bottles, is on the wrong side of it: the tree
measures 98% of the bottles, correctly and for nothing. Why distance stops meaning anything in high
dimensions is the next section.

For the homework, the tree only has to replace the "find the k nearest" step. Give it the same
output as brute force (a list of `(distance, index)` pairs, nearest first) and check that the two
lists are identical for every test point and several values of `k`; identical lists can only come
from a correct tree.

## The curse of dimensionality

Adding features feels like adding information. Past a point it is the opposite, and kNN suffers
first because it depends entirely on "near" being meaningful.

**Space empties out.** Put `N` points in an `n`-dimensional unit cube. The smallest cube containing
a point's `k` nearest neighbours has side `l = (k/N)^(1/n)`. With `k = 10` and a **million** points:

| dimensions | side of the neighbourhood |
| ---------: | ------------------------: |
|          2 |                     0.003 |
|          3 |                     0.022 |
|         10 |                     0.316 |
|        100 |                 **0.891** |

In a hundred dimensions the ten nearest of a million points fill a cube spanning 89% of the range
of every feature. They are not local in any useful sense, and calling them "neighbours" is
wishful thinking.

**Everything is on the surface.** The fraction of a unit hypercube lying within 1% of its boundary
is `1 - 0.99ⁿ`: about 2% at `n = 2`, 63% at `n = 100`, and 87% at `n = 200`. In high dimensions
almost every point is an edge case.

**Irrelevant features drown the signal.** This is the practical form of the problem. Distance sums
over all features, so features carrying no information still contribute their full share of noise.
Measured on the wine data, adding pure random-noise columns to the thirteen real ones:

| noise features added | test accuracy |
| -------------------: | ------------: |
|                    0 |         1.000 |
|                   13 |         0.861 |
|                   50 |         0.833 |
|                  200 |         0.778 |
|                  500 |         0.472 |

Thirteen real features do not get any worse. They simply stop being audible, until at 500 noise
columns the classifier is barely above guessing.

## Measuring: bias and variance

Two sources of error, and they trade off:

- **Bias** — the model is systematically wrong, because its family cannot express the truth.
  Underfitting. Easy to spot: performance is poor on the training set too.
- **Variance** — the model reacts to noise in the particular sample it was trained on, so a
  different sample would give a noticeably different model. Overfitting. Harder to spot, because
  training performance looks excellent.

The tell for variance is the **gap** between training and held-out performance. kNN with `k = 1`
scores 1.000 on its training set and less on anything else; that gap is the entire diagnosis.

## Train, validation and test

One split is not enough, because the moment you use a set to make a choice, it stops measuring.

- **Training set** — fit the model.
- **Validation set** — choose between models: `k`, the metric, the features.
- **Test set** — touched **once**, at the end, to report.

The rule that matters: **if a number influenced a decision, it is no longer an unbiased estimate of
anything.** Picking `k` by test accuracy and then quoting that accuracy is the most common way to
report a result that will not survive contact with new data. It is not fraud, it is arithmetic —
you searched twenty values of `k` for the one that best fits 36 examples.

### Cross-validation

Holding out a validation set is wasteful when data is scarce — and it is itself a single small
sample, so the estimate is noisy.

**k-fold cross-validation** fixes both. Split the training data into `k` equal folds; for each fold,
train on the other `k-1` and score on it. Every example is used for training `k-1` times and for
testing exactly once. The result is `k` scores, and both numbers matter: the **mean** estimates
performance, and the **standard deviation** says how much to trust it.

**Stratified** k-fold keeps each fold's class proportions equal to the whole. With small or
imbalanced data this is not a refinement but a requirement — an unstratified fold can end up
missing a class entirely.

The standard deviation deserves more respect than it usually gets. In the notebook, 10-fold CV on
the wine data gives spreads of ±3 to ±5 percentage points across folds, which is larger than the
difference between most values of `k`. On a single run the ranking of `k` is mostly noise. Repeat
the CV over twenty different shuffles and the real curve appears:

|   k | mean CV accuracy over 20 shuffles |
| --: | --------------------------------: |
|   1 |                            0.9494 |
|   5 |                            0.9386 |
|   9 |                            0.9582 |
|  25 |                        **0.9712** |
|  51 |                            0.9553 |

Now the shape is unmistakable — a dip at `k = 3`, a climb to a peak in the twenties, then a fall
away again — and the uncertainty on each entry is around half a percentage point. Nothing changed
but how many times it was measured.

## Beyond accuracy

Accuracy is one number over all classes, and it hides everything interesting. The **confusion
matrix** does not: rows are the true class, columns the predicted, so every off-diagonal cell says
exactly which mistake was made.

For two classes, writing TP, FP, TN, FN for the four cells:

    Accuracy  = (TP + TN) / (TP + TN + FP + FN)
    Precision = TP / (TP + FP)      of the ones flagged, how many were right
    Recall    = TP / (TP + FN)      of the ones there were, how many were found

Precision and recall pull against each other: flag everything and recall is perfect while precision
collapses; flag only the certain cases and the reverse. Which one matters is a question about the
application, not the model. A cancer screen wants recall — a missed case is far worse than a false
alarm. A spam filter wants precision — a lost letter is worse than a nuisance.

**F1** combines them as a harmonic mean, which punishes imbalance between the two:

    F1 = 2PR / (P + R)

and the general form lets you say which you care about, with `β > 1` favouring recall:

    Fβ = (β² + 1)PR / (β²P + R)

### When accuracy lies

On imbalanced data accuracy is close to meaningless. From the notebook, a problem where 8.5% of
cases are positive:

| model               |  accuracy | precision | recall |    F1 |
| ------------------- | --------: | --------: | -----: | ----: |
| always predict "no" |     0.912 |     0.000 |  0.000 | 0.000 |
| kNN, raw features   |     0.912 |     0.000 |  0.000 | 0.000 |
| kNN, standardised   | **0.965** |     0.800 |  0.800 | 0.800 |

The first two rows have identical accuracy, and both are worthless — they never find a single
positive case. Accuracy cannot tell them apart from a useful model, and recall separates them
instantly. Whenever one class is rare, quote precision and recall, or quote the confusion matrix
and let the reader do it.

## What to carry forward

|            | kNN                                                          |
| ---------- | ------------------------------------------------------------ |
| Training   | store the data                                               |
| Prediction | `O(Nn)` brute force, better with a kd-tree in low dimensions |
| Assumes    | nothing about the boundary shape                             |
| Breaks on  | unscaled features, irrelevant features, many dimensions      |
| Tuning     | `k`, the distance metric, the weighting                      |

Three things that generalise past this week:

- **The simplest algorithm is competitive more often than you expect.** kNN with no parameters to
  fit reaches 100% on the wine test set. Reach for it as a baseline before anything clever.
- **Preprocessing outweighs the algorithm.** Scaling moved accuracy from 0.722 to 1.000; no choice
  of `k` comes close to mattering that much.
- **A single number is not a result.** One split of 36 examples moves by 2.8% per sample. Report a
  mean and a spread, or report the confusion matrix, but do not report one accuracy and call it
  measured.
