# Genetic Algorithms

Last week ended with min-conflicts: start from a complete guess and repair it. That is local
search with a population of one. A genetic algorithm keeps a whole population of guesses, lets the
better ones have more children, and builds the next generation by **combining** pairs of them
rather than by tweaking one.

That combination step — crossover — is the only real novelty. Everything else here is hill
climbing with more bookkeeping.

## Basic Concepts

- Individuals, population, fitness
- Selection
- Crossover
- Mutation
- Elitism, culling, and how a generation is replaced

## What a genetic algorithm is

A population of individuals, each encoding a candidate solution. Score them, let the fitter ones
reproduce, and repeat.

    start with a random population
    repeat until good enough, or out of patience:
        score every individual with the fitness function
        choose parents, favouring the fitter ones
        combine each pair into children (crossover)
        perturb the children slightly (mutation)
        the children, plus perhaps the best few parents, become the next generation

Written as the cycle it usually gets drawn as:

    initial population
        -> evaluation      score everyone with the fitness function
        -> selection       choose who reproduces
        -> crossover       combine pairs into children
        -> mutation        perturb the children
        -> evolved population
        -> convergence check  ---- not yet? back to evaluation
        -> end

The **convergence check** is the part that is easy to leave vague. A GA has no natural stopping
point, so you have to supply one: a fitness good enough to accept, a generation budget, or a number
of generations with no improvement. That last one is usually the most useful in practice, because
it stops as soon as the population has run out of ideas rather than after an arbitrary count.

Seen from week 3's angle, it is a **stochastic beam search with a recombination step**: instead of
keeping the `k` best states and generating successors of each, it keeps `k` states and generates
successors from _pairs_. Take the crossover away — combine one parent with itself — and that is
exactly what you have left.

## The parts that vary

Nearly every design decision in a GA is one of these, and they are what you tune when it does not
work:

- **Population size.** Too small and it converges on the first decent idea it finds; too large and
  each generation costs more than it is worth.
- **Representation.** Classically a string over a fixed alphabet, the way DNA is a string over
  ACGT. It can also be a vector of real numbers, or — in genetic programming — a program.
- **The mixing number**, how many parents make a child. Two is usual. **One is stochastic beam
  search**, which is a useful thing to notice. More than two is rare in nature and easy on a
  computer.
- **Selection**, how parents are picked. Proportional to fitness, or by tournament, or by rank.
- **Recombination**, how two parents are combined. The next section.
- **Mutation rate**, how often a child is randomly perturbed after being formed.
- **The makeup of the next generation** — only the new children, or some of the old best as well.

## Fitness

The one function you must supply, and the one that decides whether any of this works. It maps an
individual to a number, higher being better.

Two practical points:

- **It is evaluated constantly** — population size times generations, at least. If it is expensive,
  nothing else you tune will matter.
- **Make it graded, not binary.** A fitness that answers "is this the answer" gives the search
  nothing to climb. One that answers "how close is this" gives it a slope. Counting matching
  characters against a target string is a good fitness; comparing the whole string for equality is
  a useless one.

For 8-queens the standard choice is the number of **non-attacking pairs** of queens, which is
`8 x 7 / 2 = 28` for a solution.

## Selection

**Fitness-proportionate** (roulette wheel) — the chance of being chosen is the individual's share
of total fitness. Simple, and it has two well-known faults: one very fit individual can dominate
the whole population early, and once everyone is nearly equal the pressure vanishes. It also needs
non-negative fitness, so a fitness that can go negative must be shifted first.

**Tournament** — pick `k` at random, take the best of them. Cheap, needs no normalising, and `k`
gives direct control over how hard the selection pushes.

**Rank** — sort the population and select on position rather than score. Immune to the scale of
the fitness function, so one outlier cannot take over.

## Crossover

Given two parents, produce children that inherit from both. A child is usually no better than its
better parent: crossover recombines rather than improves, and it pays off over generations, not in
one step.

**One-point.** Pick a position, and swap everything after it.

    Crossover point: 4
    Parent 1: 1101|0110        Child 1: 1101|1011
    Parent 2: 1001|1011        Child 2: 1001|0110

Note what the second child is: the _first_ part of parent 2 followed by the _second_ part of
parent 1. Genes must stay in the positions they came from. Concatenating the tail of one parent to
the head of the other rotates everything, which destroys exactly the structure crossover is meant
to preserve.

**Two-point.** Two positions, swap the middle. Less biased toward keeping the ends together.

    Crossover points: 2 and 5
    Parent 1: 11|011|010       Child 1: 11|101|010
    Parent 2: 10|101|110       Child 2: 10|011|110

**Uniform.** A random mask decides, per position, which parent contributes.

    Parent 1:  1 1 0 1 1 0 1 0
    Parent 2:  0 0 1 1 0 1 0 1
    Mask:      1 0 1 0 1 1 0 0
    Child 1:   1 0 0 1 1 0 0 1

Position-independent, and the most disruptive — it will happily break up a good run of genes.

**Arithmetic.** For real-valued genes, a child can be a blend rather than a choice:

    child = a * mother + (1 - a) * father          for some a in [0, 1]

With `a = 0.5` this is the midpoint of the two parents. It keeps children inside the region the
population already covers, which is stable but explores poorly on its own — pair it with a
mutation that can leave that region.

### Crossover for permutations

Where every value must appear exactly once — a tour, a schedule, an ordering — all of the above
produce invalid children: cut `1 2 3 4` and `4 3 2 1` anywhere and you get repeats and omissions.
Two standard answers.

**Partially-mapped (PMX).** Swap a segment, then repair the duplicates outside it using the
mapping the swap implies.

    Parent 1: 1 2 3|4 5 6|7 8 9
    Parent 2: 4 5 2|1 8 7|6 9 3
    Mapping from the swapped segments: 4<->1, 5<->8, 6<->7

    Child 1:  4 2 3|1 8 7|6 5 9
    Child 2:  1 8 2|4 5 6|7 9 3

Child 1 takes parent 2's middle. Outside it, each of parent 1's values is copied unless it already
appears in the segment — position 0 wanted a 1, which is now in the middle, so the mapping sends
it to 4.

**Cycle (CX).** Follow cycles of positions between the parents; take each whole cycle from one
parent or the other. Every position then holds a value one of its parents had _there_.

    Parent 1: 1 2 3 4 5 6 7 8
    Parent 2: 2 4 6 8 7 5 3 1

    Cycle 1: 1 -> 2 -> 4 -> 8 -> back to 1     (positions 0, 1, 3, 7)
    Cycle 2: 3 -> 6 -> 5 -> 7 -> back to 3     (positions 2, 4, 5, 6)

    Child 1:  1 2 6 4 7 5 3 8      cycle 1 from parent 1, cycle 2 from parent 2
    Child 2:  2 4 3 8 5 6 7 1      the other way round

Both keep values close to the positions they had in a parent. When the problem cares that B
follows A rather than that B is third, as a tour does, use order crossover, shown in the homework
section below.

### Choosing one

| encoding                 | use                           |
| ------------------------ | ----------------------------- |
| binary or fixed alphabet | one-point, two-point, uniform |
| permutation              | PMX, cycle                    |
| real-valued              | arithmetic                    |

Then, within those: one-point is cheapest and most position-biased; two-point removes the bias
toward keeping the ends together; uniform removes positional bias entirely at the cost of breaking
up every block. Roughly, the more disruptive the operator, the more exploration and the less
preservation of whatever the population has already found.

## Mutation

Crossover can only shuffle what is already in the population. Once every individual has a `0` in
some position, no amount of crossover will ever produce a `1` there. Mutation is what keeps that
from being permanent.

- **Bit flip** — each bit flips with some small probability. The usual starting rate is `1/L` for a
  string of length `L`, so about one change per child.
- **Random resetting** — the same for non-binary alphabets: replace a gene with a random legal value.
- **Swap** — exchange two positions. Keeps permutations valid.
- **Scramble** — shuffle a random slice.
- **Inversion** — reverse a random slice.
- **Gaussian** — for real-valued genes, add a small normally-distributed nudge.

### Choosing a rate

| encoding    | starting point                   |
| ----------- | -------------------------------- |
| binary      | `1/L`, for chromosome length `L` |
| real-valued | `1/n`, for `n` variables         |
| permutation | lower, around 0.01 to 0.05       |

The rate is the tuning knob that matters most after population size. Too low and the population
converges and stops moving; too high and it stops being a search at all and becomes a random walk.

**Adaptive mutation** ties the rate to how varied the population still is:

    if diversity is low:  raise the mutation rate
    else:                 lower it

This is a direct attack on premature convergence — when everyone starts looking the same, push
harder. It needs a diversity measure, which for strings can be as simple as the average number of
positions on which two random individuals differ.

Whatever you choose, **mutation must leave the individual valid**. Flipping a gene in a permutation
gives a duplicate; that is why permutations mutate by swapping, scrambling or inverting rather
than by resetting.

## Elitism and culling

**Elitism** — copy the best few individuals into the next generation untouched. This guarantees
the best fitness never gets worse, which is worth having, since crossover and mutation are quite
capable of losing the best solution found so far. Keep it small; too much elitism collapses
diversity.

**Culling** — discard everything below a threshold before selecting. Cheaper generations, at the
cost of diversity.

The two pull in opposite directions from mutation. Elitism and culling push toward converging,
mutation pushes toward exploring, and a GA that is not working is usually badly balanced between
them.

**Generational or steady-state.** A _generational_ GA replaces the whole population at once, which
is what the notebook's engine does. A _steady-state_ GA produces one or two children per step and
puts them straight back into the population, displacing the worst. Steady-state converges faster
and keeps less diversity, and it is the easier of the two to run with a population that is
expensive to evaluate.

## When a genetic algorithm is the right tool

| use one when                                    | avoid it when                              |
| ----------------------------------------------- | ------------------------------------------ |
| the search space is large and poorly understood | a better method applies, e.g. a CSP solver |
| fitness is meaningful but has no gradient       | fitness is expensive to evaluate           |
| good partial solutions combine into good ones   | solutions do not split into reusable parts |
| a good answer is enough                         | you need a proof of optimality             |

## Compared with the local search we already have

|                        | keeps          | moves by                             | can it get stuck                      |
| ---------------------- | -------------- | ------------------------------------ | ------------------------------------- |
| Hill climbing          | one state      | best neighbour                       | yes, at a local maximum               |
| Simulated annealing    | one state      | random neighbour, sometimes worse    | escapes, given a slow enough schedule |
| Min-conflicts          | one assignment | repairing a conflicted variable      | yes, needs restarts                   |
| Stochastic beam search | k states       | successors of each                   | diversity collapses                   |
| **Genetic algorithm**  | k states       | **combining pairs**, then perturbing | converges early without mutation      |

Read the last column. Every one of these has the same failure — settling somewhere that is not the
answer — and every technique in this week's notebook is a different way of paying to avoid it.

## The homework: travelling salesman

The path does not return to its start, so `n` cities make `n - 1` edges. Counting the edge back to
the start puts UK12's optimum of 1595.7385 out of reach.

**Representation.** An individual is a permutation of the city indices. Build the distance matrix
once, and a tour's length is `n - 1` lookups. Shorter is better, and tournament selection only
compares, so it can minimise the length directly.

**Crossover: order crossover (OX).** A tour cares which city follows which, not which position a
city holds. OX keeps a slice of one parent in place and fills the gaps with the remaining cities in
the order the other parent visits them:

    Parent 1: 1 2 3|4 5 6|7 8 9
    Parent 2: 4 5 2|1 8 7|6 9 3
    Child:    2 1 8|4 5 6|7 9 3      2 1 8 7 9 3 is parent 2 without 4, 5 and 6

PMX and cycle crossover also work, but they reach the optimum less often.

**Mutation: inversion.** Reverse a random segment. It replaces only two edges, where a swap replaces
up to four, so it disturbs the tour least. Give each child one inversion with a probability around
0.3; a per-gene rate makes little sense for an ordering. With OX, inversion, population 200 and
`k = 3`, about 9 runs in 10 find the UK12 optimum, in well under a second each.

**Random cities, `N ≤ 100`.** There is no known optimum, so compare against a simple 2-opt local
search: keep reversing any segment that shortens the path until none does. Starting half the
population from nearest-neighbour tours (always go to the closest unvisited city) gets the GA close
to that yardstick quickly; keep the other half random for diversity.

**Output and stopping.** With elitism the best length never gets worse, so the values you print are
non-increasing. Compute the final length from the route you print. Stop after some number of
generations without improvement rather than after a fixed count.

## The homework: knapsack

**Representation.** A bit string, one gene per item: 1 means the item is taken. Use uniform
crossover and a bit flip at about `1/N` per gene. The items' order in the file means nothing, so
one-point crossover has no neighbouring blocks to preserve.

**The constraint is the whole difficulty.** In the long test set a random string with half the bits
set is about 47 times over the capacity, so almost every individual is overweight. Scoring those as
0 gives selection nothing to choose between, and the search never starts. A penalty for excess
weight works only if it is tuned to the data.

**Repair instead.** Sort the items once by value per unit of weight. Before scoring an individual:

    drop: remove its items with the worst ratio until it fits
    fill: add the items it lacks, best ratio first, whenever one still fits

Write the repaired string back into the individual so its children inherit it. The fitness is then
just the total value. The fill alone is the greedy algorithm, which is not optimal: it gets 5017 on
the long set, whose optimum is 5119. The GA does the rest.

**The large instance.** Sort once at the start, and keep each individual's weight and value as you
repair it, so a child costs `O(N)`. On `inst_10000.txt` a population of 200 to 300 usually reaches
the optimum and otherwise ends within a few units of it.

**Checking your answer.** On your own data, compute the true optimum with the `O(N · M)` dynamic
programme: for each item, for each capacity `c` from `M` down to the item's weight,
`best[c] = max(best[c], best[c - weight] + value)`.
