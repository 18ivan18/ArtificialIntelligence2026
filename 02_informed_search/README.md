# Informed (Heuristic) Search

Last week's algorithms knew only the problem definition: which states exist, which actions apply,
what each one costs. None of them had any idea which direction the goal was in, so they explored
outwards in every direction at once.

This week every node carries an estimate of how far it still has to go, and that estimate decides
what gets expanded next.

## Basic Concepts

- Heuristic function
- Best-First Search as a general framework
- Greedy Best-First Search
- A\*
- Admissibility and consistency
- Beam Search
- Hill Climbing and local maxima
- Weighted A\*
- Iterative Deepening A\*
- Recursive Best-First Search
- Memory-Bounded A\*
- Simulated Annealing

## The heuristic function

A **heuristic** is a function

    h(n) = estimated cost of the cheapest path from the state at node n to a goal state

with one hard requirement: `h(goal) = 0`.

The important thing about `h` is where it comes from. It cannot be computed from the problem
definition — the successor function and the action costs say nothing about which way the goal
lies. It is extra knowledge about the world that you supply. For route finding on a map the
obvious choice is the **straight-line distance** to the destination, and it takes knowing
something about roads and geography to believe that straight-line distance is a useful guide to
driving distance at all.

## Best-First Search

Everything in both weeks is the same loop: keep a frontier in a priority queue, pop the node with
the smallest **evaluation function** `f(n)`, expand it, push its children. The only thing that
changes between the algorithms is `f`.

| algorithm                    | f(n)          | reads as                      |
| ---------------------------- | ------------- | ----------------------------- |
| Uniform-Cost Search (week 1) | `g(n)`        | cheapest so far               |
| Greedy Best-First Search     | `h(n)`        | looks closest to the goal     |
| A\*                          | `g(n) + h(n)` | cheapest estimated total trip |

Seen this way, uniform-cost search is just A\* with a heuristic that is always 0 — it is the
special case where you know nothing. That is worth holding on to: the heuristic is the _only_
difference between the search that wanders outwards in all directions and the one that heads for
the goal.

## The algorithms

In the costs below, `b` is the branching factor, `d` the depth of the shallowest goal, `m` the
maximum depth of the search space, `k` a beam width, `C*` the optimal cost and `ε` the smallest
action cost.

### Greedy Best-First Search

`f(n) = h(n)`. Expand whatever looks closest to the goal and pay no attention to what the trip has
cost so far.

- **Complete?** **Yes** in finite state spaces, provided you check for repeated states. **No** in
  infinite ones — it can follow an infinite branch that always looks promising.
- **Cost optimal?** **No.**
- **Time?** **O(b^m)** in the worst case; with a good heuristic, far better.
- **Space?** **O(b^m)** — it keeps the whole frontier, so the space cost is the time cost.

On the Romania map with Bucharest as the goal, greedy search from Arad goes to Sibiu, then to
Fagaras — because Fagaras _looks_ closer than Rimnicu Vilcea — and reaches Bucharest having
driven 450. The best route is 418. Greedy search took the bait: the first step of the better route
looks worse, so it never tried it. That is the whole failure mode in one example.

### A\*

`f(n) = g(n) + h(n)`. The cost already paid plus the estimated cost still to come, so `f(n)` is an
estimate of the total cost of the cheapest solution _through_ `n`.

- **Complete?** **Yes.**
- **Cost optimal?** **Yes**, provided `h` is admissible (for tree-like search) or consistent (for
  graph search).
- **Time?** **O(b^d)** in the worst case, and sharply better than that with a good heuristic.
- **Space?** **O(b^d)** — it keeps every generated node, and in practice this is what stops A\*
  long before time does.

A\* expands every node with `f(n) < C*` and no node with `f(n) > C*`, where `C*` is the optimal
cost. Nodes with `f(n) = C*` may or may not be expanded. So the quality of the heuristic decides
how much of the map you look at: the closer `h` gets to the true remaining cost, the thinner the
band of nodes with `f(n) < C*` becomes.

### Weighted A\*

`f(n) = g(n) + W · h(n)`, with `W >= 1`.

At `W = 1` this is A\*. As `W` rises the estimate counts for more and the cost already paid counts
for less, so the search leans harder towards the goal, expands fewer nodes, and slides towards
greedy best-first — which is the `W → ∞` end of the same dial.

- **Complete?** **Yes.**
- **Cost optimal?** **No** — but never worse than **W times** the optimal cost.
- **Time and space?** Both fall as `W` rises, often steeply.

That bound is what makes it worth knowing. Giving up optimality is easy; giving it up by a stated
factor is a real engineering result — "at most 20% off, in a third of the time" is a sentence you
can defend to someone.

On our map the interesting part is the middle of the dial. `W = 1.1` expands **4** nodes where A\*
expands 5, and still returns the optimal 418. Push on to `W = 1.2` and it drops to 3 nodes and 450.
A little inadmissibility bought a real saving before it cost anything at all.

### Beam Search

Best-first search with a fixed-size frontier: keep only the `k` best nodes by `f(n)` and throw the
rest away.

- **Complete?** **No** — the only path to the goal can be discarded.
- **Cost optimal?** **No.**
- **Time?** **O(b · m · k)** — `k` nodes expanded per level, `b` successors each, over `m` levels.
- **Space?** **O(b · k)** — the beam itself, plus the successors it is picking from. Constant in
  the size of the problem, which is the entire point.

`k = 1` is hill climbing, below. `k = ∞` is ordinary best-first search. Everything useful is in
between, and choosing `k` is trading the guarantee of finding an answer against being able to run
at all.

### Hill Climbing

Beam search with `k = 1`. Look at the successors of the current node, move to the best one, forget
everything else. There is no frontier and no way back.

- **Complete?** **No.**
- **Cost optimal?** **No.**
- **Time?** **O(bm)**
- **Space?** **O(b)** — it never holds more than one node's successors, and that is the whole
  appeal. Every other algorithm here runs out of memory before it runs out of time.

It is sometimes described as climbing Everest in thick fog with amnesia. You can feel which way is
up; that is all you know.

Hill climbing is not really a path-finding algorithm at all. It is an **optimisation** algorithm
that we are pointing at a graph, and the three ways it fails all have names:

- **Local maximum** — every neighbour looks worse, but you are not at the best peak. Hill climbing
  stops and reports where it happens to be standing.
- **Shoulder / plateau** — a flat region where every neighbour looks equal. Refusing sideways
  moves means stopping on a shoulder you could have walked across; allowing them means risking an
  endless walk on a genuinely flat plateau.
- **Ridge** — a line of local maxima that the available moves cannot climb directly, so every
  single step looks like a step down.

The notebook draws this landscape, and it is worth having the picture in your head: hill climbing
sees only the ground immediately under its feet.

**It fails on our map, and only in one place.** Run over all 20 cities, Mehadia is the single
local minimum of `h`: it sits at 241, and both roads out of it — Lugoj at 244 and Drobeta at 242 —
look like steps backwards. So hill climbing from Timisoara goes Timisoara → Lugoj → Mehadia and
stops, 181 miles in and 434 miles from Bucharest by road. From Arad it reaches Bucharest quite
happily. One starting city is the difference between a working algorithm and a stranded one, and
nothing in the code changed.

### Iterative Deepening A\*

A\* runs out of memory long before it runs out of time. IDA\* is the same trade week 1 made when it
replaced breadth-first search with iterative deepening: keep the guarantees, give up the queue.

Instead of a depth limit, iterate on an **`f`-limit**. Do a depth-first search that prunes any node
with `f(n) > limit`. If the search fails, the next limit is the **smallest `f` value that exceeded
the current one**, and you go round again.

- **Complete?** **Yes.**
- **Cost optimal?** **Yes**, with an admissible heuristic.
- **Time?** **O(b^d)**, and it re-expands the upper levels on every pass.
- **Space?** **O(bd)** — linear, and that is the entire reason it exists.

The catch is worth stating plainly, because it decides whether IDA\* helps you at all: **it is only
efficient when `f` takes few distinct values.** Each iteration raises the limit to the next `f`, so
if every node has a different `f`, each pass adds roughly one node and you re-run the whole search
to gain it.

That is exactly our situation. Road distances like 71, 75, 118 and 140 make almost every node's `f`
unique, so IDA\* on this map is close to the worst case for it. On the sliding-tile puzzle in the
homework, where every move costs 1 and Manhattan distance is an integer, `f` takes very few values
and IDA\* is excellent. Same algorithm, opposite verdict, decided entirely by the shape of the
cost function.

### Memory-Bounded A\*

Run A\* until memory is full, then make room: drop the worst leaf in the frontier and continue.

Done naively, that is beam search with `f = g + h`, and it loses the same guarantees. The real
version (**SMA\***) adds one idea that buys most of them back: when a node is dropped, its `f`
value is **backed up into its parent**, so the parent remembers how good the best thing beneath it
was. If every other option later turns out worse, the parent can regenerate the child it forgot.
Nothing is lost permanently — only the memory holding it.

- **Complete?** **Yes**, if the shallowest goal is at a depth that fits in memory.
- **Cost optimal?** **Yes**, if the optimal path fits in memory. Otherwise it returns the best that
  does fit.
- **Space?** Whatever you give it, by construction.

The notebook's exercise is the simple version — A\* with the frontier capped. Backing up the
forgotten values is the part worth understanding even if you do not implement it.

### Recursive Best-First Search

Best-first search on linear memory, and a different bargain from IDA\*.

RBFS walks the route A\* would walk, but it only ever holds the current path and the siblings of
each node on it. When the route it is following starts to look worse than something it passed
higher up, it unwinds — and as it unwinds it **backs up** the best `f` it saw below into the node
it retreated through.

That number is the point. IDA\* forgets everything between rounds, so every round rediscovers what
the last one knew. RBFS forgets the nodes too, but keeps one value per node it backtracks past, and
that is enough to stop it re-exploring a subtree it has already judged.

- **Complete?** **Yes.**
- **Cost optimal?** **Yes**, with an admissible heuristic.
- **Time?** Hard to state usefully — it depends on the heuristic and on how often the search has to
  switch between subtrees, since each switch costs a regeneration.
- **Space?** **O(bd)** — linear, like IDA\*.

The difference shows up immediately. Arad to Bucharest: A\* expands 5 nodes, **RBFS 6**, IDA\* 20.
RBFS gets nearly A\*'s node count on IDA\*'s memory, which is the best of both on this map.

## Beyond hill climbing: Simulated Annealing

Theory only this week — there is no exercise for it — but it is the standard answer to the problem
hill climbing has, and it is a genuinely lovely idea.

Hill climbing gets stuck because it never accepts a worse move. Simulated annealing sometimes
does. At each step, pick a random neighbour:

- if it is better, always take it
- if it is worse by `ΔE`, take it anyway with probability `exp(-ΔE / T)`

`T` is the **temperature**, and it falls over time on a **cooling schedule**. When `T` is high,
almost any move is accepted and the search wanders more or less at random. As `T` falls, bad moves
get rarer, and at `T` near zero it is plain hill climbing again.

The name comes from metallurgy: heat a metal and cool it slowly, and the atoms settle into a
low-energy crystal; cool it fast and they freeze into whatever disordered arrangement they happened
to be in. The search does the same thing — early on it is free to leave a local maximum, and by the
end it is committed to climbing whichever hill it finished on.

If the cooling is slow enough, it finds the global optimum with probability approaching 1. The
practical difficulty is that "slow enough" is usually slower than you can afford, so the real work
is in choosing a schedule.

Compare it with Mehadia. Hill climbing walks in and stops, because both exits look worse.
Simulated annealing, while `T` is still high, will take one of those uphill steps anyway and get
out.

## Admissibility and consistency

Two properties of a heuristic, and the reason A\* can be trusted.

**Admissible** — `h` never overestimates:

    0 <= h(n) <= h*(n)      where h*(n) is the true cheapest cost from n to a goal

An admissible heuristic is optimistic: it always thinks the goal is at least as close as it
really is. Straight-line distance is admissible for road travel because the shortest path between
two points is a straight line, and no road can beat it.

**Consistent** (or monotone) — the triangle inequality holds at every step:

    h(n) <= cost(n, a, n') + h(n')

for every node `n` and every successor `n'` it can reach with action `a`.

Consistency is the stronger property: **every consistent heuristic is admissible**, but not the
other way round. It matters because with a consistent `h`, `f` never decreases along a path, and
the first time A\* expands a node it has already found the cheapest route to it. That is exactly
the condition that makes it safe to discard repeated states — so graph search needs consistency,
while tree-like search only needs admissibility.

The straight-line distances used in the notebook are both. You can check this yourself, and the
notebook does: no city's `h` exceeds its true road distance to Bucharest, and the inequality holds
across all 23 roads.

## Comparing them

| Algorithm          | Complete? | Cost optimal?         | Time                         | Space             | What it holds                  |
| ------------------ | --------- | --------------------- | ---------------------------- | ----------------- | ------------------------------ |
| Uniform-Cost       | Yes       | Yes                   | O(b^(1+⌊C\*/ε⌋))             | O(b^(1+⌊C\*/ε⌋))  | priority queue                 |
| Greedy Best-First  | Yes ¹     | No                    | O(b^m)                       | O(b^m)            | priority queue                 |
| A\*                | Yes       | Yes ²                 | O(b^d)                       | O(b^d)            | priority queue                 |
| Weighted A\* (W)   | Yes       | No, but ≤ W × optimal | falls as W rises             | falls as W rises  | priority queue                 |
| IDA\*              | Yes       | Yes ²                 | O(b^d), with repeats         | **O(bd)**         | a stack                        |
| RBFS               | Yes       | Yes ²                 | depends on subtree switching | **O(bd)**         | current path + siblings        |
| Memory-Bounded A\* | Yes ³     | Yes ³                 | —                            | **O(limit)**      | priority queue capped at limit |
| Beam (width k)     | No        | No                    | O(b · m · k)                 | **O(b · k)**      | priority queue capped at k     |
| Hill Climbing      | No        | No                    | O(bm)                        | **O(b)**          | the best successor, only       |

1. In finite state spaces, with repeated-state checking.
2. With an admissible heuristic for tree-like search, or a consistent one for graph search.
3. Provided the shallowest goal, and the optimal path, fit inside the limit you gave it.

Some things worth drawing out:

- **A\* with `h = 0` is uniform-cost search.** Not similar to it — the same algorithm. If your A\*
  is expanding as many nodes as UCS did, your heuristic is doing nothing.
- **A better heuristic is worth more than a better data structure.** If `h1(n) <= h2(n) <= h*(n)`
  everywhere, then `h2` **dominates** `h1` and A\* with `h2` never expands more nodes than A\* with
  `h1`. Work on the estimate before optimising the queue.
- **Greedy and A\* differ by one term.** Dropping `g(n)` is what turns a cost-optimal algorithm
  into one that can be led 32 miles astray on a 20-city map. It is a very small change in the code
  and a total change in the guarantees.
- **IDA\* and A\* answer the same question with different memory.** They return the same path; the
  difference is a stack instead of a priority queue, and paying for it in repeated work. Which one
  you want depends on whether time or memory runs out first.
- **Optimality is not all-or-nothing.** Between A\* and greedy sits weighted A\*, which gives up
  exactly as much as you ask it to and tells you the worst case in advance. Most real systems live
  somewhere on that dial rather than at either end.
- **The whole column is about memory.** Read the space column on its own: A\* keeps everything, beam
  keeps `k` and the successors it is choosing between, hill climbing keeps one node's successors. Every guarantee in the table above is paid for in memory,
  and the algorithms people actually run on large problems are the ones lower down it. IDA\* and
  RBFS are the two that keep A\*'s answer while giving up A\*'s appetite — and between them, RBFS
  is the one that also remembers what it learned.

## The map

The example this week is a simplified road map of part of Romania: 20 cities, 23 roads, distances
in miles. The goal is Bucharest, and `h` is the straight-line distance to Bucharest.

| city      | h   | city           | h   |
| --------- | --- | -------------- | --- |
| Arad      | 366 | Mehadia        | 241 |
| Bucharest | 0   | Neamt          | 234 |
| Craiova   | 160 | Oradea         | 380 |
| Drobeta   | 242 | Pitesti        | 100 |
| Eforie    | 161 | Rimnicu Vilcea | 193 |
| Fagaras   | 176 | Sibiu          | 253 |
| Giurgiu   | 77  | Timisoara      | 329 |
| Hirsova   | 151 | Urziceni       | 80  |
| Iasi      | 226 | Vaslui         | 199 |
| Lugoj     | 244 | Zerind         | 374 |

These are straight-line distances, so every one of them is below the real road distance — the
roads bend and the crow does not. How far below varies a lot, and that variation is what decides
how well A\* does from any given city:

|                | h   | by road | slack |
| -------------- | --- | ------- | ----- |
| Pitesti        | 100 | 101     | 1     |
| Rimnicu Vilcea | 193 | 198     | 5     |
| Arad           | 366 | 418     | 52    |
| Timisoara      | 329 | 536     | 207   |
| Lugoj          | 244 | 504     | 260   |

Pitesti is one mile out — about as good as a heuristic ever gets, and A\* starting there would
walk almost straight to the goal. Lugoj is 260 miles out, because the only roads out of it run
away from Bucharest before they turn back. A heuristic is only as useful as the geography it
describes.

Arad to Bucharest is the route to know. The optimal answer is **418**, through Sibiu, Rimnicu
Vilcea and Pitesti. Uniform-cost search found it last week by looking at nearly the whole map.
This week the point is to find the same answer having looked at much less of it.
