# Adversarial Search and Games

Everything so far has searched a world that sits still. The map does not move while you plan a
route; the Sudoku does not rewrite itself between assignments. In a game there is someone on the
other side of the board, and every move you consider is followed by a move chosen to hurt you.

That is the only new idea this week, and it changes one thing in the search: you can no longer
take the best branch at every level, because half the levels are not yours to choose.

## Basic Concepts

- Games as search problems
- The game tree and the minimax value
- Minimax
- Alpha-beta pruning
- Move ordering
- Evaluation functions and cutting off search

## Kinds of game

Worth naming, because each label removes or adds an algorithm:

- **Zero-sum or not.** Zero-sum means one player's gain is exactly the other's loss, so a single
  number describes the position from both sides. Everything below assumes it.
- **Perfect or imperfect information.** Perfect means every player can see the whole state — chess,
  go, nim. Imperfect means they cannot — poker, kriegspiel. Note the distinction from _complete_
  information, which is about knowing the other players' available strategies and payoffs rather
  than their actual moves.
- **Deterministic or stochastic.** Backgammon has dice, chess does not. Dice add chance nodes and
  turn minimax into expectiminimax.
- **Two-player or more.** With three or more players a single number no longer suffices: each node
  carries a vector of utilities, one per player, and each player maximises their own component.
- **Sequential or simultaneous.** We assume players alternate.

This week is the easy corner of that space: **two-player, zero-sum, deterministic, perfect
information, turn-taking**. Chess, checkers, go, nim, tic-tac-toe.

## A game as a formal problem

The same shape as week 3's CSP: state the problem in a fixed vocabulary, and one algorithm serves
every instance.

- **`initial`** — the starting position
- **`to_move(s)`** — whose turn it is in `s`
- **`actions(s)`** — the legal moves in `s`
- **`result(s, a)`** — the position after playing `a`
- **`is_terminal(s)`** — is the game over
- **`utility(s, p)`** — the payoff to player `p` in a terminal state

By convention one player is **MAX**, who wants utility high, and the other is **MIN**, who wants it
low. In a zero-sum game that single number describes both, since whatever MAX gains MIN loses.

## The game tree and the minimax value

The tree has the initial state at the root, one branch per legal move, and terminal positions at
the leaves. The **minimax value** of a node is the utility of reaching it, assuming both players
play optimally from there on:

    minimax(s) = utility(s)                        if s is terminal
               = max over a of minimax(result(s,a))  if MAX is to move
               = min over a of minimax(result(s,a))  if MIN is to move

Read the recursion carefully, because it is the whole idea. At your own nodes you take the best
child; at the opponent's nodes you take the **worst**, because that is what they will hand you.
Optimism at your levels and pessimism at theirs, alternating all the way down.

That also says what minimax assumes: **a perfect opponent**. Against an imperfect one, minimax is
still safe — it can only do better than predicted — but it will not exploit a weak player, because
it has already assumed they will not blunder.

## Minimax

Depth-first through the whole tree, returning values up.

- **Complete?** Yes, for a finite tree.
- **Optimal?** Yes, against an optimal opponent.
- **Time?** **O(b^m)** — branching factor `b`, tree depth `m`. Every leaf is examined.
- **Space?** **O(bm)**, depth-first.

The time is the problem. Chess has `b` around 35 and games run 80 or more plies; `35^80` is not a
number you search. So for anything beyond a toy, minimax is a definition of correct play rather
than a way to play.

## Alpha-beta pruning

The observation: **you do not need a node's exact value if you already know it cannot affect the
decision.** Alpha-beta computes exactly the minimax value while skipping branches that provably
cannot change it.

Two bounds travel down the tree:

- **α** — the best value MAX can already guarantee, anywhere along the current path. It only rises.
- **β** — the best value MIN can already guarantee. It only falls.

At a MAX node, once its value reaches `β`, stop: MIN already has a way to hold the score to `β`
somewhere above, so MIN will never let play reach here. At a MIN node, once its value falls to `α`,
stop, for the mirror reason.

    if at a MAX node and v >= beta:  return v     # MIN would avoid this node
    if at a MIN node and v <= alpha: return v     # MAX would avoid this node

Two things worth being clear about:

- **It changes nothing about the answer.** Alpha-beta returns the same value minimax does. It is
  not an approximation, it is the same search with the provably irrelevant parts skipped.
- **It prunes subtrees, not nodes.** When a cutoff fires, everything below the remaining children
  goes unexamined, which is why the saving is so large.

## Move ordering

How much alpha-beta saves depends entirely on the order the moves are tried in, because a cutoff
can only fire once a good enough value has been seen.

| ordering                     | nodes examined                  |
| ---------------------------- | ------------------------------- |
| worst case (best moves last) | O(b^m) — no better than minimax |
| random                       | about O(b^(3m/4))               |
| perfect (best moves first)   | **O(b^(m/2))**                  |

Perfect ordering halves the exponent, which makes the effective branching factor `√b` rather than
`b` — for chess about **6 instead of 35**. Put another way, the same time budget searches roughly
**twice as deep**.

Of course you cannot order perfectly: a function that always knew the best move first would
already be a perfect player. But you can get close, and it is cheap:

- **Cheap static rules** — in chess, try captures, then threats, then forward moves. This alone
  gets within about a factor of two of the perfect-ordering bound.
- **Killer moves** — a move that caused a cutoff elsewhere at the same depth is likely to cause one
  again. Try it first.
- **Iterative deepening** — search one ply, record how the moves ranked, then search one ply deeper
  trying them in that order. The re-search costs less than the better ordering saves.

## Transposition tables

Different move orders often reach the same position — a **transposition**. Without memory, the
search re-explores each one. A **transposition table** caches the value of positions already
evaluated, keyed by the position rather than by the path taken to it.

This is the same idea as the visited set in week 1, with the same caveat: it costs memory
proportional to the number of distinct positions stored, and it interacts with depth, since a
value computed at one search depth is not directly reusable at another.

## When the tree is too big

For real games you cannot reach the leaves, so you stop early and guess. Two changes to minimax:

- replace `is_terminal` with a **cutoff test** — usually a depth limit
- replace `utility` with an **evaluation function** that estimates the value of a non-terminal
  position

An evaluation function must agree with the real utility at terminal states, be quick to compute,
and — most importantly — **correlate with the actual chance of winning**. The classic form is a
weighted sum of features: in chess, material with pawn 1, knight and bishop 3, rook 5, queen 9,
plus terms for structure and mobility.

Two well-known problems come with stopping early:

- **The horizon effect.** The search cannot see a disaster that lies one ply past the cutoff, so it
  will happily play moves that postpone a loss beyond the horizon rather than avoid it.
- **Quiescence.** Cutting off in the middle of an exchange gives a nonsense reading. The fix is a
  **quiescence search**: at the cutoff, keep searching but only through "noisy" moves such as
  captures, until the position is quiet.

**Forward pruning** goes further and discards moves that merely look bad, without proof. Beam
search over the top `k` moves is the simple version. Unlike alpha-beta this _can_ lose the best
move — it trades correctness for depth, deliberately.

## Compared with the search we already have

|                 | the world           | at each level you              |
| --------------- | ------------------- | ------------------------------ |
| Week 1–2 search | still               | take the best branch           |
| Week 3 CSP      | still               | assign any variable, propagate |
| Week 4 GA       | still               | recombine a population         |
| **Games**       | **has an opponent** | **alternate best and worst**   |

And within this week:

|                      | returns                 | examines                                  |
| -------------------- | ----------------------- | ----------------------------------------- |
| Minimax              | the exact minimax value | every leaf, O(b^m)                        |
| Alpha-beta           | **the same value**      | as few as O(b^(m/2)), ordering permitting |
| Depth-limited + eval | an estimate             | whatever the cutoff allows                |

The middle row is the one to hold on to. Alpha-beta is the rare optimisation that costs nothing at
all in answer quality — every node it skips is one it has proved cannot matter.

### What that is worth, measured

Nim with piles of 3, 4 and 5, searched to the end:

|            |     nodes |   time |
| ---------- | --------: | -----: |
| minimax    | 1,038,768 |  1.1 s |
| alpha-beta |    33,916 | 0.04 s |

Same value, same recommended move, thirty times less work.

And Connect Four to a depth of six, where ordering has room to matter:

|                                   |     nodes |
| --------------------------------- | --------: |
| minimax                           |   137,257 |
| alpha-beta, moves in column order |    17,364 |
| alpha-beta, moves in random order |     8,049 |
| alpha-beta, centre columns first  | **2,127** |

Two things in that table are worth sitting with. Sorting the moves is seven lines, and it saved
more than the pruning did. And a _random_ order beat the natural left-to-right one — the default
ordering is not neutral, it is simply an arbitrary one that happens to be bad here.

### But depth is not the same as strength

Once you cut the search off, the guarantees stop. Playing depth-`d` Connect Four players against a
depth-2 opponent, twenty games each from random openings:

| depth |    won | lost | drawn |
| ----: | -----: | ---: | ----: |
|     3 |      6 |   11 |     3 |
|     4 |     16 |    3 |     1 |
|     5 | **19** |    0 |     1 |
|     6 |     11 |    6 |     3 |

Depth 3 loses to depth 2, and depth 6 is worse than depth 5. Every one of those players runs the
same algorithm with the same evaluation function, and the search is exact above its cutoff. What
changes is where the cutoff falls: stopping just after your own move shows a position you have just
improved and the opponent has not yet answered, which flatters it, and stopping one ply later does
the opposite. The estimate therefore swings with the **parity** of the depth, and a crude
evaluation function cannot absorb that. Real engines either fix the parity or extend unstable
positions instead of stopping on them.

So the two halves of this week make opposite promises. Alpha-beta is free. The cutoff is not.

## The homework: tic-tac-toe

The whole game tree is about half a million nodes, so this is the one game on the list where none of
the second half of this week applies: no evaluation function, no depth limit, no quiescence. The
agent plays perfectly, and the only question is whether your search can say so.

**Two modes, one program.** The first line of input picks. JUDGE reads `TURN X` and a board, prints
one move and exits — that is the only mode the automatic tests use. GAME reads `FIRST` and `HUMAN`
and plays a full game at the terminal. Both are required; only JUDGE is graded.

**Reading the framed board.** Nothing in `+---+---+---+` or the bars carries information, so do not
parse the frame: scan the text for `X`, `O` and `_` and take them in reading order. Three lines of
code, and it is immune to stray characters in the assignment text. In GAME mode read exactly
`2n + 1` lines and no more, or the same read will swallow the moves typed after the board.

**The state is the vocabulary from the top of this page.** `to_move` comes from `TURN`, `actions(s)`
is the empty cells, `result(s, a)` writes the symbol and flips the player, `is_terminal(s)` is a line
of three or a full board. One thing is easy to miss: a board handed to you mid-game may already be
won, so compute the winner from the board you read rather than assuming the game is still live. That
is also what makes the terminal case work — JUDGE expects `-1` there, not a move.

**Who is MAX.** `TURN` names the side to move and it is not always X. Either make that side the
maximiser, so the root is always a max node and a positive value means the side on move is winning,
or keep X as MAX for ever and negate when O is on move. Both work; mixing them does not.

**Depth belongs in the utility, and this is the graded part.** Scoring a win `+1` and a loss `-1` is
not enough, because a move that wins now and a move that wins three plies later both score `+1` and
the search keeps whichever it happened to see first. Score a terminal by when it arrives instead:

    utility = 1000 - moves    if the maximiser won
            = moves - 1000    if the maximiser lost
            = 0               draw

The subtraction does both halves of the job. A win is worth more the sooner it comes, so the agent
finishes instead of dawdling and handing a human another turn to escape. A loss is worth more the
later it comes, so a lost position plays on and makes the opponent prove it rather than resigning
into the fastest defeat — which is exactly where a human blunders.

Measured on the official tests: the depth-aware utility passes all 14, and the `±1` version passes 9
and fails 5 with `suboptimal: violates win-now/lose-late tie-break`. At 3×3 nothing else about the
search can earn or cost you points. This line can.

**What the checker accepts.** It runs its own depth-aware search and accepts any move from the
optimal set, so the sample output is not a target: on the empty board all nine moves draw, all nine
are accepted, and `1 1` is as correct as the `2 2` printed in the assignment. Where only one move is
optimal — the win, the block, the one reply that saves a draw — it demands that move. The accepted
set is not closed under symmetry, though. In

    X _ _
    _ O _
    _ _ X

with O to move, `1 3` is accepted and its mirror image `3 1` is not, although reflecting the board
maps one onto the other. Enumerate the cells in reading order and replace your best move only on a
strictly better value — which is what the checker itself does — and you stay inside its set.

**Output and timing.** `row col`, 1-based, or `-1` for a terminal position. You may print
`# TIMES_MS: alg=<milliseconds>` as the first line; the checker strips it and `--bench` reports it.
The limit is one second per test, which is three orders of magnitude more than this problem needs.
Run the tests with `judge run --bench -p tic-tac-toe <your executable>`; it runs an already-built
binary rather than compiling, and `-p` is worth passing because the automatic detection goes by file
name.

**What alpha-beta is worth here.** The empty board, X to move, searched to the end:

|                                              |                nodes |
| -------------------------------------------- | -------------------: |
| minimax                                      |              549,946 |
| alpha-beta, cells in reading order           |               20,866 |
| alpha-beta, cells in random order            | 22,649 (mean of 200) |
| alpha-beta, centre, then corners, then edges |            **7,865** |

Twenty-six times from the pruning, and another 2.7 from ordering nine cells by hand. Note that the
random order comes out slightly worse than reading order, the opposite of the Connect Four table
above: reading order here starts with a corner, which is already a decent opening. None of this is
needed to pass — half a million nodes is milliseconds — but minimax and alpha-beta must agree on the
value of every position, and that is the free check that your two cutoff conditions are the right
way round.

**GAME mode is where you find the bugs.** Print the board after every move, reject an illegal move
and ask again instead of crashing, and end with `WINNER: X`, `WINNER: O` or `DRAW`. Perfect play from
either side is a draw, so against a correct agent you cannot win. If you do win, the position where
your fork appeared is a test case: write it out as a JUDGE input and see what the search says about
it.
