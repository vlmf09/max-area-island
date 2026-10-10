# max area of island: DFS, BFS, and Union-Find solutions that actually pass, plus the recursion trap that fails hidden tests

You found the problem, you see the grid, you know the answer is "find the biggest blob of 1s." Then you submit and get a wrong answer on a case you can't see, or a `RecursionError` at 50×50. This is one of those problems where the idea is obvious and the execution is where people leak points.

Here's the whole thing: what's really being asked, three ways to solve it, the specific mistakes that cost people the interview, and what the follow-up questions look like.

## What the problem is actually asking

You're given an `m x n` binary matrix where `1` is land and `0` is water. An island is a group of `1`s connected 4-directionally — up, down, left, right, nothing diagonal. The area of an island is simply how many cells it contains. Return the largest such area, or `0` if there's no land at all.

The classic LeetCode example is the 8×13 grid that returns `6`. The reason it trips people up is that the grid contains 11 total `1`s, and the problem statement has to explicitly say "the answer is not 11" because the two clusters touching at a corner are *not* one island. Every article about this problem repeats that line for a reason.

The constraints matter more than they look:

- `1 <= m, n <= 50`
- Every cell is `0` or `1`

Fifty by fifty is 2,500 cells. Remember that number, because it decides which implementation you should write.

This is LeetCode 695, tagged Medium, and the official topic list is Array, Depth-First Search, Breadth-First Search, Union Find, Matrix. That tag list is basically a table of contents for this article.

## The actual pattern: connected components, not search

People frame this as a search problem. It isn't. It's a labeling problem.

Every `1` in the grid belongs to exactly one island, and once you've counted a cell you must never count it again. That gives you two rules that cover the entire solution:

1. When you hit an unvisited `1`, explore its whole connected component and count the cells as you go.
2. Track the biggest count you've seen across all starting points.

That's it. The DFS/BFS decision is a detail — both produce identical answers at the same asymptotic cost. The thing that's actually being tested is whether you handle the "never count twice" part correctly, because that's where the bugs live.

This pattern repeats constantly. Number of Islands is the same traversal with a counter instead of a running max. Count Sub Islands adds a comparison between two grids. Surrounded Regions flips the logic and works from the border inward. If you can write max area of island cleanly, you can write all of them.

👉 [See how AlgoMonster maps grid traversal into a reusable template](https://algo.monster?fpr=li69)

## Solution 1: recursive DFS with in-place marking

The neat trick here is that you don't need a separate `visited` matrix. Sink the land as you walk it:

python
class Solution:
    def maxAreaOfIsland(self, grid: list[list[int]]) -> int:
        m, n = len(grid), len(grid[0])

        def dfs(r: int, c: int) -> int:
            if r < 0 or r >= m or c < 0 or c >= n or grid[r][c] == 0:
                return 0
            grid[r][c] = 0  # mark visited
            return (1
                    + dfs(r + 1, c)
                    + dfs(r - 1, c)
                    + dfs(r, c + 1)
                    + dfs(r, c - 1))

        best = 0
        for r in range(m):
            for c in range(n):
                if grid[r][c] == 1:
                    best = max(best, dfs(r, c))
        return best


Time is `O(m·n)` — every cell is entered at most once, because the moment it's entered it becomes water. Extra space beyond the call stack is `O(1)`, which is the part interviewers like hearing.

## The recursion trap nobody mentions until it fails

The recursive version above is what most people write, and for small grids it's fine. But `m` and `n` go up to 50, which means a single snake-shaped island can be 2,500 cells long. A DFS down that path is 2,500 frames deep. Python's default recursion limit is 1,000.

Whether that bites you depends on the test set — plenty of accepted Python solutions use recursion without touching `sys.setrecursionlimit`. But if you're writing this live in front of an interviewer, saying "in Python I'd either raise the recursion limit or go iterative, because a 50×50 snake island exceeds the default stack" is a much better answer than silently hoping.

Two ways out. Either `sys.setrecursionlimit(10**6)` at the top, or switch to an explicit stack.

## Solution 2: iterative DFS

Same logic, no call stack:

python
class Solution:
    def maxAreaOfIsland(self, grid: list[list[int]]) -> int:
        m, n = len(grid), len(grid[0])
        best = 0

        for r in range(m):
            for c in range(n):
                if grid[r][c] != 1:
                    continue
                grid[r][c] = 0
                stack = [(r, c)]
                area = 0
                while stack:
                    cr, cc = stack.pop()
                    area += 1
                    for nr, nc in ((cr + 1, cc), (cr - 1, cc),
                                   (cr, cc + 1), (cr, cc - 1)):
                        if 0 <= nr < m and 0 <= nc < n and grid[nr][nc] == 1:
                            grid[nr][nc] = 0  # mark on push, not on pop
                            stack.append((nr, nc))
                best = max(best, area)

        return best


The comment on the marking line is the whole point. Mark cells when you push them onto the stack, not when you pop them. If you mark on pop, the same cell can be pushed multiple times by different neighbors and you'll overcount. Same bug exists in BFS.

## Solution 3: BFS

BFS differs from iterative DFS only in the container — `collections.deque` with `popleft()` instead of a list with `pop()`. It visits cells in expanding rings from the starting point instead of diving deep. The answer is identical and the complexity is identical; the shape of the traversal is what changes.

The practical reason to reach for BFS: no recursion depth to worry about at all, and when a problem adds a distance or step dimension (adding walls, counting minimum moves), BFS is the one that carries that information naturally. For pure area counting, it's a matter of taste.

## Solution 4: Union-Find

Union-Find treats every land cell as a node. Scan the grid; for each `1`, union it with its already-visited land neighbors (up and left is enough if you're scanning in order). Keep a size array so the root of each set knows how many cells are under it. The largest size array entry at any point is your answer.

Complexity is `O(m·n·α(n))` time with path compression and union by rank, where `α` is the inverse Ackermann function — effectively constant. Space is `O(m·n)` for the parent and size arrays.

Is it worth writing in an interview? Usually not. It's more code, more extra memory, and it doesn't beat flood fill for a static grid. The right time to bring it up is when the interviewer asks a follow-up about merging connectivity dynamically, or when you recognize you're in a "connected components" frame and want to name it out loud. Naming it earns you credit even if you implement DFS.

## Complexity side by side

| Approach | Time | Space | When it's the right call |
| --- | --- | --- | --- |
| Recursive DFS | `O(m·n)` | `O(m·n)` stack worst case | Shortest code; fine for the stated constraints |
| Iterative DFS | `O(m·n)` | `O(m·n)` stack | When recursion depth is a real risk |
| BFS | `O(m·n)` | `O(m·n)` queue | When depth is a concern or the problem adds distance |
| Union-Find | `O(m·n·α(n))` | `O(m·n)` arrays | When you're modeling dynamic connectivity |

All four are linear-ish in the number of cells. There's no clever asymptotic win hiding here — the grid is the input, and you have to look at every cell at least once.

## Mistakes that actually cost points

**Counting diagonals.** The most common conceptual error, and the reason the sample explanation is worded so bluntly. Four neighbors. Not eight.

**Forgetting to mark visited, or marking in the wrong place.** Marking after the recursive calls return, or on pop instead of on push, means a cell gets revisited. In the recursive case that's infinite recursion and a stack overflow; in the iterative case it's a wrong answer.

**Getting the bounds check order wrong.** `grid[r][c] == 0` must be evaluated *after* the range check, and the whole thing short-circuits with `or`. Write it the other way and you index out of range.

**Returning something other than 0 for an all-water grid.** Trivial, and still a real submission failure.

**Mutating the input without saying so.** The in-place trick destroys the caller's grid. LeetCode doesn't care. A human interviewer might, and some of them will ask. The clean answer: "I'm mutating the grid to save `O(m·n)` space. If the input needs to survive, I'll copy it or use a visited set, and I'll trade that for the memory." Saying it out loud converts a potential ding into a point in your favor.

## Follow-up questions interviewers actually ask

Once you solve it, the conversation usually moves sideways:

- *What if I want the number of islands instead of the largest?* Increment a counter at each new starting cell instead of tracking a max.
- *What if diagonal connections count?* Swap the four-direction offsets for eight. Two lines of change.
- *What about the perimeter of the largest island?* Count edges adjacent to water or the boundary while traversing.
- *What if the grid doesn't fit in memory?* This is the interesting one. Flood fill assumes random access to the whole grid, so you need a different conversation: streaming by row, union-find with only the previous row retained, or external processing. There's no single right answer, and interviewers ask it to watch how you reason.
- *What's the time complexity if I add a constraint?* Same answer as before, but they want to hear the reasoning about which cells can still be touched.

The pattern-recognition skill in play across all of these is the same one: look at a grid of values, ask whether you're counting connected components, then reach for the traversal template. That's the transferable piece.

## If you want structured practice instead of one-off solutions

AlgoMonster is the platform behind the link in this article. It's built by ex-Google engineers and competitive programmers, and its organizing idea is that interviews draw from roughly 48 reusable patterns rather than thousands of unique problems. Grid DFS/BFS — the pattern governing max area of island — is one of them.

What's in it, per the site's own description: interactive text lessons rather than video, a decision flowchart that walks you from a problem description to the likely pattern, an inline IDE for running code in the browser, code templates in multiple languages (Python, Java, JavaScript, C++, Go and others appear in its template list), Speedrun mode for timed review, and an AI assistant that's embedded in the lessons rather than bolted on. The subscribe page currently advertises 500+ lessons across 6 courses, 1,000+ illustrations, and 3,000+ editorials.

Its pitch is narrow and specific: if grid traversal is one of a dozen patterns you keep fumbling, mapping it to a template is worth more than solving another 20 random grid problems. If you already write flood fill in your sleep, paying for a course won't add much.

### Plans and current pricing

| Plan | What you get | Price | Billing | Link |
| --- | --- | --- | --- | --- |
| Free | Limited access to sample lessons and problems | $0 | — | [Start free and look around](https://algo.monster?fpr=li69) |
| Pro — Monthly | All patterns, lessons and problems unlocked; unlimited AI assistant; unlimited Speedrun; company question sets | $18.99/mo | Billed monthly | [Check the monthly plan](https://algo.monster/subscribe?fpr=li69) |
| Pro — Yearly | Everything in monthly, plus 12 months of full access and member discount on live bootcamps | $5.84/mo, billed annually ($69.99/year) | Billed yearly | [See the yearly plan](https://algo.monster/subscribe?fpr=li69) |
| Pro — Lifetime | Everything in the annual plan, billed once, with lifetime updates | $89 | One-time | [Look at lifetime access](https://algo.monster/subscribe?fpr=li69) |

A few things worth knowing before you pay:

AlgoMonster runs frequent promotional pricing, and the numbers move. Coupon and review sites circulating numbers like $99/year or $189 lifetime are quoting older sale tiers — the price on the actual checkout page is the one that matters, so check it before entering a card. There's no free trial beyond the free tier, so the free tier is your preview: use it to judge whether you actually like learning from text and diagrams.

The honest limitations, which several independent reviews flag consistently: there are no video walkthroughs at all, the problem count sits in the low hundreds rather than LeetCode's thousands, system design content is introductory rather than senior-level, and there's no live mock interview or human feedback loop. If you learn best from watching someone solve problems on screen, this isn't the format for you.

For a single problem like max area of island, you don't need to buy anything. The editorial exists, and the pattern is small enough to hold in your head after one careful read. The purchase case is about the sixteen patterns you *don't* yet recognize on sight.

## FAQ

**Does max area of island need BFS or DFS?**
Either. Both run in `O(m·n)` time and produce the same result. Pick DFS for shorter code, or BFS/iterative DFS if you want to sidestep Python's recursion limit on a large snake-shaped island.

**Can I solve it without a visited array?**
Yes — overwrite each visited `1` with `0` as you traverse it. That drops your extra space to `O(1)` outside the stack, at the cost of destroying the input grid.

**Is Union-Find overkill here?**
For the LeetCode constraints, yes. It's more code and more memory with no complexity win on a static grid. Mention it as an alternative and explain why you didn't pick it; that reads as judgment, not hedging.

**What's the space complexity of the recursive solution?**
`O(m·n)` in the worst case, because an island can fill the whole grid and the call stack follows it. People often quote `O(1)` by forgetting the stack — don't, because interviewers check.

👉 [Work through the grid traversal pattern with AlgoMonster's free tier](https://algo.monster?fpr=li69)
