# 01 Matrix: Multi-Source BFS Step by Step, the Two-Pass DP Alternative, and How to Spot the Pattern Under Time Pressure

"01 Matrix" is LeetCode 542, and it sits in an awkward spot. The problem statement is short enough to read in fifteen seconds, the optimal solution is about twenty lines, and yet it's one of the most common places where prepared candidates lose a coding screen. Not because the code is hard — because the way the problem is phrased pushes you toward the slow solution.

Here's the setup: given an `m x n` binary matrix, return a matrix of the same shape where every cell holds the distance to the nearest `0`. Distance means steps, moving up, down, left or right — no diagonals. Cells that already contain `0` get `0`. Constraints are `1 <= m, n <= 10^4`, `m * n <= 10^4`, and the matrix is guaranteed to contain at least one `0`.

text
Input:  [[0,0,0],        Output: [[0,0,0],
         [0,1,0],                 [0,1,0],
         [1,1,1]]                 [1,2,1]]


The bottom-middle `1` is two steps from the nearest zero, so it gets `2`. That's the whole problem. Everything interesting is in how you get there.

## The trap in the literal reading

Read the question again and you'll naturally think: for each `1`, walk outward until you bump into a `0`.

That approach is *correct*. BFS from a single cell finds the shortest path on an unweighted grid — that's the property that makes BFS the default tool for this family of problems. The catch is cost. Every one of those searches can crawl across nearly the entire grid before it finds a zero, and you're launching a separate search for every `1`. Worst case that's `O((m*n)^2)` — around 10^8 operations at the constraint ceiling, which is exactly the kind of number that turns into a time limit exceeded verdict.

The waste is obvious once you see it. Those independent searches retrace each other's steps constantly. If the grid has one zero in a corner and thousands of ones, a thousand searches are all walking the same paths toward the same source.

## Reframe: start from the zeros instead

Flip the direction of the question. Instead of asking every `1` where its nearest `0` is, let every `0` spread outward at the same time.

The reason this works is that all the zeros are already solved — their distance is `0`, no search required. So seed a queue with every zero cell, then expand layer by layer. The first time any cell gets reached, that distance is final, because BFS visits cells in strictly increasing distance order. There is no later wave that could arrive with a shorter path.

If you prefer the formal framing: imagine a virtual "super source" wired to every zero cell with an edge weight of `0`. A standard BFS from that super source gives you the answer for the whole grid. Multi-source BFS is just that idea with the virtual node removed and the sources dropped into the queue directly.

Two implementation details carry most of the weight:

- **Mark cells when you enqueue them, not when you pop them.** Assigning the distance at enqueue time is what guarantees each cell enters the queue exactly once. If you mark on pop, the same cell can be pushed multiple times.
- **Assign each cell's distance exactly once.** In the usual version, the answer matrix itself doubles as the visited set, with `-1` meaning "not reached yet".

## A dry run on the second example

Tracing `[[0,0,0],[0,1,0],[1,1,1]]` makes the layering concrete. Every zero goes into the queue at the start:

| BFS layer | Cells confirmed this layer | Distance matrix |
| --- | --- | --- |
| initial seed | (0,0), (0,1), (0,2), (1,0), (1,2) | `[[0,0,0],[0,-1,0],[-1,-1,-1]]` |
| distance 1 | (1,1), (2,0), (2,2) | `[[0,0,0],[0,1,0],[1,-1,1]]` |
| distance 2 | (2,1) | `[[0,0,0],[0,1,0],[1,2,1]]` |

The queue empties after layer 2 and the answer is `[[0,0,0],[0,1,0],[1,2,1]]`. Notice that `(1,1)`, `(2,0)` and `(2,2)` all get distance `1` in the same layer — they're all one step from some zero, and it doesn't matter which one.

## The code

Python, multi-source BFS:

python
from collections import deque
from typing import List

class Solution:
    def updateMatrix(self, mat: List[List[int]]) -> List[List[int]]:
        rows, cols = len(mat), len(mat[0])
        dist = [[-1] * cols for _ in range(rows)]
        q = deque()

        # Every 0 is already solved: its distance is 0.
        for r in range(rows):
            for c in range(cols):
                if mat[r][c] == 0:
                    dist[r][c] = 0
                    q.append((r, c))

        dirs = ((1, 0), (-1, 0), (0, 1), (0, -1))

        while q:
            r, c = q.popleft()
            for dr, dc in dirs:
                nr, nc = r + dr, c + dc
                if 0 <= nr < rows and 0 <= nc < cols and dist[nr][nc] == -1:
                    dist[nr][nc] = dist[r][c] + 1   # assigned before enqueue
                    q.append((nr, nc))

        return dist


You'll also see two popular variants in write-ups. One processes the queue level by level with an inner `for _ in range(len(q))` loop and increments a distance counter, which visually mirrors the wave expansion. The other stores `(row, col, distance)` tuples in the queue so each entry carries its own distance. Both are fine. Neither is necessary here, since `dist[r][c] + 1` already tells you the layer. The level-based version is slightly more awkward because the counter has to be advanced in the right place, and that's a common off-by-one spot in live interviews.

## Complexity, and the space trick

Time is `O(m * n)`. Each cell is enqueued at most once and processed once, with four constant-time neighbor checks per cell. No search revisits a cell.

Space is `O(m * n)` for the queue in the worst case (an all-zero grid seeds every cell at once) plus the result matrix. If you want to avoid the second allocation, you can overwrite the input: turn the `1`s into `-1` as you scan for zeros, then write distances back into `mat` itself. That's what AlgoMonster's C++ and Java versions of 542 do — the input matrix carries the answer and no separate `dist` array is needed. It's a legitimate micro-optimization, though in an interview you should say out loud that you're mutating the caller's input and ask whether that's acceptable.

## The second valid answer: two passes of DP

There's a completely different solution that never touches a queue, and interviewers sometimes fish for it.

The observation is that the shortest path from a cell to the nearest zero only ever moves in two of the four directions at each step (say, up and left, or down and right) — so you can build the answer with two sweeps. First, a forward pass from top-left to bottom-right, where each cell takes `min(itself, top + 1, left + 1)`. Then a backward pass from bottom-right to top-left, taking `min(itself, bottom + 1, right + 1)`. Initialize zeros to `0` and everything else to a large value like `10**9`, and after both sweeps the matrix is correct. Same `O(m * n)` time, but no queue and no visited array.

Which should you present? Multi-source BFS is easier to defend live because it directly models shortest-path expansion in an unweighted graph, and the "why is the first visit always optimal" argument is a clean one-liner. DP is the better answer if the interviewer changes the problem — for instance, if grid cells carry different weights, multi-source BFS stops being correct and you need Dijkstra, while the two-pass DP idea also breaks down and you're back to a priority queue. Knowing where each approach dies is worth more than knowing both cold.

## Where people actually lose points

The failures on this problem cluster in a few predictable places.

**Running BFS from every `1` in the final answer.** It passes the small examples and dies on the large ones. If you catch yourself writing the naive version, say why it's `O((m*n)^2)` and move to the multi-source version rather than quietly submitting something you know is wrong.

**Marking on dequeue instead of enqueue.** This is the single most common performance bug. It still produces correct distances, which is what makes it dangerous — it just pushes far more entries than necessary.

**Using DFS.** DFS finds *a* path to a zero, not the nearest one. Grid search for shortest distance is a BFS job.

**Direction-array typos.** Writing `dist[nc][nc]` instead of `dist[nr][nc]`, or swapping row and column in the delta pair, produces output where a few cells have inexplicably large numbers. If your dry run shows a distance that's obviously too big, check the indices before you check the algorithm.

**Forgetting that zeros can be dense.** A grid that's entirely zeros is a legal input and should come back unchanged. So is a `1x1` matrix containing a single `0`, which returns `[[0]]`. Single-row and single-column grids catch diagonal moves if you accidentally allow them.

**Ignoring the constraint that `m*n <= 10^4`.** That bound is a hint about the intended complexity. If your solution is quadratic in the number of cells, the constraints are telling you it's the wrong solution.

## The pattern behind the problem

Strip away the grid and 01 Matrix is: "for every cell in a graph, find the distance to the nearest member of a source set." That shows up constantly.

- **Walls and Gates** (LeetCode 286) is the same algorithm with a different story — flood out from every gate, write the distance into each empty room.
- **Rotting Oranges** (LeetCode 994) spreads rot from all initially rotten oranges simultaneously and asks how many minutes until the wave finishes.
- **CSES "Monsters"** runs a multi-source BFS from the monster positions to decide where and when a player can escape.

Once you recognize the "many sources, nearest one wins" structure, all three collapse into the same template. That's the actual skill being tested, and it's why practicing by topic beats practicing by problem count — you want the recognition step to fire before you start typing.

AlgoMonster is built around exactly that bet. It's a course, not a problem bank: 48 named patterns, each with a concept explanation, a reusable code template and mapped practice problems, plus a decision flowchart for figuring out which pattern a new problem needs. Its own write-up of 542 leads with the multi-source insight and then devotes a section to the single-source mistake, which is the failure mode that matters. If you want to see how the platform explains a problem you already know before paying for anything, 👉 [read AlgoMonster's 01 Matrix walkthrough](https://algo.monster/liteproblems/542?fpr=li69) — it's the free side of the site.

Format is text and diagrams, not video. The subscribe page advertises 500+ lessons across 6 courses, 1,000+ illustrations and 3,000+ editorials; reviewers who've gone through the core track describe roughly 230 lessons and problems mapped to the 48 patterns. Either way, the footprint is far smaller than LeetCode's 3,000+ problems. That's the trade: fewer problems, each one tied to a pattern you're supposed to reuse.

Tools worth knowing about before you buy: an inline editor so you can code the problem on the same page as the explanation, an AI assistant you can select text and ask questions of, Speedrun mode for timed review passes in the week before an interview, and an algorithm visualizer that animates the execution step by step.

## Plans and current pricing

AlgoMonster sells two paid plans and a free tier. There's no monthly option and no team or enterprise tier — it's built for individual, self-directed prep.

| Plan | What you get | Price | Billing | Link |
| --- | --- | --- | --- | --- |
| Free | Limited access to a subset of patterns and problems — enough to judge the text-and-diagram format before paying | $0 | Nothing to cancel | [Start with the free tier](https://algo.monster?fpr=li69) |
| Pro – Yearly | All patterns, lessons and problems; unlimited AI assistant; unlimited Speedrun; company question sets updated monthly; 500+ lessons across 6 courses; member discount on live bootcamps | $99/year (roughly $8.25/month) | Billed annually, cancel or upgrade anytime | [Get Pro Yearly](https://algo.monster/subscribe?fpr=li69) |
| Pro – Lifetime | Everything in the yearly plan plus lifetime updates to all current and future content | $189 | One-time payment, no renewals | [Get Pro Lifetime](https://algo.monster/subscribe?fpr=li69) |

Two notes on those numbers. First, AlgoMonster runs frequent promotions and the checkout price genuinely moves — published figures from the last year range from about $69 to $119 for a year and about $89 to $189 for lifetime, so treat the table as the current list state and confirm the live number before you commit. Second, coupon sites advertise stackable partner codes, but those claims aren't independently verifiable and the codes expire without notice; if a code exists, the checkout page is where you'll find out, not on a coupon blog.

Which one makes sense depends on your timeline. A single interview loop in the next two months is a yearly-plan problem. If you expect to interview again in a couple of years — a common pattern for engineers chasing promotions — the lifetime plan starts paying for itself around the second year, and it locks in access to whatever gets added later.

Where the platform is thinner: no video walkthroughs at all, system design content is introductory rather than L5-level depth, and there's no live mock interview or timer pressure built into the problems. If those are your gaps, AlgoMonster covers the pattern-recognition half of the problem and you'll need something else for the rest.

## Questions that come up

**What's the time complexity of 01 Matrix?** `O(m * n)` for the optimal solution, with `O(m * n)` extra space. The brute-force version that runs BFS from each `1` is `O((m*n)^2)`.

**Can you solve it with dynamic programming instead of BFS?** Yes. Two sweeps — forward using top and left neighbors, backward using bottom and right — each taking the minimum of the current value and a neighbor plus one. Same asymptotic time, no queue.

**Why does BFS from all zeros at once give the correct answer?** Because BFS expands in order of increasing distance. The first time a cell is reached, no shorter path could have existed, or that path would have been discovered in an earlier layer.

**What if the grid has no zeros?** The constraints rule it out — at least one `0` is guaranteed, so there's no unreachable case to handle.

If you can explain the reverse-thinking move out loud in two sentences, write the twenty lines without the marking bug, and name the two related problems, you've got 01 Matrix covered. The interviewers asking it aren't checking whether you've seen it before. They're checking whether you notice the naive reading is a trap.
