# backtracking vs recursion: how to tell them apart, write both without the state bug, and know when a problem needs a search tree

If you searched "backtracking vs recursion," you're probably in one of two spots. Either you wrote a recursive function that spits out garbage and someone told you the fix is "backtracking," or your prep sheet lists them as two separate patterns and you can't tell whether that's two concepts or one concept wearing two hats.

Mostly it's one concept wearing two hats. Recursion is how you write the code. Backtracking is what you're doing with it. The confusion sticks around because almost every teaching example of backtracking is recursive, which makes it look like backtracking *is* recursion, the way driving looks like sitting in a car.

|  | Recursion | Backtracking |
| --- | --- | --- |
| What it actually is | A way of writing a function that calls itself | A search strategy for exploring many partial solutions |
| Work per call | One subproblem, usually one child call | One decision, usually several child calls |
| What `return` means | "Here's the answer to the subproblem" | "This branch is dead, unwind and try the next one" |
| Where state lives | Mostly on the call stack | Explicitly mutated, then undone |
| Typical problems | factorial, tree traversal, merge sort, reverse a linked list | subsets, permutations, N-Queens, Sudoku, word search |
| Can it exist alone | Yes, plenty of recursion never backtracks | Yes, you can write backtracking with a stack and a loop |

## Recursion, in the form you'll actually use

A recursive function has two parts: a base case that stops the descent, and a recursive case that makes the problem smaller. That's it. The interesting detail is that Python, Java, and C++ keep each unfinished call on the call stack, so when the deepest call finally returns, every caller above it resumes with its local variables intact.

That stack is why basic recursion is good at problems with a self-similar shape:

python
def factorial(n):
    if n <= 1:
        return 1              # base case
    return n * factorial(n - 1)   # smaller subproblem


Nothing is being undone here. Each call hands a value back up the chain, and the final answer is assembled on the way out. The same shape handles tree depth, merge sort, reversing a linked list, and most divide-and-conquer algorithms. One call in, one answer out.

If your function only ever calls itself once per level and every call contributes a piece to a single result, you're doing plain recursion. Backtracking is not involved, and adding it would only create bugs.

## What backtracking adds: choose, explore, unchoose

Backtracking shows up when a problem has *many* valid next moves at each step and you need to try them, keep the ones that work, and abandon the ones that can't. You're walking a decision tree, depth-first, and the discipline that makes it work is undoing your choice before you try the alternative.

The canonical three-step loop:

python
def subsets(nums):
    result, path = [], []

    def dfs(i):
        if i == len(nums):
            result.append(path[:])   # snapshot of a complete solution
            return
        path.append(nums[i])         # choose
        dfs(i + 1)                   # explore
        path.pop()                   # unchoose
        dfs(i + 1)                   # explore the branch without nums[i]

    dfs(0)
    return result


Read the `return` on line 6 as "this path is finished, go back up." Read `path.pop()` as "take back the decision I just made so my sibling can make a different one." In basic recursion, a return carries data upward. In backtracking, a return means *undo and continue*.

That's the whole mental shift. The function shape is identical; the job is different.

## Three flavors of backtracking problems

Recursion problems with backtracking tend to fall into one of three buckets, and knowing which bucket you're in tells you whether you can stop early:

1. **Generate everything** — all subsets, all permutations, all valid parentheses. You hit the base case every time and collect; there's no early exit.
2. **Find one** — does a path exist, is this board solvable, does a valid arrangement exist. Here you return `True` up the stack the moment you find a solution, which skips an enormous amount of work.
3. **Find the best** — the cheapest path, the largest arrangement. You explore everything but track the best-so-far, and prune branches that can't beat it.

Bucket 2 and 3 are where interviewers probe hardest, because the difference between "it's correct" and "it finishes before the interview ends" is usually the return-early logic.

## Pruning: where backtracking actually gets fast

Plain backtracking is exponential. Permutations of 10 items is 3,628,800 arrangements; a naive N-Queens search over 8 columns considers up to 8^8 = 16,777,216 placements. Nobody's sitting through that.

The fix is checking constraints *before* you recurse, not after. In N-Queens you track which columns and both diagonal families are already occupied, and refuse a placement that collides:

python
def solve_n_queens(n):
    cols, diag, anti = set(), set(), set()
    placements = []

    def place(row):
        if row == n:
            return True
        for col in range(n):
            if col in cols or (row - col) in diag or (row + col) in anti:
                continue                      # prune before recursing
            cols.add(col); diag.add(row - col); anti.add(row + col)
            placements.append((row, col))
            if place(row + 1):
                return True                   # found one solution; unwind
            placements.pop()                  # undo
            cols.remove(col); diag.remove(row - col); anti.remove(row + col)
        return False

    return placements if place(0) else []


Two things to notice. The constraint check happens before the recursive call, which is what actually shrinks the tree. And every mutation has a matching undo after the call returns — if you delete the three `remove` calls, the function still "runs" but produces wrong answers, because the state from a rejected branch leaks into the next one.

Word search on a grid uses the same discipline with a `visited` flag: mark the cell, recurse in four directions, unmark on the way out. Palindrome partitioning does it with an index and a growing list of pieces, checking `s[left:right] == reversed` before descending.

> A useful rule of thumb: if you find yourself checking a constraint *after* the recursive call returns, you're paying for the whole subtree and then throwing it away. Move the check up one line and the same code gets dramatically faster.

## Complexity: what to say out loud

In interviews, the expected answer for backtracking is usually about the shape of the search tree: `O(b^d)` nodes, where `b` is the branching factor and `d` the depth, times whatever work you do per node. Then you refine it per problem — `O(n · n!)` for permutations, `O(2^n · n)` for subsets if you copy the path at each leaf, roughly `O(n!)` nodes explored for N-Queens *before* pruning.

Saying "exponential, but the pruning cuts it to something practical for n ≤ 9" is a better answer than pretending N-Queens is polynomial. Interviewers generally care that you know why it's exponential and what reduces the constant.

## Backtracking without recursion, and recursion without backtracking

Two clarifications that clear up most of the confusion:

**Backtracking doesn't require recursion.** You can generate subsets with a loop and no stack at all:

python
def subsets_iterative(nums):
    result = [[]]
    for x in nums:
        result += [cur + [x] for cur in result]
    return result


Every recursive backtracking solution can be rewritten with an explicit stack holding `(index, path)` pairs. The recursion is a convenience, not a requirement. What makes it *backtracking* is the exploring-and-abandoning, not the syntax.

**Recursion doesn't imply backtracking.** Traveling a binary tree to compute its depth is recursion. There's exactly one path down, nothing to choose, nothing to undo. Same for merge sort, binary search, and computing Fibonacci.

And a third one people miss: backtracking isn't a synonym for DFS. DFS is the traversal order. Backtracking adds two things on top — a validity check that prunes branches early, and the undo discipline that keeps state clean between siblings.

## Mistakes that break backtracking code

These seven account for most wrong answers in practice:

- **Appending `path` instead of `path[:]`.** Every result in your output list points at the same live list, and they all end up empty or identical by the time the function returns.
- **Forgetting the undo.** A `visited` flag or a `set` membership that never gets removed poisons every later branch.
- **Choosing the undo location carelessly.** The `pop()` has to sit after the recursive call and before the sibling branch, not before the recursive call.
- **Pruning too late.** Checks placed after recursion add overhead without cutting anything.
- **Base case off by one.** `i == len(nums)` versus `i >= len(nums)` matters the moment you index an array.
- **Deep recursion crashing.** Python's default recursion limit is around 1,000 frames. A depth-2,000 search needs an iterative rewrite or an explicit stack. This is exactly why "backtracking isn't recursion" is a practical fact and not a trivia point.
- **Skipping deduplication.** For "subsets with duplicates" or "combination sum," sort the input and skip `nums[i-1] == nums[i]` at the same level, or you'll emit the same result repeatedly.

## How to tell which one a problem wants

Signal-based reading works better than memorizing problem lists:

- "Generate all…", "return any…", "count the number of ways…", "is it possible to…" with a choice at each step → backtracking.
- Constraints that let you reject a partial answer (no two queens attack, the word must continue from an adjacent cell) → backtracking with pruning, and you should move the check as early as possible.
- A single value derived from a smaller version of the same input → plain recursion.
- Overlapping subproblems where you keep recomputing the same state → that's dynamic programming, not plain backtracking. If your recursive solution memoizes cleanly, you probably want DP.

In a real interview loop, the follow-ups are where candidates split. "How would you prune this?", "what's the worst-case complexity after your last optimization?", "what happens if the board is 12×12 instead of 8×8?" Those questions test whether you understand the search tree rather than whether you can reproduce a template.

## Where to drill recursion and backtracking

Pattern-recognition on recursion problems is a skill you build by seeing the same structure repeated with small variations — subsets first, then permutations, then combinations with a target, then partitioning, then constraint satisfaction. Doing that in the right order matters more than doing 200 random problems.

AlgoMonster is built around exactly this idea. It's a pattern-based course from ex-Google engineers and competitive programmers, with dedicated backtracking lessons (the site has separate Backtracking 1 and Backtracking 2 units) alongside the recursion material, code templates you adapt instead of memorize, and decision flowcharts for the "which approach does this problem want" step. The format is text plus interactive diagrams with an inline code editor, no videos. 👉 [👉 Check the backtracking and recursion lessons here](https://algo.monster/subscribe?fpr=li69)

The tradeoffs worth knowing before you pay: the problem set is smaller than LeetCode's (roughly 230 problems versus 3,000+), system design coverage is introductory and won't carry an L5+ loop, and there's no live coaching or mentorship. If your gap is implementation speed and you already write correct code, the pattern framework is the useful part. If you need raw problem volume, you'll end up pairing it with something else.

### AlgoMonster plans and current pricing

The platform has no team or enterprise tier — it's built for individuals. Prices move with promotions, and the same subscribe page has listed noticeably different numbers at different points, so check before buying rather than trusting a cached screenshot.

| Plan | Core inclusions | Price | Billing cycle | Link |
| --- | --- | --- | --- | --- |
| Free | Limited sample lessons and problems, enough to test whether the text-and-diagram style works for you | $0 | — | [Start with the free content](https://algo.monster/subscribe?fpr=li69) |
| Pro Monthly | All patterns, lessons and problems; unlimited AI assistant; unlimited Speedrun; company question banks updated monthly; inline IDE and templates | $18.99/mo | Monthly, cancel or upgrade anytime | [See the monthly plan](https://algo.monster/subscribe?fpr=li69) |
| Pro Yearly | Everything in Pro, 12 months of full access, member discount on live bootcamps, bonus Foundation and Build-an-LLM courses included | $99/year (~$8.25/mo); has also been listed at $69.99/year during deeper promotions | Billed annually | [See the yearly plan](https://algo.monster/subscribe?fpr=li69) |
| Pro Lifetime | Everything in Yearly plus lifetime updates on all current and future Pro content | $189 one-time; has also been listed at $89 during deeper promotions | Billed once | [See the lifetime plan](https://algo.monster/subscribe?fpr=li69) |

A few practical notes on that table. The lifetime option is the one that pays for itself if you expect to interview more than once every couple of years — otherwise a single annual pass covers the four-to-eight-week curriculum comfortably. Promo codes circulate through tech blogs and YouTube channels and typically stack on top of whatever sale is already running. And if you're most interested in the recursion-to-backtracking progression specifically, the free tier plus one paid month is enough to judge whether the format suits how you learn; nothing about the curriculum requires the lifetime purchase to be useful.

## The five-second version

Recursion is a function calling itself. Backtracking is a search that builds partial solutions, checks them early, and takes back choices that don't pan out. Every backtracking solution you'll write in an interview will probably use recursion, and almost none of the recursion you write day to day will use backtracking. Learn the call stack first, then learn the undo. The undo is where the bugs live.
