# backtracking algorithm: The choose-explore-un-choose template, pruning rules, and complexity answers that survive follow-up questions

Most people learn the backtracking algorithm in the same broken order. They read a solution for Subsets, nod along, then open Permutations and freeze, because the code looks nothing like what they just memorized. Then comes Combination Sum and the recursion signature changes again.

It isn't three different algorithms. Subsets, permutations, combination sum, N-Queens, word search and Sudoku all run on the same three-step loop with a different pruning check bolted on. The loop is the easy part. Picking the right check, and defending the complexity afterward, is what the interview is actually testing.

This is the loop, the four questions that generate every variant, where pruning earns its keep, and the complexity phrasing that quietly loses points.

## What the backtracking algorithm is doing, mechanically

Backtracking is depth-first search over a decision tree that your code builds as it goes, rather than a graph handed to you in the input.

That distinction matters because interviewers ask about it. Plain DFS on a graph marks a node visited and never unmarks it, since reaching the same node twice adds nothing. Backtracking un-marks, because a value excluded from one branch may be perfectly legal in a sibling branch. You mutate state going down, restore it coming back up.

The other useful framing: backtracking is brute force that checks constraints during generation instead of after. For combination sum with target 7 and candidates `[2,3,6,7]`, brute force generates all 16 subsets and filters. Backtracking stops the moment the running sum exceeds 7. Same worst case, wildly different work in practice.

Where it gets confused with dynamic programming: memoization helps when you need a count or an optimum. If the question asks you to *list* all n! permutations, nothing beats n!, because the required output is already that large. Getting that output-sensitive point out loud is usually worth more than a clever optimization.

## The template, and the four questions that generate every variant

Every backtracking solution answers the same four questions before a line of code gets written:

1. **What is the path?** The partial answer built so far. Almost always a list you append to and pop from.
2. **What is the choice list at each level?** What is still available to pick.
3. **What is the base case?** When the current state is a complete, valid answer worth recording. Missing this rule means you never record anything.
4. **What prune applies?** Which branches provably cannot reach a valid answer.

The loop itself:

python
def dfs(start, path):
    if is_leaf(start):
        report(path[:])       # copy, not the live list
        return
    for edge in get_edges(start):
        path.append(edge)     # choose
        dfs(start + 1, path)  # explore
        path.pop()            # un-choose


That is the whole shape. `path.pop()` is not decoration. `path` is reused across the entire search, so a branch that does not restore its state corrupts every sibling that follows.

AlgoMonster teaches the same structure in its pattern materials under the names `start_index` and `path`, with a second variant for problems that aggregate a result up the tree instead of collecting leaves, which is how word break is handled. If you want to see the template split into its choose and un-choose halves with diagrams rather than text alone, 👉 [AlgoMonster's pattern-based lessons](https://algo.monster?fpr=li69) walk through it that way.

## The variants worth actually memorizing

### Subsets and combinations: the `start` index

Here the recursion carries an index that only moves forward. `[1,2]` and `[2,1]` are the same combination, and `start` is what prevents you from revisiting earlier elements and generating both.

python
def subsets(nums):
    res, path = [], []
    def dfs(start):
        res.append(path[:])
        for i in range(start, len(nums)):
            path.append(nums[i])
            dfs(i + 1)
            path.pop()
    dfs(0)
    return res


Everything that keeps the search small lives in that loop header.

### Permutations: the `used` array

Permutations care about order, so every unused element is a legal choice at every level. A `start` pointer would collapse it into combinations. Instead you carry a `used` array and skip anything already taken, which cuts the search space from O(n^n) down to O(n!).

The swap-based version trades the array for in-place swaps: swap `nums[i]` into position `k`, recurse, swap back. Same result, less auxiliary memory, and the un-choose step is the swap-back. Forgetting it is one of the most common silent bugs in the whole pattern.

### Combination sum and the first real prune

The single decision that trips people up is `i` versus `i + 1`. Recurse with `i + 1` and each candidate is used once. Recurse with `i` and the same candidate can be reused, which is exactly what combination sum needs.

Sorting first is what unlocks the strongest prune available:

python
candidates.sort()
for i in range(start, len(candidates)):
    if total + candidates[i] > target:
        break            # nothing later can fit either
    ...


The `break` is the payoff. Once the smallest remaining option overshoots the budget, every later option overshoots too.

Combinations of size *k* get a different, purely arithmetic prune: stop the loop when fewer numbers remain than slots still open, roughly `n - i + 1 < k - len(path)`. It sounds like a micro-optimization. On large inputs it removes entire subtrees before the recursion even starts.

### Duplicate handling, done correctly

When the input can contain duplicates, you sort and skip an element equal to the previous one at the same level. The condition has to be `i > start`, not `i > 0`. Writing `i > 0` also skips legitimate repeats on deeper levels, so subsets like `[2,2]` disappear from the output. Skipping before sorting does nothing at all, because equal values are not adjacent yet.

### N-Queens, word search, Sudoku

N-Queens is the clean example of constraint tracking replacing scanning. Three sets give O(1) conflict checks: columns, `row - col` for one diagonal family, `row + col` for the other. Each of those values is constant along its diagonal, which is why they work as keys. A solution exists for every n except 2 and 3, and n=8 has 92.

Grid problems follow the same loop over four neighbors, with the visited marker restored on the way out. Word search time is O(rows × cols × 4^L) for a word of length L. Sudoku is the pattern with the most aggressive prune, since each placement immediately kills an entire row, column and box; a typical puzzle finishes in microseconds despite a naive worst case around 9^81.

## Where the speed actually comes from

Two statements in an interview do all the work here:

- Pruning changes which branches get explored. It usually does not change the worst-case bound.
- Unless you can bound the pruned tree, "the pruning makes it fast" is not a complexity claim.

Saying "the bound is unchanged, but the constraint checks eliminate most of the tree near the root" survives the follow-up. Claiming "N-Queens is O(n²) because of pruning" without proof does not.

## Complexities interviewers expect, and the answers that lose points

| Problem | Search tree | Time | Auxiliary space (excluding output) | Answer that loses points |
| --- | --- | --- | --- | --- |
| Subsets of n | 2^n nodes, every node emits | O(n · 2^n) | O(n) | "O(2^n)", dropping the copy at each node |
| Permutations of n | n! leaves | O(n · n!) | O(n) | "O(n!)", same omission |
| Combination sum, target T, smallest candidate m | branching n, depth ≤ T/m | O(n^(T/m)) | O(T/m) | "O(2^n)", when n is not the depth |
| Restore IPv4 addresses | ≤ 3^4 leaves, depth fixed at 4 | O(1) under the 12-digit cap | O(1) | anything exponential in string length |
| Sudoku, k blanks | ≤ 9 choices per cell | O(k · 9^k) | O(k) | "MRV makes it polynomial" |

The gap between the two growth rates is why permutation questions arrive with smaller inputs than subset questions. At n=10, subsets give 1,024 results and permutations give 3,628,800. At n=15 that becomes 32,768 versus roughly 1.3 trillion. Recursion depth is only O(n) in both cases, so the call stack stays small and the result list is the real memory cost.

## Bugs that keep showing up

- **Storing the live list.** `res.append(path)` records a reference, so every later `pop()` mutates what you already stored. You get a list of empty lists. Use `path[:]` or `list(path)`.
- **Skipping the un-choose.** The code still runs, which is why it is hard to spot. The second option on a level gets appended after the first instead of replacing it.
- **Not restoring shared state.** Anything mutated before the recursive call has to be reverted after it, including `used` flags, board cells, and `visited` sentinels.
- **Off-by-one in the duplicate skip.** `i > start`, not `i > 0`.

## How to tell a problem wants backtracking

Constraints are the loudest signal. A subset problem with n ≤ 20, or a permutation problem with n ≤ 10, is an invitation to enumerate. n = 10^5 is not, however naturally the question phrases itself as "find all".

Wording is the second signal. "Find all", "list every", "return all possible" point at enumeration. "How many" or "what is the best" should send you toward DP or greedy first. Counting paths in a grid, word break and coin change all have exponential backtracking solutions and polynomial-time alternatives.

The exceptions are worth knowing, because interviewers plant them. Some questions put "permutation" or "backtracking" in the title and want something else entirely: a single sweep, or a merge sort counting inversions. Reading the constraint and the required output shape tells you more than the label does.

## How to say it out loud

Three sentences cover the approach for any of these problems:

> I build the answer one choice at a time. At each step I try every legal next choice, recurse, then undo the choice. I prune a branch as soon as it cannot produce a valid answer.

Then state the base case, the prune, and the complexity including the O(n) copy cost per result. That structure is what separates a candidate who can code the recursion from one who understands it.

## Where AlgoMonster fits into this

The reason backtracking feels slippery is that it is a *pattern*, not a topic with a fixed signature, and most problem banks are organized by topic. You can solve forty array problems and never learn that the same five-line loop answers all six of the problems above.

AlgoMonster builds its curriculum around that idea. It was put together by ex-Google engineers and competitive programmers, and the content is pattern-first: each pattern gets a conceptual explanation, a code template, a decision flowchart, and a set of problems that are variations on the same structure rather than a random sample. The site quotes a Reddit user describing it as "like a textbook with an emphasis on interview prep," which is a fair description of the format, since there is no video at all. Everything is text, diagrams and runnable code.

Its decision flowchart is the part most relevant here. When you walk through it, "small constraints?" routes you to DFS/backtracking, which is the same constraint-reading instinct described above, formalized into a lookup. The official plan page currently lists unrestricted access to all patterns, lessons and problems, unlimited AI assistant usage, unlimited Speedrun practice, and 500+ lessons across 6 courses with 1,000+ illustrations and 3,000+ editorials. Two bonus courses, a computer science foundation course and a build-an-LLM course, are included; specialized partner content such as quant interview prep is sold separately.

Where it genuinely falls short: there are no video walkthroughs, the problem volume is a few hundred rather than LeetCode's few thousand, system design coverage is introductory, and there is no live coaching or mock interview component. Several third-party reviews make the same point, that most candidates pair it with a larger problem bank for raw repetition. If your gap is volume, this is not the tool. If your gap is that you cannot tell which technique a problem wants, it is aimed directly at that.

## AlgoMonster plans and current listed pricing

There is no monthly option. The listed choice is annual or lifetime, plus a free tier that exposes a subset of the material.

| Plan | Price as currently listed | Billing | What you get |
| --- | --- | --- | --- |
| Free | $0 | No card required | Limited access to a subset of patterns, lessons and problems |
| Pro Yearly | $99/year ($8.25/mo) | Billed annually, cancel or upgrade anytime | All patterns, lessons and problems; unlimited AI assistant; unlimited Speedrun; company question banks updated monthly; 500+ lessons, 1,000+ illustrations, 3,000+ editorials; bonus foundation and LLM courses; member discount on live bootcamps |
| Pro Lifetime | $189 | One-time payment | Everything in the annual plan, plus lifetime updates with no renewal |

👉 [Compare the free and Pro plans](https://algo.monster?fpr=li69)

👉 [See the current Pro Yearly price](https://algo.monster?fpr=li69)

👉 [See the current Pro Lifetime price](https://algo.monster?fpr=li69)

A caveat worth stating plainly: AlgoMonster runs frequent promotions, and the discounts on the site fluctuate. Third-party write-ups published at different times cite annual figures from $89 to $299 and lifetime figures from $99 to $459, which reflects sale timing rather than different products. The numbers above are what the plan page showed during this writing. Check the page itself before you decide, and treat any coupon code you find elsewhere as unverified unless the checkout accepts it.

For the decision itself: if you are prepping for interviews in the next two to three months, the annual plan is the cheaper commitment. If you expect to interview again in two or three years, the lifetime plan settles the question for roughly the price of two months of a recruiter's time.

## Questions that come up

**Is backtracking the same as DFS?**
No. Both go deep first, but backtracking runs on a decision tree your code constructs and undoes each choice on the way back up. Graph DFS marks nodes permanently visited.

**Why is my output a list of empty lists?**
You appended the live `path` instead of a copy. `path[:]` fixes it.

**Why do subsets with duplicates come out wrong?**
Either the input is not sorted before the skip check, or the check uses `i > 0` instead of `i > start`.

**Do I need to state pruning separately in the complexity answer?**
Yes, and state that it does not change the worst-case bound unless you can prove it does.

**Is n! ever avoidable?**
Not when the required output is all n! permutations. That is why "can we memoize this" is usually the wrong question for an enumeration problem.
