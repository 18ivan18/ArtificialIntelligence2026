# Uninformed (Blind) Search

This file holds the theory. The notebook holds only the graph, the drawing helpers and the
algorithms, so you can read here and implement there.

## Basic Concepts

- State Space
- Representation of the State Space
- Search Strategies Evaluation
- Global Search vs Local Search
- Uninformed Search vs Informed Search
- Graph and Tree Traversal

## State Space

- **State**: a representation (formulation) of the task in the process of its solution.

- **Initial State** (presented with **_S_**)
- **Intermediate State** (presented with any capital letter except **_S_** and **_G_**)
- **Goal State** (presented with **_G_**, if there are more: **_G1_**, **_G2_**, etc.)
- **Successor Function** **_(Operator)_**: obtaining one state from another

- **Path Cost**: additive; e.g., sum of distances, number of actions executed, etc.

- **State Space**: the totality of all possible states that can be obtained from a given initial
  state.

- **A _solution_**: _is a sequence of actions leading from the initial state to a goal state._

## Representation of the State Space

Graph or Tree where each state is represented with a Node and the successor function (operator) is
represented with an Edge.

When the state space can be represented as a tree, it is called a Search Tree. The _initial state_
is the _root_ of the tree. The terminal states and the _goal state_ are represented with leaves.

## Evaluating a Search Strategy

Every search strategy is evaluated along four dimensions:

- **Completeness**: is the algorithm guaranteed to find a solution when there is one, and to
  correctly report failure when there is not?
- **Cost optimality**: does it find a solution with the lowest path cost of all solutions?
- **Time complexity**: how long does it take - measured in states and actions considered.
- **Space complexity**: how much memory is needed.

We look at the **_worst case_** (Big O notation). Time and space are measured in terms of:

- **_b_** — the branching factor, the maximum number of successors of any node
- **_d_** — the depth of the shallowest goal node (the number of actions on the shallowest
  solution path)
- **_m_** — the maximum number of actions in any path (may be ∞)
- **_ℓ_** — the depth limit, for the depth-limited strategies

Note that **_d_** is the depth of the **shallowest** solution - not necessarily the cheapest one.
Those two coming apart is exactly what separates BFS from UCS below.

## Global Search vs Local Search

- **Global Search**: looks all over the state space. If necessary, all states will be traversed.

- **Local Search**: only looks at the local area, so it can only look at states in this area. If
  the solution is outside of it, it will not be found.

> Generally Local Search is not used for finding a path from state A to state B.

## Local Search

- **Local Search** in _Artificial Intelligence_ is an optimizing algorithm to find the optimal
  solution more quickly.
- **Local search algorithms** are used when we care only about a solution but not the path to a
  solution.
- **Local search** is a heuristic method for solving computationally hard optimization problems.
- **Local search** can be used on problems that can be formulated as finding a solution maximizing
  a criterion among a number of candidate solutions.
- **Local search algorithms** move from solution to solution in the space of candidate solutions
  (the search space) by applying local changes, until a solution deemed optimal is found or a time
  bound is elapsed.

## Uninformed (Blind) vs Informed (Heuristic) Search

**Uninformed Search** strategies use only the information available in the problem definition.

- _Examples: DFS, BFS, UCS, DLS, IDS._

**Informed Search** strategies have information about the goal state which helps in more efficient
searching. This information is obtained by a function (heuristic) that estimates how close a state
is to the goal state.

- _Examples: Greedy Best-First Search, A\*, Beam Search._

This week is entirely the first group. The heuristic column in the drawings stays at 0 until next
week.

## The colour protocol

Every algorithm in the notebook paints the graph as it runs, and every drawing uses the same five
colours:

| colour | meaning                                                 |
| ------ | ------------------------------------------------------- |
| white  | undiscovered — the algorithm has not seen this node yet |
| gray   | in the frontier — discovered, but not yet expanded      |
| red    | currently being expanded                                |
| blue   | expanded — finished with                                |
| green  | the goal                                                |

The frontier (older texts call it the _open list_) is the gray fringe between what you have
explored and what you have not. Watching the gray band move is the whole point of the pictures:
**the shape of that band is the difference between the algorithms.** BFS keeps it wide and shallow,
DFS keeps it narrow and deep.

## The algorithms

### Depth-First Search (DFS)

Expands the deepest node in the frontier first. Implemented with a **stack** (LIFO), or by
recursion, which uses the call stack for you.

- **Complete?** **No** — it fails in infinite-depth spaces and in spaces with loops. Modify it to
  avoid repeated states along the path and it becomes _complete in finite spaces_.
- **Cost optimal?** **No.**
- **Time?** **O(b^m)** — terrible if _m_ is much larger than _d_. But if solutions are dense, it
  may be much faster than breadth-first.
- **Space?** **O(bm)** — i.e. _linear space_, and this is the reason DFS is worth having at all.

### Breadth-First Search (BFS)

Expands the shallowest node in the frontier first. Implemented with a **queue** (FIFO).

- **Complete?** **Yes** (if _b_ is finite).
- **Cost optimal?** **Yes if every action has the same cost**; not optimal in general.
- **Time?** **O(b^d)**
- **Space?** **O(b^d)** — it keeps every node in memory, and this is what kills it first.

A detail worth stating out loud: BFS can apply the goal test when a node is **generated** rather
than when it is expanded, which saves a whole level of work.

### Depth-Limited Search (DLS)

DFS with a depth limit _ℓ_ — nodes at depth _ℓ_ are treated as having no successors.

- **Complete?** **Yes, if ℓ ≥ d.** Otherwise it returns failure even though a solution exists.
- **Cost optimal?** **No.**
- **Time?** **O(b^ℓ)**
- **Space?** **O(bℓ)**

### Iterative Deepening Search (IDS)

Run DLS with limit 0, then 1, then 2, … until a solution is found. It sounds wasteful and is not:
the nodes at the bottom level dominate the count, so re-expanding the upper levels costs little.

- **Complete?** **Yes** (if _b_ is finite).
- **Cost optimal?** **Yes if every action has the same cost.**
- **Time?** **O(b^d)** — the repeated work is a constant factor, not an extra exponent.
- **Space?** **O(bd)** — _linear space_, like DFS.

IDS is the one to use when you want BFS's guarantees but cannot afford BFS's memory. It is the
preferred uninformed method when the state space is larger than memory and the solution depth is
unknown.

### Uniform-Cost Search (UCS)

Expands the node with the lowest **path cost g(n)** first, using a **priority queue**. This is
Dijkstra's algorithm under another name — the same algorithm, arrived at from two directions.

- **Complete?** **Yes**, provided every action cost is ≥ ε for some ε > 0.
- **Cost optimal?** **Yes** — nodes are expanded in increasing order of _g(n)_.
- **Time?** **O(b^(1+⌊C\*/ε⌋))**, where _C\*_ is the cost of the optimal solution.
- **Space?** **O(b^(1+⌊C\*/ε⌋))**

When every action costs the same, UCS degenerates into BFS.

### Bidirectional Search

Run two searches at the same time — one forward from the initial state, one backward from the
goal — and hope they meet in the middle. The motivation is pure arithmetic: two trees of depth
_d_/2 are far smaller than one tree of depth _d_. With b = d = 10, b^(d/2) + b^(d/2) is about
**50,000 times less** than b^d.

- **Complete?** **Yes**, if _b_ is finite and both halves are breadth-first or uniform-cost.
- **Cost optimal?** **Yes**, under the same condition.
- **Time?** **O(b^(d/2))**
- **Space?** **O(b^(d/2))**

Two things make it harder than it looks:

- **You must be able to search backwards.** That means knowing the predecessors of a state. Our
  graph is undirected, so the same adjacency lists work in both directions and we get this for
  free — on a directed graph you would have to build the reversed edges. It also needs the goal
  to be an explicit state, not just a goal _test_.
- **Meeting is not the same as finishing.** With unit costs, the first time the two frontiers
  touch you have a shortest path. With weights you do not: the first meeting point may not lie on
  the cheapest path, so the general version keeps the best solution found so far and terminates
  only when no better one can remain.

## Comparing them

All of them side by side:

| Criterion     | Breadth-First | Uniform-Cost     | Depth-First | Depth-Limited | Iterative Deepening | Bidirectional |
| ------------- | ------------- | ---------------- | ----------- | ------------- | ------------------- | ------------- |
| Complete?     | Yes ¹         | Yes ^(1,2)       | No          | No            | Yes ¹               | Yes ^(1,4)    |
| Optimal cost? | Yes ³         | Yes              | No          | No            | Yes ³               | Yes ^(3,4)    |
| Time          | O(b^d)        | O(b^(1+⌊C\*/ε⌋)) | O(b^m)      | O(b^ℓ)        | O(b^d)              | O(b^(d/2))    |
| Space         | O(b^d)        | O(b^(1+⌊C\*/ε⌋)) | O(bm)       | O(bℓ)         | O(bd)               | O(b^(d/2))    |

1. Complete if _b_ is finite, and the state space either has a solution or is finite.
2. Complete if all action costs are ≥ ε > 0.
3. Cost-optimal if action costs are all identical.
4. If both directions are breadth-first or uniform-cost.

Two things about this table that are easy to miss:

- It describes **tree-like search**, which does not check for repeated states. For **graph search**,
  which does, DFS becomes complete on finite state spaces and both complexities are bounded by the
  size of the state space, `|V| + |E|`.
- The only row where BFS and IDS differ is **space** — and that single difference is why IDS
  exists.

- **Bidirectional is the only column that beats the exponent itself** rather than trading one
  resource for another. Everything else on this table moves work between time and space; halving
  _d_ changes the size of the problem.
