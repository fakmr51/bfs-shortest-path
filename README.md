# bfs shortest path: why the queue guarantees it, the template to memorize, and when the answer is actually Dijkstra

Type "bfs shortest path" into a search bar and you're usually in one of three situations: you're writing your first BFS and want to know *why* a plain queue magically returns the shortest route, you're cramming for an interview and want the template that survives a whiteboard, or you got a wrong answer on a grid problem and can't figure out which line is lying to you.

All three get solved here. The queue trick first, then the code, then the cases where BFS quietly gives you a longer path than you think.

## The one property doing all the work

BFS explores a graph in layers. Everything at distance 1 from the source, then everything at distance 2, then distance 3. Nothing at distance 3 gets looked at while a distance-2 node is still in the queue.

That ordering is the entire proof. In an **unweighted** graph — or one where every edge costs the same — the length of a path is just how many edges it has. If BFS reaches node `v` after expanding every node closer to the source, then the first time `v` comes out of the queue, it arrived by a path with the fewest possible edges. A shorter route would have put `v` in an earlier layer.

Two consequences worth internalizing:

- The BFS tree (the parent pointers built during traversal) contains shortest paths from the source to every reachable vertex in its component. A DFS tree does not — DFS trees are long and skinny, BFS trees are short and fat.
- The complexity is `O(|V| + |E|)` with adjacency lists, since each vertex is enqueued once and each edge is examined once.

This is not an abstract textbook fact. Network routing counts hops exactly this way, and "degrees of separation" in a social graph is a BFS distance.

And here's the caveat that costs people real points: **BFS finds shortest paths only when edge weights are equal.** The moment your edges are weighted, "fewest edges" and "cheapest path" stop being the same thing. More on that below.

## The template, with the three lines that actually matter

python
from collections import deque

def bfs_shortest(graph, src, target):
    dist = {src: 0}
    q = deque([src])

    while q:
        u = q.popleft()
        if u == target:
            return dist[u]
        for v in graph[u]:
            if v not in dist:        # mark visited when enqueueing
                dist[v] = dist[u] + 1
                q.append(v)
    return -1


Everything that goes wrong in a BFS implementation is hiding in this snippet:

1. **`deque`, not a list.** `list.pop(0)` shifts every remaining element and turns your linear algorithm into a quadratic one. On a small grid you won't notice. On a 25,000-node graph in a timed interview you will.
2. **Mark visited when you enqueue, not when you dequeue.** If you wait until the pop, the same node can be pushed by several neighbors before anyone processes it. The queue grows, and in grid problems the recorded distance can also come out wrong.
3. **`dist[v] = dist[u] + 1`.** The distance is a property of the edge you just crossed, not of the moment you happened to visit. Level-by-level loops (`for _ in range(len(q))`) with an external counter work too, but you have to be careful that the counter lives outside the inner loop.

## Getting the path back, not just the number

Most interview questions ask for the length. A meaningful minority want the route. Keep a parent map alongside the distance map and walk backwards from the target:

python
prev = {src: None}
...
prev[v] = u
# afterwards
path, node = [], target
while node is not None:
    path.append(node)
    node = prev[node]
path.reverse()


Princeton's classic `PathFinder` implementation does exactly this with two tables — `prev[v]` for the previous vertex on the shortest path and `dist[v]` for its length — so that after one BFS pass you can answer path queries from the source to any vertex in the component in time proportional to the path length.

## Where BFS stops being the right tool

This is the part that separates people who memorized a template from people who can defend a choice in an interview.

| Situation | Right approach | Why BFS fails |
| --- | --- | --- |
| All edges weight 1 | BFS with a queue | — |
| Weights 0 and 1 only | 0-1 BFS with a `deque` | Push 0-weight neighbors to the front, 1-weight to the back |
| Non-negative weights | Dijkstra with a priority queue | BFS commits to the first arrival; with weights, the first arrival isn't the cheapest |
| Negative weights | Bellman-Ford | Dijkstra's correctness assumption breaks |
| All paths, not just the shortest | BFS for distances + DFS backtracking over parents | Enumerating paths is exponential |

The 0-1 BFS case is real and shows up in current problem sets. In LeetCode 3552 (Grid Teleportation Traversal), teleporting between matching cells costs 0 while a normal move costs 1 — the solution is a deque where zero-cost edges are pushed to the front and cost-1 edges to the back, which reproduces Dijkstra's ordering without a heap.

One useful piece of triage: AlgoMonster's own problem-statistics breakdown classifies named academic algorithms — Kruskal and Prim, Ford-Fulkerson, Bellman-Ford — as rarely or never asked in real interviews. Dijkstra is worth a high-level understanding mostly because of the priority queue it uses, since heaps get asked constantly. So if you're deciding where to spend a week, the plain unweighted BFS pattern outranks the weighted variants by a wide margin.

## Grid BFS: same algorithm, four new ways to be wrong

Most "shortest path" problems you'll actually see are grids, not adjacency lists. The grid is just an implicit graph where each cell is a node. The pattern is identical; the traps are not.

**Direction count.** LeetCode 1091 (Shortest Path in Binary Matrix) allows 8-directional movement — edges *and* corners. A 4-directional solution is simply wrong there, not merely suboptimal. Check the problem statement before you write the direction array.

**Multi-source BFS.** LeetCode 542 (01 Matrix) asks for each cell's distance to the nearest zero. The efficient version seeds the queue with *every* zero at once and expands outward, so each cell is assigned exactly once. Running a fresh BFS from each `1` repeats enormous amounts of work. The same pattern drives LeetCode 994 (Rotting Oranges), where all rotten oranges start in the queue together.

**State, not just position.** LeetCode 1293 (Shortest Path in a Grid with Obstacles Elimination) lets you remove up to `k` obstacles. The node is now `(row, col, k)` and the runtime becomes `O(m × n × k)`. A cell can legitimately be visited more than once — once per distinct obstacle budget — so a plain 2D visited array will produce wrong answers.

**Destructive marking.** The standard 1091 solution sets `grid[r][c] = 1` to mark cells visited, which mutates the caller's input data permanently. It's the kind of thing that passes tests and then breaks somebody else's code downstream.

**Count cells, not moves.** 1091 returns the number of cells in the path, including the start and the end. A path that crosses three cells has length 3, not 2. This off-by-one is the single most common wrong-answer submission on that problem.

**Bidirectional BFS exists.** Running one search from the source and one from the target and expanding whichever frontier is smaller cuts the explored area substantially — instead of one circle of radius `d`, you get two of radius `d/2`. Worth knowing when a problem says the graph is huge and the endpoints are fixed.

## The mistakes that actually cost the round

Compressed, in the order they tend to appear:

1. Marking visited on dequeue instead of enqueue.
2. Using four directions when the problem allows eight (or vice versa).
3. Forgetting to check that the start and end cells are passable before starting the search.
4. Returning the edge count when the problem wants the cell count.
5. Assuming BFS handles weighted edges.
6. Reusing a `visited` array across test cases, or mutating the input grid and leaving it mutated.

If you internalize nothing else from this page, items 1 and 4 cover the majority of failed submissions.

## How much BFS is actually worth in an interview

Quite a lot, according to the platforms that publish their own data. AlgoMonster's statistics page names DFS, BFS, and two pointers as the highest-ROI topics in coding interviews, with Amazon and Meta both drawing roughly half their problem sets from two pointers, DFS, and BFS. Google is different — the company deliberately avoids reusing publicly known problems and combines patterns (prefix sum + binary search, DFS + prefix sum) — which is an argument for fluency over memorization.

So if you're going to learn one graph pattern properly, this is the one with the best return.

👉 [Learn the BFS pattern properly at AlgoMonster](https://algo.monster?fpr=li69)

## What AlgoMonster does with this specific pattern

AlgoMonster is built by ex-Google engineers around 48 coding patterns rather than a giant problem dump. For BFS specifically, the parts that matter:

- **A decision flowchart that asks "shortest-path problem?" before it asks anything else.** The flowchart routes you to BFS based on the problem's wording and constraints, which is the skill that actually breaks under interview pressure — not writing the queue loop.
- **Solution pages with the complexity worked out and the pitfalls listed.** The writeup for LeetCode 1091 walks through the `O(n²)` time and space analysis and flags the destructive in-place marking issue directly, rather than presenting the trick as free.
- **~700 interactive diagrams across 500+ lessons and 3,000+ editorials.** Grid BFS is a visual process; watching the frontier expand one layer at a time lands faster than reading pseudocode.
- **Speedrun mode** for timed reps, and an AI assistant you can query while working a problem.

Being straight about the limits: AlgoMonster is entirely text and diagrams, with no video walkthroughs at all. Its problem count — 231 lessons and problems across the 48 patterns — is small next to LeetCode's 3,000+, and third-party reviews consistently describe its system design coverage as introductory. There's no coaching or human feedback loop. The realistic setup for most candidates is AlgoMonster for the pattern framework plus LeetCode for pure volume, and reviewers on both sides say that out loud.

## What it costs

AlgoMonster doesn't sell a monthly plan. Your choices are the free tier, a year, or a one-time lifetime purchase:

| Plan | What you get | Price | Billing | Purchase |
| --- | --- | --- | --- | --- |
| Free | Limited access to a subset of patterns and practice problems | $0 | — | [Start with the free tier](https://algo.monster?fpr=li69) |
| Pro — Yearly | All 48 patterns, lessons and problems; unlimited AI assistant; unlimited Speedrun; 500+ lessons across 6 courses; 1,000+ illustrations; 3,000+ editorials; real company questions updated monthly; member discount on live bootcamps; includes the Foundation and Build an LLM courses | $99/year (about $8.25/month billed annually) | 12 months of full access | [Get Pro Yearly access](https://algo.monster?fpr=li69) |
| Pro — Lifetime | Everything in the annual plan, plus lifetime updates | $189 | One-time payment | [Buy Pro Lifetime once](https://algo.monster?fpr=li69) |

Two honest notes on those numbers. First, AlgoMonster runs sales constantly, and the subscribe page price has moved — snapshots have shown the annual plan at $99/year and at $69.99/year, so the figure you land on depends on which promotion is live when you look. Second, one promo-code tracker lists the pre-sale prices as $198/year and $378 lifetime, which is consistent with the half-off framing the platform uses. Specialized courses developed with outside partners, like Quant Interview Prep, are sold separately and aren't part of the subscription.

## So should you pay for this?

If BFS is the pattern you keep fumbling — you can write the loop but you don't reliably recognize *when* a problem is a shortest-path problem — the flowchart and template approach addresses precisely that gap, and $99 for a year is cheaper than the LeetCode Premium subscription most people are already paying for.

If you're two weeks from an interview, don't buy anything. BFS fluency comes from reps, and there isn't room for reps in two weeks. If you still have to look up how a queue works, start with the free Foundation material first.

And if you're prepping once and stopping, buy the year. The lifetime plan only pays off if you expect to interview again — which, for most engineers, you will.

Either way, go write the template from memory right now, without looking. If you can produce the `deque`, the visited-on-enqueue line, and `dist[v] = dist[u] + 1` from scratch, you already own the part of this that matters.
