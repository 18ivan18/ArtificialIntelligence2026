# Constraint Satisfaction Problems

For the last two weeks a state was a black box. The search knew how to ask three questions of it —
is this the goal, what can I do from here, what does that cost — and nothing else. Every problem
looked the same from the outside, so the only thing an algorithm could do was try actions and see.

A constraint satisfaction problem opens the box. A state is no longer opaque: it is an assignment
of values to named variables, and the goal test is not a function you call but a set of
**constraints** you can read. Once the algorithm can see that structure, it can do things no search
in week 1 or 2 could: notice that a variable has only one value left, or that an assignment has
already made some later variable impossible, and rule out whole regions of the space without ever
visiting them.

## Basic Concepts

- Variables, domains, constraints
- The constraint graph
- Backtracking search
- Variable and value ordering: MRV, degree, LCV
- Inference: forward checking and arc consistency
- Local search: min-conflicts
- The structure of the problem

## Defining a CSP

Three things:

- a set of **variables** `X1 ... Xn`
- a **domain** `Di` for each one — the values it is allowed to take
- a set of **constraints**, each naming some variables and saying which combinations of their
  values are allowed

An **assignment** gives values to some of the variables. It is **consistent** if it violates no
constraint, **complete** if every variable has a value, and a **solution** if it is both.

That is the whole formalism, and its narrowness is the point. Because every problem is stated the
same way, one solver handles map colouring, timetabling, Sudoku and circuit layout without knowing
what any of them mean.

## The constraint graph

When every constraint involves at most two variables, the problem is a graph: one node per
variable, one edge per constraint. This is worth drawing before writing any code, because it makes
the shape of the problem visible — and the shape decides which algorithm to use.

Some useful vocabulary:

- **Unary** constraint: one variable. `SA != green`. These just shrink a domain, once, at the start.
- **Binary** constraint: two variables. `WA != NT`. Everything below assumes these.
- **Global** constraint: any number of variables. `alldiff(X1..Xn)` is the common one, and it has
  specialised algorithms that beat decomposing it into pairs.

A constraint with more than two variables can always be rewritten as binary ones by introducing
extra variables, so nothing is lost by building a solver that only understands pairs.

## The problems

**Map colouring.** Colour each region so no two neighbours share a colour. Variables are regions,
domains are colours, constraints are `A != B` for every shared border. The Australia map in the
notebook is the standard example, and it has a detail worth noticing: **Tasmania is an island**, so
it borders nothing. It is a variable with no constraints at all, and any colour will do.

**Sudoku.** Variables are the 81 squares, domains are 1 to 9, and there are 27 `alldiff`
constraints — one per row, column and box. Given squares are variables whose domain has one value.
Most published puzzles are solved by constraint propagation alone, without any search at all.

**N-Queens.** One variable per column, the domain being which row that column's queen sits in. The
constraint is that no two queens share a row or a diagonal. This is the one where backtracking is
the wrong tool and local search is the right one — see below.

**Job-shop scheduling.** Variables are tasks, values are start times. Two kinds of constraint:

- **Precedence** — if task `T1` takes `d1` and must finish before `T2` starts, then `T1 + d1 <= T2`.
  Putting a wheel on before the hubcap is this.
- **Disjunctive** — two tasks needing the same tool cannot overlap, so _either_ `T1 + d1 <= T2`
  _or_ `T2 + d2 <= T1`. That mixes arithmetic and logic, but it still reduces to a set of allowed
  pairs, so it is an ordinary binary constraint.

This is the example that shows domains need not be small symbolic sets. Values are integers,
domains are time windows, and an unbounded schedule has to be bounded before search can start.

## Backtracking search

Depth-first search with one observation: **the order in which you assign variables does not
matter.** A CSP solution is a set of assignments, not a sequence, so at every node you may pick one
unassigned variable and consider only its values. That alone cuts the branching factor from
`n × d` to `d`.

    assign a value to one unassigned variable
    if it is consistent with what is already assigned, recurse
    if the recursion fails, undo the assignment and try the next value
    if no value works, fail back to the caller

That is the whole algorithm, and on its own it is slow. Everything that makes CSP solvers good is
what you add to it.

## Variable and value ordering

Which variable to assign next, and which value to try first, change the running time by orders of
magnitude. They pull in opposite directions, which is the thing to understand.

**Minimum remaining values (MRV)** — choose the variable with the fewest legal values left. Also
called _most constrained variable_, or _fail-first_: if a variable is going to fail, it is cheaper
to find out now, near the top of the tree, than after assigning ten other variables beneath it.

**Degree heuristic** — choose the variable involved in the most constraints on unassigned
variables. Useful mainly as a tie-breaker for MRV, and as the opening move when nothing is assigned
yet and every domain is still full.

**Least constraining value (LCV)** — having picked a variable, try the value that rules out the
fewest choices for its neighbours. Note the reversal: fail-first for variables, fail-last for
values. It is not a contradiction. You need every variable to work out, so probe the fragile one
early; but you only need one value to work out, so try the flexible one first.

## Inference

Search tries things. Inference works out what cannot be true without trying anything, and the two
interleave: assign a variable, propagate the consequences, shrink the domains, assign again.

**Node consistency** — a variable is node-consistent when every value in its domain satisfies its
unary constraints. Enforce it once at the start and unary constraints never need thinking about
again.

**Forward checking** — whenever you assign a variable, delete that value from the domains of its
unassigned neighbours. If a domain empties, this branch is dead, so backtrack immediately. Cheap,
and it catches failure much earlier than checking consistency at each step would.

**Arc consistency** — stronger. An arc `X -> Y` is consistent when, for every value in `X`'s
domain, there is _some_ value in `Y`'s domain that satisfies the constraint. If a value in `X` has
no partner, nothing can use it, so delete it. Deleting it may break another arc pointing into `X`,
so those go back in the queue. That is **AC-3**: keep a queue of arcs, and re-examine the
neighbours of any domain you shrink.

Forward checking is what you get if you only ever check the arcs pointing at the variable you just
assigned. Arc consistency keeps going until nothing changes, and catches failures forward checking
misses.

**MAC** — maintaining arc consistency: run AC-3 after every assignment, not just once at the start.

## Local search: min-conflicts

Backtracking builds a solution from nothing, one variable at a time. Local search starts with a
complete assignment — wrong, but complete — and repairs it.

    pick a variable that is in conflict
    give it the value that leaves it in conflict with the fewest other variables
    repeat

That is **min-conflicts**, and it is startlingly effective. It solves the million-queens problem in
about fifty steps, more or less independently of `n`. Backtracking on that is hopeless.

The reason is the shape of the space, not the cleverness of the rule. N-Queens has solutions spread
densely through the state space, and the conflict count is a smooth guide to them — so from almost
anywhere, walking downhill works. That generalises: local search wins when solutions are dense and
the constraints are loose. It fails on tightly constrained problems where the solutions are rare
and isolated, and it can never prove a problem unsolvable — it just keeps trying.

It has hill climbing's weakness, too: a local minimum where every move makes things worse. The
usual fixes are the usual ones — random restarts, or allowing sideways moves.

### The homework: min-conflicts for N-Queens

The notebook's `min_conflicts` is the idea, but it asks the generic `CSP` for conflicts, and in
N-Queens every column is a neighbour of every other, so a single repair costs `O(n²)`. The homework
has to do `n = 10,000` in under a second, which needs the same algorithm with a representation built
for the problem.

**Representation.** One queen per column, so queen `i` lives in column `i` and the column constraint
holds by construction. Keep four arrays:

| array    |   length | holds                                                              |
| -------- | -------: | ------------------------------------------------------------------ |
| `queens` |      `n` | the row of the queen in each column                                |
| `rows`   |      `n` | how many queens are in each row                                    |
| `diag1`  | `2n - 1` | how many queens are on each diagonal where `row + col` is constant |
| `diag2`  | `2n - 1` | how many queens are on each diagonal where `row - col` is constant |

An `n × n` board has `2n - 1` diagonals in each direction. Square `(row, col)` is on `diag1[row + col]`
and `diag2[n - 1 - row + col]`; the `n - 1` shifts `row - col` from `-(n-1)..n-1` to `0..2n-2`.

**Conflicts of a square** are then three lookups instead of a scan of the board:

    conflicts(row, col) = rows[row] + diag1[row + col] + diag2[n - 1 - row + col]

minus 3 if the queen being considered is already standing on that square, since it would be counting
itself once on each line. Moving a queen is six counter updates: decrement its old row and
diagonals, increment the new ones.

**The quantity to drive to zero**, `h`, is the number of attacking pairs. Every pair of queens on a
line attacks, so a line holding `k` queens contributes `k(k-1)/2`, and `h` is that summed over all
rows and diagonals (no column ever holds two). You never need `h` itself to run the algorithm, only
to report it or to check a solution.

**The algorithm.**

    place each queen on some row (randomly, or greedily: see below), filling the counters
    until no queen is in conflict:
        pick a random queen that is in conflict
        take her off the board (decrement her row and diagonal counters)
        count the conflicts of every row in her column
        put her back on one of the rows with the fewest, chosen at random; update the counters

Each repair is `O(n)`: one pass over the rows of a single column.

Details that decide whether it finishes:

- **Break ties at random.** Always choosing the first best row lets the search walk the same loop
  for ever. The random choice is not decoration.
- **Staying put is allowed.** If the queen's current row is among the best, she may stay; that is a
  sideways move, and it is fine.
- **A greedy start saves most of the work.** Placing the queens one column at a time, each on a row
  with the fewest conflicts with those already placed, leaves only a handful of conflicts. The cost
  is one `O(n²)` pass. For a very large `n`, sampling a few rows per column instead of all of them
  keeps the start cheap.
- **Finding a conflicted queen** by rescanning all `n` columns every step is `O(n)`, the same as the
  repair itself, so it is acceptable. Keeping a list of conflicted columns is faster but has to be
  updated whenever a move changes someone else's conflicts.
- **Restart if it stalls.** If the conflicts have not reached zero after some budget, say a few
  times `n` repairs, start again from a new placement rather than continuing.
- **`n = 2` and `n = 3` have no solution**, and min-conflicts cannot prove that; it would simply never
  stop. Answer `-1` for them directly. Every other `n`, including 1, has a solution.

The week's page on the course site steps through this on a board, with the three counter arrays
shown as they change, and runs it up to `n = 10,000` in the browser.

## The structure of the problem

Sometimes the constraint graph itself tells you the answer.

**Independent subproblems.** If the graph has disconnected components, they are separate problems.
Tasmania is a component of size one, which is why it takes no work at all.

**Tree-structured CSPs.** If the constraint graph is a tree, the problem is solvable in **O(n d²)** —
linear, no backtracking needed. Order the variables so each one's parent comes first, make every
arc consistent from the leaves up, then assign from the root down. Anything tree-shaped is easy.

**Cutset conditioning.** Most graphs are not trees but are close. Find a small set of variables — a
**cycle cutset** — whose removal leaves a tree. Try every combination of values for the cutset, and
for each one solve the remaining tree in linear time. Cost is exponential in the size of the cutset
only. On the Australia map, removing South Australia leaves a tree, so a cutset of one is enough.

## Comparing the approaches

|                         | Builds a solution                    | Complete? | Good when                            |
| ----------------------- | ------------------------------------ | --------- | ------------------------------------ |
| Backtracking            | from nothing, one variable at a time | yes       | always works, rarely fast alone      |
| + MRV / degree / LCV    | same                                 | yes       | almost always worth it, costs little |
| + forward checking      | same                                 | yes       | cheap, catches dead branches early   |
| + arc consistency (MAC) | same                                 | yes       | tight constraints, large domains     |
| Min-conflicts           | by repairing a complete guess        | **no**    | solutions dense, constraints loose   |
| Tree / cutset methods   | exploits the graph shape             | yes       | the graph is a tree or nearly one    |

### What that is worth, measured

On the Sudoku in the notebook, with nothing but the first-unassigned variable and values in any
order:

|                        | assignments |    time |
| ---------------------- | ----------: | ------: |
| plain backtracking     |  14,740,819 |    61 s |
| MRV + forward checking |          81 | 0.005 s |

Eighty-one assignments for eighty-one squares means it never backtracked at all. The two
techniques are perhaps thirty lines between them, and they are the difference between a minute and
five milliseconds on a puzzle a person solves over coffee.

Two things to carry away:

- **Structure is the whole point.** Week 1 and 2 could only try actions. A CSP solver can read the
  constraints and reason about them, and every technique here is a way of spending time thinking
  instead of searching.
- **Which algorithm wins is a property of the problem, not of the algorithm.** Backtracking with
  MAC is right for Sudoku and hopeless for a million queens; min-conflicts is the reverse. Look at
  the constraint graph before choosing.
