# Introduction to Machine Learning: Linear and Logistic Regression

For five weeks the program has known the rules. The map, the constraints, the game — someone wrote
them down, and the work was searching the space those rules defined.

This week nobody writes the rules down. We have examples instead: inputs paired with the answers
we wanted, and a program that has to find the rule itself. That is the whole of supervised
learning, and it changes what "solving" means. There is no goal state to reach and no proof of
optimality at the end — only a hypothesis that fits the data we have, and a hope that it fits the
data we do not.

## Basic Concepts

- Supervised, unsupervised and reinforcement learning
- Supervised learning: features, labels, hypothesis
- Regression and classification
- Loss functions, weight space and the loss surface
- Gradient descent
- The normal equation
- Feature scaling
- Measuring a fit: RMSE and R²
- Overfitting and regularisation
- The sigmoid, cross entropy and logistic regression
- Linear separability

## Three kinds of learning

Supervised learning is one of three settings, and what separates them is the kind of feedback the
program gets.

|                    | given                              | learned                         | example                                 |
| ------------------ | ---------------------------------- | ------------------------------- | --------------------------------------- |
| **Supervised**     | inputs paired with correct answers | a function from input to answer | this week: diagnosis, spam detection    |
| **Unsupervised**   | inputs only, no answers            | structure in the data itself    | grouping customers nobody has labelled  |
| **Reinforcement**  | only a reward, after acting        | a policy: which action, when    | learning a game from the final score    |

The real difference is where the supervision comes from, and what it costs. Supervised learning
needs somebody to have labelled the examples by hand — that is usually the binding constraint, not
the algorithm. Unsupervised learning needs no labels but cannot be told what to look for, so
"is this the right grouping" has no clean answer. Reinforcement learning gets the sparsest signal
of the three: a reward that may arrive hundreds of actions after the decision that earned it.

This course is mostly supervised — this week through week 9, then weeks 11 and 12. Week 10, k-means
clustering, is the unsupervised one. Reinforcement learning sits outside the course, though week 5
was the same shape of problem, acting now for a payoff later, solved by knowing the rules rather
than learning them.

## Supervised learning

Three things:

- **Features** `X` — the inputs, one row per example, one column per measurement.
- **Labels** `y` — the correct answer for each row.
- **A hypothesis** `h` — a function from a row to a prediction, chosen from some family by fitting
  it to the pairs.

The pairs are the specification. The program is never told what a tumour is or what a house is
worth; it is told what the answer was 400 times and asked for the 401st.

Two kinds of answer, and the split runs through everything below:

|               | Regression                 | Classification                   |
| ------------- | -------------------------- | -------------------------------- |
| Output        | a continuous number        | one of a fixed set of labels     |
| Question      | how much, how many         | which one                        |
| Example       | price, temperature, dosage | spam or not, malignant or benign |
| Typical loss  | squared error              | cross entropy                    |
| Typical score | RMSE, R²                   | accuracy                         |

Linear regression answers the first. Logistic regression — despite the name — answers the second.

## Univariate linear regression

One feature, one output, a straight line:

    h(x) = w₁x + w₀

`w₀` and `w₁` are the **weights**, and fitting means choosing them. To choose, we need to say what
a bad choice is. The traditional answer is the **squared error**, summed over the training set:

    Loss(w) = Σⱼ (yⱼ - h(xⱼ))² = Σⱼ (yⱼ - (w₁xⱼ + w₀))²

Squared rather than absolute for two reasons: it punishes a few large errors more than many small
ones, and it is differentiable everywhere, which is what the next section needs.

For this case the minimum can be written down directly. Setting both partial derivatives to zero
gives a unique solution:

    w₁ = (N Σxⱼyⱼ - Σxⱼ Σyⱼ) / (N Σxⱼ² - (Σxⱼ)²)
    w₀ = (Σyⱼ - w₁ Σxⱼ) / N

So for a straight line through a cloud of points, no search is needed at all. That is the
exception, not the rule, which is why we keep going.

## Weight space and the loss surface

Worth building the picture now, because every learning algorithm later is a walk across it.

Fix the training data. Then `Loss` is a function of the weights alone — not of `x`. With two
weights that is a surface over the `(w₀, w₁)` plane: every point is a possible line, and its height
is how badly that line fits. Fitting means finding the lowest point.

For squared-error loss with a linear hypothesis the surface is a **bowl** — convex, one minimum, no
local traps. That is a strong property and it does not survive contact with more interesting
models; by the time we reach neural networks the surface is full of dents.

## Gradient descent

The general method, and the reason it matters is that it never asks for a closed form. Start
anywhere, work out which way is downhill, take a step, repeat:

    w ← any point in weight space
    while not converged:
        for each wᵢ:
            wᵢ ← wᵢ - α ∂Loss/∂wᵢ

`α` is the **learning rate**: how far to step. The gradient gives the direction; `α` alone decides
the distance.

For one training example `(x, y)` the chain rule gives

    ∂Loss/∂w₀ = -2(y - h(x))        ∂Loss/∂w₁ = -2(y - h(x))·x

and folding the 2 into `α` and summing over all examples gives the update rule:

    w₀ ← w₀ + α Σⱼ (yⱼ - h(xⱼ))
    w₁ ← w₁ + α Σⱼ (yⱼ - h(xⱼ))·xⱼ

Read what it says. `(y - h(x))` is the error, signed. If the prediction is too low the error is
positive and the weight goes up; too high and it goes down. And the step is proportional to `x`, so
a feature that was large at the time of the mistake gets most of the correction — it is held most
responsible.

**Choosing `α` is the whole practical difficulty.** Too small and it crawls; too large and each
step overshoots the bottom and lands further up the other side, so the loss grows and the weights
run off to infinity. There is no safe default, because the usable range depends on the data — see
feature scaling below.

Three variants, by how many examples go into one step:

- **Batch** — all of them. Each step is an exact gradient and the cost per step is `O(N)`.
- **Stochastic (SGD)** — one example, chosen at random. Noisy steps, but very many of them, and the
  noise can knock it out of bad spots.
- **Mini-batch** — a few dozen to a few hundred. What is actually used, because it is the version
  that maps onto vector hardware.

## Multivariable linear regression

With `n` features the hypothesis is a weighted sum:

    h(x) = w₀ + w₁x₁ + ... + wₙxₙ

The intercept `w₀` is the odd one out, and the standard trick removes it: invent a feature `x₀`
that is always 1. Then the hypothesis is a plain dot product,

    h(x) = w · x

and the update rule is the univariate one with an index on it:

    wᵢ ← wᵢ + α Σⱼ (yⱼ - h(xⱼ))·xⱼ,ᵢ

### The normal equation

The closed form survives into many dimensions. With `X` the data matrix and `y` the label vector,
the loss is `‖Xw - y‖²`, and setting its gradient to zero gives

    w* = (XᵀX)⁻¹Xᵀy

`(XᵀX)⁻¹Xᵀ` is the **pseudoinverse** of `X`. One line, no learning rate, no iterations, exact.

So why iterate at all? Because of the inverse. `XᵀX` is `n × n` in the number of features, and
inverting it costs about `O(n³)`. At ten features that is nothing; at ten thousand it is the whole
problem, and at a million rows the matrix may not fit in memory. Gradient descent costs `O(Nn)` per
step and never forms the matrix.

|                         | Normal equation       | Gradient descent |
| ----------------------- | --------------------- | ---------------- |
| Iterations              | none                  | many             |
| Learning rate           | none                  | must be tuned    |
| Cost                    | `O(n³)` plus `O(Nn²)` | `O(Nn)` per step |
| Scales to many features | no                    | yes              |
| Works for other models  | no                    | yes              |

The last row is the important one. The normal equation exists only because squared error and a
linear hypothesis happen to give a solvable equation. Gradient descent needs nothing but a
derivative, which is why the rest of the course uses it.

## Feature scaling

Gradient descent takes one step size in every direction at once. If one feature ranges over
thousands and another over fractions, the surface is a long thin valley rather than a bowl, and the
learning rate that is safe for the steep direction is far too small for the shallow one.

The fix is to put the features on a common scale before fitting — usually **standardisation**,
subtracting each column's mean and dividing by its standard deviation.

This is not a small effect. On the diabetes data in the notebook, fitting all ten features:

| features     | largest learning rate that converges | R² after 2000 steps |
| ------------ | ------------------------------------ | ------------------: |
| raw          | 0.00001                              |              0.4149 |
| standardised | 0.1                                  |          **0.5176** |

The best achievable is 0.5177, so standardised gradient descent is essentially done after 2000
steps, while the raw version is still far off — and it is limited to a learning rate **ten thousand
times smaller**. Same algorithm, same data, same number of steps.

Note what scaling does not change: the normal equation gives the identical fit either way, and the
predictions after fitting are the same. Scaling is about whether gradient descent can move, not
about what the right answer is.

## Measuring a regression: RMSE and R²

Fitting minimises a **loss**; reporting needs a **score**, and they are not the same thing. A loss
has to be differentiable and its absolute value means little. A score does not have to be
differentiable, and it does have to be interpretable.

**RMSE** is the square root of the mean squared error. Its virtue is units — it comes out in
whatever `y` is measured in, so it reads as a typical miss. Its weakness is that it does not travel:
is an RMSE of 53 good? There is no answer until you know the target runs from 25 to 346.

**R²** fixes that by scoring against a baseline, and the baseline is the laziest model available —
ignore every feature and predict the mean of `y` every time.

    R² = 1 - SS_res / SS_tot = 1 - Σⱼ (yⱼ - h(xⱼ))² / Σⱼ (yⱼ - ȳ)²

`SS_res` is how much the model missed by, `SS_tot` how much the baseline missed by. The ratio is
your error as a fraction of the baseline's, and R² is one minus it, so larger is better.

| R²    | meaning                                  |
| ----- | ---------------------------------------- |
| `1`   | every prediction exact                   |
| `0`   | exactly as good as predicting the mean   |
| `< 0` | **worse** than predicting the mean       |

The last row is the one students trip over, because the name says square. Nothing here is squared —
R² is one minus a ratio, and it goes negative whenever the model is worse than a flat line. Fitted
on training data with an intercept that cannot happen, so a negative R² is always a report about
data the model never saw.

Worked on the three points `(1,2) (2,3) (3,5)` and the line `h(x) = 1.5x + 0.333` fitted earlier:

    ȳ      = 3.333
    SS_tot = (2-3.333)² + (3-3.333)² + (5-3.333)²  = 4.667
    SS_res = 0.167²     + 0.333²     + 0.167²      = 0.167
    R²     = 1 - 0.167/4.667 = 0.964

Two cautions. R² **never decreases** when you add a feature to a training fit, even a column of pure
noise, so training R² is not evidence of anything — which is the next section. And a good R² says
only that you beat the mean, not that the model is right; a fit can score 0.5 and still be
systematically wrong exactly where it matters, which is what the predicted-against-actual plot in
the notebook is for.

On the diabetes data the comparison is concrete: predicting the training mean gives test RMSE 78 and
R² -0.02, while the fitted model gives RMSE 53 and R² 0.53. Just over half the baseline's error,
removed.

## Overfitting and regularisation

With one feature there is little room to go wrong. With many, a feature that is pure noise can
still line up with the labels by chance, and the fit will use it — beautifully on the training data
and worse than useless on anything new.

The standard defence is to charge for complexity:

    Cost(h) = EmpLoss(h) + λ · Complexity(h)

with complexity measured on the weights themselves:

    Complexity(w) = Σᵢ |wᵢ|^q

`q = 2` is **L2** regularisation (ridge): it shrinks all weights smoothly toward zero. `q = 1` is
**L1** (lasso): it drives some weights to exactly zero, which selects features as a side effect.
`λ` sets the exchange rate between fitting and simplicity, and it is chosen by trying values, not
derived.

## Logistic regression

Now classification, and one change does it.

A linear function outputs any real number, but a class needs a probability. So push the linear
output through the **sigmoid**:

    σ(z) = 1 / (1 + e⁻ᶻ)
    h(x) = σ(w · x)

The sigmoid maps everything into `(0, 1)`, is smooth, and has a derivative that costs nothing once
you have the value:

    σ'(z) = σ(z)·(1 - σ(z))

The output is read as the probability of the example belonging to class 1. Predicting a class means
thresholding, usually at 0.5 — and `σ(z) ≥ 0.5` exactly when `z ≥ 0`, so the **decision boundary**
is `w · x = 0`, a straight line. Still a linear model; the sigmoid only bends the output, not the
boundary.

### Cross entropy

Squared error is the wrong loss here. It is nearly flat when the model is confidently wrong —
`σ` saturates, its derivative goes to zero, and the gradient vanishes exactly where the biggest
correction is needed.

**Cross entropy** is the right one. Between a true distribution `p` and an estimated `q`:

    H(p, q) = -Σᵢ p(i) log q(i)

For two classes, with `y` the true label and `ŷ` the predicted probability of class 1:

    Loss = -Σ [ y log ŷ + (1 - y) log(1 - ŷ) ]

One of the two terms is always zero, so each example contributes `-log(probability assigned to the
truth)`. That is unbounded: being confidently wrong costs arbitrarily much, which is precisely the
behaviour squared error lacked. For `C` classes it generalises to `-Σ_c y_c log ŷ_c`.

### The update rule does not change

Differentiate cross entropy through the sigmoid and almost everything cancels:

    ∂Loss/∂wᵢ = (y - h(x))·xᵢ

which is the same rule as linear regression, with a different `h`:

    wᵢ ← wᵢ + α Σⱼ (yⱼ - h(xⱼ))·xⱼ,ᵢ

This is worth pausing on. Linear and logistic regression differ in **one line of code** — whether
the linear output passes through the sigmoid on its way out. The loss differs, the gradient does
not. The notebook uses a single fitting function for both.

## The limit: linear separability

Logistic regression draws one straight boundary. If the classes cannot be separated by a straight
line, no amount of training helps — the model cannot represent the answer, so there is nothing for
gradient descent to find.

The smallest example is XOR. `AND` and `OR` are learned perfectly; `XOR` is not learnable at all,
and the failure is instructive: the fit converges to predicting **0.5 for all four inputs**, 50%
accuracy, the best a straight line can do when no straight line works.

This is a statement about the hypothesis family, not the algorithm. The fix is to change the family
— add features, or stack these units into layers, which is where neural networks come from.

## What this week actually gives you

|                     | fits          | loss          | closed form          | boundary        |
| ------------------- | ------------- | ------------- | -------------------- | --------------- |
| Linear regression   | a number      | squared error | yes, normal equation | not applicable  |
| Logistic regression | a probability | cross entropy | no                   | a straight line |

And the measured results from the notebook:

| task                              | model               | result                                                       |
| --------------------------------- | ------------------- | ------------------------------------------------------------ |
| Diabetes progression, 10 features | gradient descent    | test R² 0.532, matching the normal equation to three decimals |
| Breast cancer, 30 features        | logistic regression | **97.4%** test accuracy, 111 of 114                          |
| XOR                               | logistic regression | 50%, and it cannot do better                                 |

Two things to carry forward. **Gradient descent is the general method** — it asked nothing of the
problem except a derivative, and it will still be the method when the model is a network with
millions of weights. And **the model family sets the ceiling**: no optimiser rescues a hypothesis
that cannot express the answer, which is the lesson XOR is there to teach.
