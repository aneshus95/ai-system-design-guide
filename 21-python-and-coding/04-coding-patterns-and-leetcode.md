# Coding Patterns & LeetCode — Cracking the Coding Round

A pattern-first guide to LeetCode-style coding interviews. Instead of memorizing hundreds of individual problems, you learn ~30 **reusable patterns** — each with the signals that tell you to reach for it, the intuition in plain English, a Python template you can adapt, complexity, named LeetCode problems, and the pitfalls that trip people up. Master the patterns and most "new" problems become variations you already know.

**In plain English:** Interviewers don't test whether you've seen a specific problem — they test whether you can *recognize the shape* of a problem and map it to a technique. "Longest substring with a constraint" → sliding window. "Kth largest" → heap. "Prerequisites/ordering" → topological sort. This page is a lookup table from *problem wording* to *the tool that solves it*, plus how to run a 45-minute coding round.

> Pattern taxonomy cross-checked against the three canonical study lists: [NeetCode 150](https://neetcode.io/practice?tab=neetcode150), [Blind 75](https://leetcode.com/discuss/post/460599/blind-75-leetcode-questions-by-krishnade-9xev/), and [Grokking the Coding Interview](https://www.designgurus.io/course/grokking-the-coding-interview). Every LeetCode number links to the live problem.

---

## Table of Contents

1. [How to Use This Guide](#how-to-use-this-guide)
2. [Array & String Patterns](#array--string-patterns) — two pointers, sliding window, prefix sum, hashing, Kadane's, intervals, cyclic sort, fast/slow
3. [Search, Recursion & Math Patterns](#search-recursion--math-patterns) — binary search, binary search on the answer, backtracking, divide & conquer, bit manipulation, greedy, number theory
4. [Linked List, Stack, Tree & Heap Patterns](#linked-list-stack-tree--heap-patterns) — pointer surgery, monotonic stack, BFS, tree traversals, BST, trie, heaps/top-K
5. [Graph & Dynamic Programming Patterns](#graph--dynamic-programming-patterns) — DFS/BFS, topological sort, union-find, shortest path, 1D/2D DP, knapsack, LIS, the DP recipe
6. [Cracking the Coding Round — Strategy](#cracking-the-coding-round--strategy) — the 7-step approach, signal→pattern table, communication, testing, time management, scoring
7. [Curated Study Plan](#curated-study-plan) — a pattern-ordered problem list
8. [Glossary](#glossary)
9. [References](#references)

---

## How to Use This Guide

- **Learn patterns, not problems.** For each pattern, first read *"When to reach for it"* and *"In plain English"* — that's what you actually recall under pressure. The template and problems reinforce it.
- **Recall the trigger, then the template.** In a real interview you won't have the code in front of you. Practice recognizing the pattern from the wording (see the [signal→pattern table](#signal--pattern-cheat-table)), then reconstruct the template from the intuition.
- **Drill 2–3 problems per pattern**, not 50 random problems. Depth on the pattern transfers; breadth on unrelated problems does not.
- **Time-box.** After you can recognize the pattern, practice the full [45-minute round flow](#time-management-in-a-45-minute-round): clarify → examples → brute force → optimize → code → test → analyze.
- **Related pages in this guide:** [Python Core Concepts](01-python-core-concepts.md) · [OOP in Python](02-oop-in-python.md) · [DS Coding & SQL Practice](03-ds-coding-and-sql-practice.md).

---

## Array & String Patterns

> **Sources for this section:** [NeetCode 150](https://neetcode.io/practice/practice/neetcode150) · [Blind 75 (NeetCode)](https://neetcode.io/practice/practice/blind75) · [Grokking the Coding Interview — DesignGurus](https://www.designgurus.io/course/grokking-the-coding-interview) · [Cyclic Sort Pattern](https://emre.me/coding-patterns/cyclic-sort/) · [Fast & Slow Pointers](https://emre.me/coding-patterns/fast-slow-pointers/)

### Two Pointers

**When to reach for it:**
- The array or string is **sorted** (or can be treated as ordered) and you need a **pair, triplet, or partition** satisfying a numeric constraint.
- Problem says "find two numbers that sum to target," "check if palindrome," "remove duplicates in-place," or "trap rainwater."
- Opposite-ends variant: start one pointer at index 0, one at the last index, converge inward. Same-direction variant: both start at 0, one races ahead (also called the fast/slow variant for partitioning).

**In plain English:**
Instead of testing every pair with two nested loops (O(n²)), you exploit order: if the current pair-sum is too large, shrink it by moving the right pointer left; if too small, grow it by moving the left pointer right. You converge on the answer in a single pass. Think of two people walking toward each other on a sorted number line — they can skip huge swaths of useless combinations.

**Template:**
```python
def two_pointers_opposite(arr):
    left, right = 0, len(arr) - 1
    while left < right:
        current = arr[left] + arr[right]   # or any comparison
        if current == target:
            return [left, right]
        elif current < target:
            left += 1
        else:
            right -= 1
    return -1

def two_pointers_same_direction(arr):
    slow = 0
    for fast in range(len(arr)):
        if condition(arr[fast]):           # e.g., arr[fast] != val
            arr[slow] = arr[fast]
            slow += 1
    return slow                            # new length
```

**Complexity:** Time O(n) · Space O(1)

**Representative problems:**
- **[Container With Most Water (LC 11)](https://leetcode.com/problems/container-with-most-water/)** — Opposite-end pointers; always move the shorter wall inward to maximize potential area.
- **[Valid Palindrome (LC 125)](https://leetcode.com/problems/valid-palindrome/)** — Skip non-alphanumerics, compare `arr[left]` and `arr[right]`, converge inward.
- **[3Sum (LC 15)](https://leetcode.com/problems/3sum/)** — Fix one element with an outer loop, then run opposite-end two-pointer on the remainder; skip duplicates explicitly.

**Common pitfalls:**
- Using `left < right` vs `left <= right` — for most pair problems you want strict `<`; for three-pointer variants the boundary differs.
- Forgetting to skip duplicate values in 3Sum/4Sum, producing duplicate triplets in the result.
- Applying opposite-end two pointers to an **unsorted** array — the core invariant breaks; sort first or use a hash map instead.
- Off-by-one when the array has length 1 or 2 — always test the minimum valid input.

### Sliding Window

**When to reach for it:**
- Problem asks for the **longest / shortest contiguous subarray or substring** satisfying some constraint (e.g., "with at most K distinct characters," "sum ≥ target," "no repeating characters").
- Keywords: "contiguous," "subarray/substring," "window," "minimum length," "maximum sum."
- **Fixed-size window**: "find max average of subarray of length k." **Variable-size window**: "minimum size subarray with sum ≥ S."

**In plain English:**
Imagine a rubber-band window sliding across the array. You expand the right edge to include new elements and shrink the left edge whenever the window violates your constraint. Because each element enters and exits the window at most once, the total work is O(n) rather than O(n²) for all subarray pairs. Fixed windows just slide rigidly; variable windows breathe.

**Template:**
```python
# Variable-size window
from collections import defaultdict

def sliding_window(s, k):
    counts = defaultdict(int)
    left = 0
    result = 0                             # or float('inf') for minimum

    for right in range(len(s)):
        counts[s[right]] += 1             # expand window

        while window_invalid(counts, k):  # shrink until valid
            counts[s[left]] -= 1
            if counts[s[left]] == 0:
                del counts[s[left]]
            left += 1

        result = max(result, right - left + 1)

    return result

# Fixed-size window (size = k)
def fixed_window(arr, k):
    window_sum = sum(arr[:k])
    best = window_sum
    for i in range(k, len(arr)):
        window_sum += arr[i] - arr[i - k]
        best = max(best, window_sum)
    return best
```

**Complexity:** Time O(n) · Space O(k) where k is the number of distinct elements tracked

**Representative problems:**
- **[Longest Substring Without Repeating Characters (LC 3)](https://leetcode.com/problems/longest-substring-without-repeating-characters/)** — Variable window; shrink `left` whenever a duplicate enters the window.
- **[Minimum Window Substring (LC 76)](https://leetcode.com/problems/minimum-window-substring/)** — Variable window tracking character frequency; use a `have/need` counter to know when all target chars are satisfied.
- **[Best Time to Buy and Sell Stock (LC 121)](https://leetcode.com/problems/best-time-to-buy-and-sell-stock/)** — Implicit fixed window of 2; track running minimum buy price and update max profit each day.

**Common pitfalls:**
- Shrinking the window with `if` instead of `while` in variable windows — the inner loop must keep shrinking until the window is valid again.
- Not resetting or correctly decrementing the count map when `left` advances.
- For minimum-length problems, initializing `result = 0` instead of `float('inf')` and forgetting to check if the window ever became valid.
- Fixed window: forgetting to compute the initial window sum before the loop begins.

### Prefix Sum / Running Totals

**When to reach for it:**
- Multiple **range-sum queries** on a static array, or you need to count subarrays/substrings where the sum (or XOR, product) equals a target.
- Keywords: "subarray sum equals K," "number of subarrays," "range sum query," "pivot index," "balance point."
- Spotting signal: you find yourself re-summing overlapping ranges — a prefix array eliminates all redundant work.

**In plain English:**
Build an array `prefix` where `prefix[i]` = sum of `arr[0..i-1]`. Any subarray sum `arr[l..r]` then becomes `prefix[r+1] - prefix[l]` — a single O(1) subtraction. For counting subarrays with sum = k, combine the prefix array with a hash map: you've seen `prefix[r+1] - k` before exactly as many times as there are valid subarrays ending at `r`.

**Template:**
```python
# Range-sum query
def build_prefix(arr):
    prefix = [0] * (len(arr) + 1)
    for i, val in enumerate(arr):
        prefix[i + 1] = prefix[i] + val
    return prefix

def range_sum(prefix, l, r):            # inclusive [l, r]
    return prefix[r + 1] - prefix[l]

# Count subarrays with sum == k
from collections import defaultdict

def subarray_sum_equals_k(nums, k):
    count = defaultdict(int)
    count[0] = 1
    running = 0
    result = 0
    for n in nums:
        running += n
        result += count[running - k]    # subarrays ending here with sum k
        count[running] += 1
    return result
```

**Complexity:** Time O(n) build + O(1) query · Space O(n)

**Representative problems:**
- **[Subarray Sum Equals K (LC 560)](https://leetcode.com/problems/subarray-sum-equals-k/)** — Prefix sum + hash map; count how many times `running - k` has appeared.
- **[Range Sum Query — Immutable (LC 303)](https://leetcode.com/problems/range-sum-query-immutable/)** — Classic prefix array; answer each query in O(1) after O(n) build.
- **[Find Pivot Index (LC 724)](https://leetcode.com/problems/find-pivot-index/)** — Total sum minus prefix: check `left_sum == total - left_sum - nums[i]`.

**Common pitfalls:**
- Forgetting `count[0] = 1` before the loop — misses subarrays that start from index 0.
- Off-by-one in prefix array indexing: `prefix[r+1] - prefix[l]` vs `prefix[r] - prefix[l-1]` — pick one convention and stick with it.
- Using prefix sum when the array has **negative numbers and you need a subarray maximum** — that's Kadane's territory, not prefix sum.
- Assuming prefix sum works for sliding-window-style variable-constraint problems; it doesn't replace the window when the constraint is not purely additive.

### Hashing & Frequency Counting

**When to reach for it:**
- You need to check **membership, duplicates, anagrams, or frequency** in O(1) per query.
- Keywords: "contains duplicate," "valid anagram," "two sum," "group anagrams," "top K frequent," "first unique character."
- Any time a brute-force solution would use a nested loop to look up values — replace the inner loop with a hash map.

**In plain English:**
A hash map is a memo pad: as you walk through the data once, you write down what you've seen and look things up in constant time. For two-sum you store `target - num` as you go; for frequency counting you tally each element; for anagram grouping you use sorted characters or a tuple of 26 counts as a key. The trade-off is O(n) extra space for O(n) time.

**Template:**
```python
# Two-sum style (complement lookup)
def two_sum(nums, target):
    seen = {}                            # value -> index
    for i, n in enumerate(nums):
        complement = target - n
        if complement in seen:
            return [seen[complement], i]
        seen[n] = i
    return []

# Frequency count / anagram
from collections import Counter

def is_anagram(s, t):
    return Counter(s) == Counter(t)

def group_anagrams(words):
    groups = defaultdict(list)
    for w in words:
        groups[tuple(sorted(w))].append(w)
    return list(groups.values())
```

**Complexity:** Time O(n) · Space O(n)

**Representative problems:**
- **[Two Sum (LC 1)](https://leetcode.com/problems/two-sum/)** — Single-pass hash map storing `value → index`; look up complement before storing current.
- **[Valid Anagram (LC 242)](https://leetcode.com/problems/valid-anagram/)** — Compare `Counter(s)` vs `Counter(t)`, or use a 26-element array for pure lowercase ASCII.
- **[Top K Frequent Elements (LC 347)](https://leetcode.com/problems/top-k-frequent-elements/)** — `Counter` + `heapq.nlargest`, or bucket sort by frequency for O(n) solution.

**Common pitfalls:**
- Using a list/set when you need the **index** — sets discard position; use a dict mapping value to index.
- Using a list as a dict key raises `TypeError`; use `tuple` instead.
- Forgetting that `Counter` subtraction drops zero/negative counts — use `.get()` or explicit checks when decrementing.
- In "first unique" problems, relying on dict insertion-order (guaranteed in Python 3.7+) consciously rather than by accident.

### Kadane's / Maximum Subarray

**When to reach for it:**
- Find the **contiguous subarray with the maximum (or minimum) sum**.
- Keywords: "maximum subarray sum," "largest sum contiguous," "maximum product subarray" (variant).
- If you see "subarray" + "maximum" + no mention of a fixed size or a constraint, Kadane's is almost always the answer.

**In plain English:**
Walk left to right keeping a "running sum." At each element you decide: is it better to extend the existing subarray, or start fresh from this element? If the running sum drops below zero it's deadweight — drop it and restart. Meanwhile, record the global maximum seen so far. The insight is that a negative prefix can only hurt a future subarray.

**Template:**
```python
def max_subarray(nums):
    max_sum = nums[0]
    current = nums[0]

    for n in nums[1:]:
        current = max(n, current + n)   # extend or restart
        max_sum = max(max_sum, current)

    return max_sum
```

**Complexity:** Time O(n) · Space O(1)

**Representative problems:**
- **[Maximum Subarray (LC 53)](https://leetcode.com/problems/maximum-subarray/)** — The canonical Kadane's problem; return the maximum sum.
- **[Maximum Product Subarray (LC 152)](https://leetcode.com/problems/maximum-product-subarray/)** — Kadane's variant tracking both `max_prod` and `min_prod` at each step (negatives flip sign).
- **[Best Time to Buy and Sell Stock (LC 121)](https://leetcode.com/problems/best-time-to-buy-and-sell-stock/)** — Equivalent to max-subarray on the daily price-difference array.

**Common pitfalls:**
- Initializing `max_sum = 0` instead of `nums[0]` — fails when all elements are negative (answer should be the least-negative element).
- Confusing this with "maximum sum subsequence" (non-contiguous) — that's a different DP problem.
- For the product variant, forgetting that two negatives multiply to a positive — you must track the running minimum as well as maximum.
- Empty array input: guard with `if not nums: return 0` before the algorithm.

### Merge Intervals

**When to reach for it:**
- Problem gives a list of **intervals** (pairs `[start, end]`) and asks to merge, insert, find gaps, or count overlaps/meetings.
- Keywords: "meeting rooms," "overlapping intervals," "insert interval," "minimum number of platforms/arrows."
- Trigger: any time you have ranges on a number line that may overlap.

**In plain English:**
Sort intervals by start time. Then walk through: if the current interval's start is ≤ the previous interval's end, they overlap — merge by extending the end to the maximum of both ends. Otherwise, the current interval is disjoint — push the previous one to output and start tracking the new one. Sorting is the key that lets you process intervals linearly without comparing every pair.

**Template:**
```python
def merge_intervals(intervals):
    if not intervals:
        return []

    intervals.sort(key=lambda x: x[0])
    merged = [intervals[0]]

    for start, end in intervals[1:]:
        if start <= merged[-1][1]:                  # overlap
            merged[-1][1] = max(merged[-1][1], end) # extend
        else:
            merged.append([start, end])

    return merged
```

**Complexity:** Time O(n log n) dominated by sort · Space O(n) for output

**Representative problems:**
- **[Merge Intervals (LC 56)](https://leetcode.com/problems/merge-intervals/)** — Sort by start, greedily merge; the classic template problem.
- **[Insert Interval (LC 57)](https://leetcode.com/problems/insert-interval/)** — Collect all non-overlapping intervals before/after, merge the overlapping block, reassemble.
- **[Meeting Rooms II (LC 253)](https://leetcode.com/problems/meeting-rooms-ii/)** — Count concurrent meetings; use a min-heap of end times or the "events" sweep-line approach.

**Common pitfalls:**
- Forgetting to sort before merging — without sorting, non-adjacent overlapping intervals are missed.
- Using `end < merged[-1][1]` instead of `start <= merged[-1][1]` to test overlap — test the **start** of the new interval against the **end** of the last merged.
- Mutating `intervals` in-place while iterating — sort a copy or index carefully.
- Off-by-one on touching intervals: `[1,3]` and `[3,5]` — decide whether `start == end` counts as overlap (usually yes: `<=` not `<`).

### Cyclic Sort / Index-as-Hash

**When to reach for it:**
- Array contains integers in a **known range `[1, n]` or `[0, n]`** and the problem asks to find **missing, duplicate, or misplaced** numbers **in O(n) time and O(1) space**.
- Keywords: "find missing number," "find all duplicates," "first missing positive," "numbers in range 1 to n."
- The giveaway: you're not allowed extra space, but the range is bounded — the array itself is the hash table.

**In plain English:**
Think of each value as a "homing pigeon" that belongs at a specific index: value `v` belongs at index `v-1` (for 1-indexed ranges). Walk the array; whenever the current element is not in its home slot, swap it there. After the whole array is sorted this way in O(n) total swaps, a second pass finds any index where `arr[i] != i+1` — those are your missing or duplicate values.

**Template:**
```python
def cyclic_sort(nums):
    i = 0
    while i < len(nums):
        correct = nums[i] - 1           # where this number belongs
        if 1 <= nums[i] <= len(nums) and nums[i] != nums[correct]:
            nums[i], nums[correct] = nums[correct], nums[i]  # swap home
        else:
            i += 1
    return nums

def find_missing(nums):
    cyclic_sort(nums)
    for i, n in enumerate(nums):
        if n != i + 1:
            return i + 1
    return len(nums) + 1
```

**Complexity:** Time O(n) — each element moves to its correct slot at most once · Space O(1)

**Representative problems:**
- **[Missing Number (LC 268)](https://leetcode.com/problems/missing-number/)** — Range `[0, n]`; after cyclic sort find the first misplaced index, or use the math formula `n*(n+1)/2 - sum(nums)`.
- **[Find All Numbers Disappeared in an Array (LC 448)](https://leetcode.com/problems/find-all-numbers-disappeared-in-an-array/)** — After sort, collect every index where `nums[i] != i+1`.
- **[First Missing Positive (LC 41)](https://leetcode.com/problems/first-missing-positive/)** — Ignore out-of-range values; after sort, first `nums[i] != i+1` gives the answer in O(n)/O(1).

**Common pitfalls:**
- Swapping into an already-correct slot and creating an infinite loop — always check `nums[i] != nums[correct]` **before** swapping.
- Handling out-of-range values (negatives, zeros, values > n) — skip them with the `else: i += 1` branch, do not try to place them.
- For LC 268 (range `[0, n]`), the home index is `nums[i]` not `nums[i]-1` — adjust the formula for zero-based ranges.
- Confusing "cyclic sort" (an in-place technique) with cycle detection (Floyd's algorithm) — they are unrelated despite the word "cyclic."

### Fast & Slow Pointers

**When to reach for it:**
- Detect a **cycle** in a linked list or in an array treated as an implicit linked list (e.g., `next = nums[i]`).
- Find the **middle of a linked list** in one pass.
- Problems mentioning "cycle," "duplicate in array without extra space," "happy number," or "find the start of the loop."

**In plain English:**
Put a tortoise (slow) and a hare (fast) at the start. Slow moves one step per tick; fast moves two. If there is a cycle, the hare laps the tortoise and they meet inside the loop. If the hare reaches null, there is no cycle. To find the loop entry: reset one pointer to the head and advance both one step at a time — their next meeting point is the cycle start.

**Template:**
```python
def has_cycle(head):
    slow = fast = head
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
        if slow is fast:
            return True
    return False

# Array variant — Find the Duplicate (LC 287)
def find_duplicate(nums):
    slow = fast = nums[0]
    while True:
        slow = nums[slow]
        fast = nums[nums[fast]]
        if slow == fast:
            break
    slow = nums[0]
    while slow != fast:
        slow = nums[slow]
        fast = nums[fast]
    return slow
```

**Complexity:** Time O(n) · Space O(1)

**Representative problems:**
- **[Linked List Cycle (LC 141)](https://leetcode.com/problems/linked-list-cycle/)** — Return `True` if slow and fast ever meet.
- **[Find the Duplicate Number (LC 287)](https://leetcode.com/problems/find-the-duplicate-number/)** — Treat the array as a linked list (`nums[i]` → index `nums[i]`); Floyd's finds the duplicate in O(n)/O(1).
- **[Happy Number (LC 202)](https://leetcode.com/problems/happy-number/)** — Digit-square function is the "next" pointer; cycle detection replaces the hash-set approach for O(1) space.

**Common pitfalls:**
- Forgetting the `fast.next` null check — `fast.next.next` raises `AttributeError` if `fast.next` is already `None`.
- For LC 287, the array must have values in `[1, n]` for the implicit-list interpretation to work.
- Confusing cycle detection (do they meet?) with cycle-start finding (where does the loop begin?) — the latter requires a second phase with one pointer reset to head.

---

## Search, Recursion & Math Patterns

> These seven patterns appear across [NeetCode 150](https://neetcode.io/practice/practice/neetcode150), [Blind 75](https://neetcode.io/practice/practice/blind75), and [Grokking the Coding Interview](https://www.educative.io/courses/grokking-coding-interview) and cover most non-DP algorithmic problem types.

### Binary Search (Classic on Sorted Array)

**When to reach for it:** The array (or search space) is sorted, you need O(log n) lookup, or the problem says "find target / find index." Trigger phrases: *"sorted array"*, *"O(log n) required"*, *"find first/last occurrence"*.

**In plain English:** Think of a phone book. You open to the middle page; if your name comes before the middle entry, tear off the right half and repeat — you halve the problem every step. The key discipline is maintaining a half-open interval `[lo, hi)` so the loop termination and index arithmetic are always consistent.

**Template:**
```python
def binary_search(nums: list[int], target: int) -> int:
    lo, hi = 0, len(nums)          # half-open: lo inclusive, hi exclusive
    while lo < hi:
        mid = lo + (hi - lo) // 2  # overflow-safe
        if nums[mid] == target:
            return mid
        elif nums[mid] < target:
            lo = mid + 1
        else:
            hi = mid
    return -1                       # not found
```

**Complexity:** Time O(log n) · Space O(1).

**Representative problems:**
- **[Binary Search (LC 704)](https://leetcode.com/problems/binary-search/)** — textbook half-open interval on a sorted array; the template problem.
- **[Search in Rotated Sorted Array (LC 33)](https://leetcode.com/problems/search-in-rotated-sorted-array/)** — identify which half is still sorted, then binary-search within the sorted half.
- **[Find Minimum in Rotated Sorted Array (LC 153)](https://leetcode.com/problems/find-minimum-in-rotated-sorted-array/)** — compare `mid` to `hi-1` to decide which side the inflection point is on.

**Common pitfalls:**
- Using `(lo + hi) // 2` risks integer overflow in fixed-width languages; always write `lo + (hi - lo) // 2`.
- Off-by-one on the termination condition: `while lo < hi` with a half-open interval vs. `while lo <= hi` with a closed interval — mixing the two models causes infinite loops or missed elements.
- Forgetting to handle the empty-array edge case (`len(nums) == 0`).
- Moving `lo = mid` instead of `lo = mid + 1` when using a half-open interval, creating an infinite loop when `hi == lo + 1`.

### Binary Search on the Answer (Search Space / "Minimize the Maximum")

**When to reach for it:** You are asked to *minimize* or *maximize* some value `k`, and you can write a `feasible(k)` predicate that is monotonic (once true, always true for larger/smaller `k`). Trigger phrases: *"minimum speed/capacity/days"*, *"maximize the minimum"*, *"can we do this in X?"*, *"answer is monotonic in a parameter"*.

**In plain English:** Instead of searching an array, you binary-search over the *answer space* itself. Imagine guessing the minimum hourly eating speed for a monkey — too slow and she can't finish; too fast and you haven't minimized. Because "feasible" flips from False to True exactly once as speed increases, binary search finds the tipping point in O(log(answer_range)) feasibility checks.

**Template:**
```python
def binary_search_on_answer(lo: int, hi: int) -> int:
    """feasible(mid) returns True if mid satisfies the constraint."""
    result = hi
    while lo <= hi:
        mid = lo + (hi - lo) // 2
        if feasible(mid):
            result = mid   # mid works; try to do better (smaller)
            hi = mid - 1
        else:
            lo = mid + 1
    return result

# Example feasibility check for "minimum speed" style:
import math
def feasible(speed, piles, h):
    return sum(math.ceil(p / speed) for p in piles) <= h
```

**Complexity:** Time O(log(answer_range) × cost_of_feasible) · Space O(1).

**Representative problems:**
- **[Koko Eating Bananas (LC 875)](https://leetcode.com/problems/koko-eating-bananas/)** — binary search over eating speed 1…max(piles); feasibility checks total hours consumed.
- **[Capacity to Ship Packages Within D Days (LC 1011)](https://leetcode.com/problems/capacity-to-ship-packages-within-d-days/)** — binary search over ship capacity; feasibility simulates a greedy packing pass.
- **[Split Array Largest Sum (LC 410)](https://leetcode.com/problems/split-array-largest-sum/)** — "minimize the maximum subarray sum"; binary-search the answer, feasibility greedily counts splits.

**Common pitfalls:**
- Setting `lo` or `hi` incorrectly — `lo` must be the *smallest valid candidate* (often 1 or `max(array)`), not 0.
- Forgetting that the feasibility function must be strictly monotone over the answer space; if it isn't, binary search won't converge.
- Off-by-one: when minimizing, save `result = mid` *before* narrowing `hi = mid - 1`.

### Backtracking (Subsets / Permutations / Combinations / Constraint Search)

**When to reach for it:** You need to enumerate *all* valid configurations, or find any one configuration satisfying hard constraints. Trigger phrases: *"find all subsets/permutations/combinations"*, *"generate all valid …"*, *"return all possible …"*, *"place N queens"*, *"solve Sudoku"*.

**In plain English:** Backtracking is a controlled brute-force: you build a candidate solution one piece at a time, and the moment you detect a constraint violation you *undo the last choice* (backtrack) and try the next option. Picture a maze — you walk until you hit a wall, then retrace to the last junction and take a different turn. The key is the "undo" step that keeps the shared state clean.

**Template:**
```python
def backtrack(start, path, result, candidates):
    result.append(path[:])          # snapshot, not a reference

    for i in range(start, len(candidates)):
        # Optional pruning: skip duplicates, check constraints early
        if i > start and candidates[i] == candidates[i - 1]:
            continue                # skip duplicate branches

        path.append(candidates[i])
        backtrack(i + 1, path, result, candidates)  # i+1 = no reuse; i = allow reuse
        path.pop()                  # undo choice

def solve(nums):
    result = []
    nums.sort()                     # sort enables duplicate-skip pruning
    backtrack(0, [], result, nums)
    return result
```

**Complexity:** Time O(2ⁿ × n) for subsets, O(n! × n) for permutations · Space O(n) call stack + output.

**Representative problems:**
- **[Subsets (LC 78)](https://leetcode.com/problems/subsets/)** — at each index, choose include or exclude; the classic binary decision tree.
- **[Permutations (LC 46)](https://leetcode.com/problems/permutations/)** — at each step, pick any unused element; track usage with a `visited` set or by swapping in-place.
- **[Combination Sum (LC 39)](https://leetcode.com/problems/combination-sum/)** — allow reuse of elements (`backtrack(i, …)` instead of `i+1`); prune when running sum exceeds target.

**Common pitfalls:**
- Appending `path` directly instead of `path[:]` — all recorded results will mutate to the final empty list.
- Forgetting to prune early (checking the constraint *before* recursing), which blows up runtime.
- Missing the sort + duplicate-skip step for problems like Subsets II (LC 90) or Combination Sum II (LC 40).
- Off-by-one in the `start` index: use `i` (not `i+1`) to allow element reuse, `i+1` to forbid it.

### Recursion & Divide-and-Conquer Fundamentals

**When to reach for it:** The problem decomposes into independent subproblems of the same shape, or you can split input in half and combine results. Trigger phrases: *"merge sort"*, *"find the k-th largest"*, *"expression evaluation"*, *"count inversions"*, *"split and combine"*.

**In plain English:** Divide-and-conquer is recursion with structure: split the problem into roughly equal halves, solve each half independently, then merge the partial answers. The classic analogy is sorting a deck of cards by splitting it into two piles, sorting each, then merging — that's merge sort. The master theorem tells you T(n) = 2T(n/2) + O(n) solves to O(n log n).

**Template:**
```python
def merge_sort(arr):
    if len(arr) <= 1:
        return arr
    mid = len(arr) // 2
    left = merge_sort(arr[:mid])
    right = merge_sort(arr[mid:])
    return merge(left, right)

def merge(left, right):
    result, i, j = [], 0, 0
    while i < len(left) and j < len(right):
        if left[i] <= right[j]:
            result.append(left[i]); i += 1
        else:
            result.append(right[j]); j += 1
    return result + left[i:] + right[j:]
```

**Complexity:** Time O(n log n) for merge-style problems · Space O(log n) call stack + O(n) merge buffer.

**Representative problems:**
- **[Sort List (LC 148)](https://leetcode.com/problems/sort-list/)** — merge sort on a linked list; use slow/fast pointers to find the midpoint, then merge recursively.
- **[Merge k Sorted Lists (LC 23)](https://leetcode.com/problems/merge-k-sorted-lists/)** — divide-and-conquer over the k lists with pairwise merging, O(N log k).
- **[Different Ways to Add Parentheses (LC 241)](https://leetcode.com/problems/different-ways-to-add-parentheses/)** — split expression at each operator, recurse on both sides, combine all result pairs.

**Common pitfalls:**
- Infinite recursion from a missing or wrong base case (ensure `len <= 1` before splitting).
- Finding the midpoint of a linked list requires the slow/fast trick, not index arithmetic.
- Merge step that doesn't handle unequal-length halves (always drain the remainder with `+ left[i:]`).
- Accidentally sharing mutable state between the left and right recursive calls.

### Bit Manipulation (XOR Tricks, Masks, Counting Bits)

**When to reach for it:** O(1) space is required alongside pairing/cancellation logic, you're asked about *unique* vs. *duplicate* elements, or the problem involves power-of-2 checks or bit counting. Trigger phrases: *"find the single number"*, *"missing number"*, *"count set bits"*, *"only O(1) extra space"*, *"swap without temp"*.

**In plain English:** XOR (`^`) is a toggling operation — XOR-ing a number with itself gives 0, and XOR-ing anything with 0 leaves it unchanged. If you XOR all numbers in a list where every element appears twice except one, the duplicates cancel and the lone survivor is your answer. AND with `n-1` strips the lowest set bit (useful for counting bits).

**Template:**
```python
def single_number(nums):
    result = 0
    for n in nums:
        result ^= n          # duplicates cancel: x ^ x == 0
    return result

def count_bits(n):           # Brian Kernighan
    count = 0
    while n:
        n &= n - 1           # strips the lowest set bit
        count += 1
    return count

is_power_of_two = lambda n: n > 0 and (n & (n - 1)) == 0
lowest_set_bit  = lambda n: n & (-n)
```

**Complexity:** Time O(n) for linear scans, O(1) for single-number tricks · Space O(1).

**Representative problems:**
- **[Single Number (LC 136)](https://leetcode.com/problems/single-number/)** — XOR all elements; pairs cancel, leaving the unique value.
- **[Number of 1 Bits (LC 191)](https://leetcode.com/problems/number-of-1-bits/)** — apply `n &= n-1` in a loop to strip set bits one by one.
- **[Missing Number (LC 268)](https://leetcode.com/problems/missing-number/)** — XOR all indices `0…n` with all array values; missing index is the survivor.

**Common pitfalls:**
- Python integers are unlimited-precision — `>>` on negatives sign-extends forever; mask with `& 0xFFFFFFFF` when simulating 32-bit behavior.
- `~n` (bitwise NOT) gives `-(n+1)` in Python; use `n ^ ((1 << 32) - 1)` for a 32-bit NOT.
- Applying XOR tricks when elements appear an *odd* number of times greater than 1 (XOR only cancels pairs).
- Forgetting that `n & (n-1)` needs `n != 0`; always guard with `while n:`.

### Greedy (with Justification of the Greedy Choice)

**When to reach for it:** An optimal global solution can be built by making the locally optimal choice at each step, and that choice never needs to be reconsidered. Trigger phrases: *"minimum number of …"*, *"maximum coverage"*, *"schedule tasks"*, *"interval overlap"*, *"jump game"*. Test: can you prove an *exchange argument* — that swapping any non-greedy choice for the greedy one never worsens the solution?

**In plain English:** Greedy is the "always take the best-looking option right now" strategy. The classic justification is the **exchange argument**: suppose an optimal solution makes a different choice at some step — show that swapping it for the greedy choice produces a solution at least as good, so the greedy solution is also optimal. Without this proof, greedy may be wrong (knapsack, coin change with arbitrary denominations).

**Template:**
```python
# Interval scheduling (keep interval with earliest end time)
def min_removals_to_non_overlap(intervals):
    intervals.sort(key=lambda x: x[1])   # sort by end time — greedy choice
    removals, prev_end = 0, float('-inf')
    for start, end in intervals:
        if start < prev_end:              # overlaps with last kept interval
            removals += 1                 # remove current (ends later — worse)
        else:
            prev_end = end               # keep current; update boundary
    return removals

# Reach / jump
def can_jump(nums):
    max_reach = 0
    for i, jump in enumerate(nums):
        if i > max_reach:
            return False                  # current index unreachable
        max_reach = max(max_reach, i + jump)
    return True
```

**Complexity:** Typically O(n log n) due to sorting, O(n) traversal · Space O(1).

**Representative problems:**
- **[Jump Game (LC 55)](https://leetcode.com/problems/jump-game/)** — track `max_reach` greedily; if the current index ever exceeds it, return False.
- **[Jump Game II (LC 45)](https://leetcode.com/problems/jump-game-ii/)** — greedily extend the "current window" boundary; increment jumps when you exhaust the window.
- **[Non-overlapping Intervals (LC 435)](https://leetcode.com/problems/non-overlapping-intervals/)** — sort by end time; keep whichever interval ends earliest.

**Common pitfalls:**
- Applying greedy to a problem that actually requires DP (0/1 knapsack, coin change with arbitrary denominations).
- Sorting by the wrong key — for interval problems, sorting by *end time* is the classic greedy invariant.
- Failing to account for ties in the sort key, which can break the exchange-argument proof.

### Math / Number-Theory Quick Hits (GCD, Modular Arithmetic, Fast Power)

**When to reach for it:** The problem involves large numbers, factorization, remainders, fast exponentiation, or the answer must be returned "mod 10⁹+7". Trigger phrases: *"return answer modulo …"*, *"trailing zeroes"*, *"power of x"*, *"GCD / LCM"*.

**In plain English:** A small toolkit of number-theory identities eliminates the need for big-integer arithmetic in most interview problems. **GCD via Euclid** runs in O(log min(a,b)). **Modular arithmetic** keeps numbers small: `(a × b) % m == ((a % m) × (b % m)) % m`. **Fast exponentiation** computes `xⁿ` in O(log n) by squaring.

**Template:**
```python
def gcd(a, b):
    while b:
        a, b = b, a % b
    return a

lcm = lambda a, b: a * b // gcd(a, b)

def power_mod(base, exp, mod):      # fast exponentiation
    result = 1
    base %= mod
    while exp > 0:
        if exp & 1:                 # odd exponent: multiply in current base
            result = result * base % mod
        base = base * base % mod    # square
        exp >>= 1
    return result
```

**Complexity:** GCD O(log min(a,b)) · Fast exponentiation O(log n) · Space O(1).

**Representative problems:**
- **[Pow(x, n) (LC 50)](https://leetcode.com/problems/powx-n/)** — fast exponentiation by squaring; handle negative `n` with `1/power(x, -n)`.
- **[Multiply Strings (LC 43)](https://leetcode.com/problems/multiply-strings/)** — simulate grade-school multiplication using `result[i+j+1] += d1 * d2`; no `int()` conversion allowed.
- **[Missing Number (LC 268)](https://leetcode.com/problems/missing-number/)** — Gauss formula `n*(n+1)//2 - sum(nums)` gives the answer in O(n) time, O(1) space.

**Common pitfalls:**
- Forgetting `base %= mod` *before* the fast-exponentiation loop — the first squaring can overflow in fixed-width languages.
- Using `//` for division inside a modular context when you need a modular inverse.
- For GCD/LCM, passing `0` without a guard — `gcd(0, 0)` is undefined.

---

## Linked List, Stack, Tree & Heap Patterns

> **Pattern taxonomy sources:** [NeetCode 150](https://neetcode.io/practice?tab=neetcode150) · [Grokking the Coding Interview (educative.io)](https://www.educative.io/courses/grokking-coding-interview) · [Design Gurus — 20 Patterns](https://www.designgurus.io/blog/grokking-the-coding-interview-patterns)

```python
# Shared node shapes — defined once, used throughout
class ListNode:
    def __init__(self, val=0, next=None):
        self.val = val
        self.next = next

class TreeNode:
    def __init__(self, val=0, left=None, right=None):
        self.val = val
        self.left = left
        self.right = right
```

### Linked List Manipulation (Dummy Node, Reversal, Fast/Slow Pointers)

**When to reach for it:**
- "Reverse a linked list" or "reverse a sublist"
- "Detect a cycle," "find the start of a cycle," or "find the middle node"
- "Merge two sorted lists," "reorder list," or any in-place pointer surgery
- Problems that say you may not use extra space and involve a singly linked list

**In plain English:**
A linked list has no random access, so every trick boils down to carefully choreographing a handful of pointers. A **dummy (sentinel) head** eliminates the special-case of operating on the head node itself. **Reversal** threads `prev → cur → next` backwards one node at a time. The **fast/slow pointer** exploits the fact that two runners at different speeds will either meet inside a cycle or let the slow pointer land on the middle node when fast reaches the tail.

**Template:**
```python
# Iterative reversal
def reverse_list(head):
    prev, cur = None, head
    while cur:
        nxt = cur.next   # save before overwriting
        cur.next = prev
        prev = cur
        cur = nxt
    return prev

# Dummy head (merge / delete operations)
def merge_two_lists(l1, l2):
    dummy = cur = ListNode(0)
    while l1 and l2:
        if l1.val <= l2.val:
            cur.next, l1 = l1, l1.next
        else:
            cur.next, l2 = l2, l2.next
        cur = cur.next
    cur.next = l1 or l2
    return dummy.next

# Fast / slow: middle node
def find_middle(head):
    slow = fast = head
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
    return slow  # for even-length lists, returns second middle
```

**Complexity:** O(n) time, O(1) space for all of the above.

**Representative problems:**
- **[Reverse Linked List (LC 206)](https://leetcode.com/problems/reverse-linked-list/)** — iteratively thread `prev→cur` backwards; base case for almost every other linked-list problem.
- **[Merge Two Sorted Lists (LC 21)](https://leetcode.com/problems/merge-two-sorted-lists/)** — dummy head lets you always write `cur.next = smaller_node` without branching on the head.
- **[Reorder List (LC 143)](https://leetcode.com/problems/reorder-list/)** — find middle with slow/fast, reverse the second half in-place, then interleave both halves.

**Common pitfalls:**
- Saving `nxt = cur.next` **before** overwriting `cur.next`; forgetting this loses the rest of the list.
- Not using a dummy head when the operation may change `head` itself (delete-node, insert-at-front).
- Off-by-one on the fast/slow split: decide upfront whether you want the **first** or **second** middle.
- Forgetting to null-terminate the first half after splitting (leaves a cycle in the list).

### Stack & Monotonic Stack

**When to reach for it:**
- "Next greater / next smaller element," "previous greater / previous smaller element"
- "Stock span," "daily temperatures," "largest rectangle in histogram"
- Matching parentheses, valid brackets, or nested structure problems
- Any phrase involving "the first X to the left/right that is larger/smaller"

**In plain English:**
A plain stack gives you LIFO access — use it to match open-close pairs or to undo operations. A **monotonic stack** is maintained in strictly increasing or decreasing order. When you push a new element, you first pop everything that violates the monotonicity — those popped elements have just found their "next greater/smaller" answer in the current element. Think of people in a queue; a tall person blocks everyone shorter behind them from seeing forward.

**Template:**
```python
# Next Greater Element (monotonic decreasing stack of indices)
def next_greater(nums):
    n = len(nums)
    result = [-1] * n
    stack = []                       # values are decreasing
    for i, val in enumerate(nums):
        while stack and nums[stack[-1]] < val:
            idx = stack.pop()
            result[idx] = val
        stack.append(i)
    return result

# Valid Parentheses (plain stack)
def is_valid(s):
    stack = []
    pairs = {')': '(', ']': '[', '}': '{'}
    for ch in s:
        if ch in pairs:
            if not stack or stack[-1] != pairs[ch]:
                return False
            stack.pop()
        else:
            stack.append(ch)
    return not stack
```

**Complexity:** O(n) time (each element pushed/popped at most once), O(n) space.

**Representative problems:**
- **[Daily Temperatures (LC 739)](https://leetcode.com/problems/daily-temperatures/)** — monotonic decreasing stack of indices; when a warmer day arrives, pop and record the gap.
- **[Next Greater Element I (LC 496)](https://leetcode.com/problems/next-greater-element-i/)** — build a hash map of `value → next_greater` for `nums2` using a monotonic stack, then look up each element of `nums1`.
- **[Online Stock Span (LC 901)](https://leetcode.com/problems/online-stock-span/)** — maintain a stack of `(price, span)` pairs; accumulate spans of popped (lower) prices into the current span.

**Common pitfalls:**
- Confusing **increasing** vs **decreasing** stack: decreasing stack for "next greater," increasing stack for "next smaller."
- Storing **indices** rather than values is almost always necessary (you need position).
- Forgetting elements left on the stack at the end — they have no answer (return `-1` or `0`).
- For circular arrays (LC 503), iterate `2n` and use `i % n` to simulate wrap-around.

### Queue / BFS on Grids & Levels

**When to reach for it:**
- "Shortest path," "minimum steps," or "fewest moves" in an unweighted graph or grid
- "Level by level," "level order," "process layer by layer"
- "Spread simultaneously" (rotting oranges, walls and gates)
- "Minimum depth," "closest leaf," or any problem where you need the **nearest** answer

**In plain English:**
BFS explores neighbors in waves — everything one step away before anything two steps away. Use a `deque` as your queue and record the **queue size at the start of each wave** so you can process a full level before moving to the next. For grids, encode `(row, col)` pairs and keep a `visited` set (or mark the cell in-place) to avoid re-processing. Think of ripples expanding outward on a pond.

**Template:**
```python
from collections import deque

# BFS level-order on a binary tree
def level_order(root):
    if not root:
        return []
    result, queue = [], deque([root])
    while queue:
        level_size = len(queue)          # snapshot current level
        level = []
        for _ in range(level_size):
            node = queue.popleft()
            level.append(node.val)
            if node.left:  queue.append(node.left)
            if node.right: queue.append(node.right)
        result.append(level)
    return result

# BFS on a grid (shortest path / flood fill)
def bfs_grid(grid, start):
    rows, cols = len(grid), len(grid[0])
    queue = deque([(*start, 0)])         # (row, col, distance)
    visited = {start}
    while queue:
        r, c, dist = queue.popleft()
        for dr, dc in ((0, 1), (0, -1), (1, 0), (-1, 0)):
            nr, nc = r + dr, c + dc
            if (0 <= nr < rows and 0 <= nc < cols
                    and (nr, nc) not in visited and grid[nr][nc] != 0):
                visited.add((nr, nc))
                queue.append((nr, nc, dist + 1))
    return -1                            # target unreachable
```

**Complexity:** O(V + E) for graphs; O(rows × cols) for grids. Space O(V).

**Representative problems:**
- **[Binary Tree Level Order Traversal (LC 102)](https://leetcode.com/problems/binary-tree-level-order-traversal/)** — snapshot `len(queue)` at the start of each iteration to bucket nodes by level.
- **[Number of Islands (LC 200)](https://leetcode.com/problems/number-of-islands/)** — BFS/DFS from each unvisited `'1'`, flood-fill the island; count how many times you start a new flood.
- **[Rotting Oranges (LC 994)](https://leetcode.com/problems/rotting-oranges/)** — multi-source BFS seeded with all initially rotten oranges simultaneously; the answer is the number of levels (minutes) processed.

**Common pitfalls:**
- Adding a node to the queue **without** immediately marking it visited — leads to exponential re-visits.
- Forgetting to check `if root` before seeding the queue for tree BFS.
- Using a list as a queue (`list.pop(0)` is O(n)) instead of `collections.deque` (`popleft()` is O(1)).
- For multi-source BFS (LC 994), seed **all** sources into the queue before starting, not one at a time.

### Binary Tree Traversals (DFS Pre/In/Post + BFS Level-Order)

**When to reach for it:**
- "Inorder traversal" → BST in sorted order; serialize/deserialize
- "Preorder" → copy/clone a tree, serialize, path problems (root first)
- "Postorder" → delete a tree, compute subtree properties bottom-up (children before parent)
- Any problem requiring aggregation from children back to parent → postorder DFS

**In plain English:**
DFS on a binary tree is recursion following one of three orderings. **Pre-order** (root → left → right) is top-down — use it when you need parent context before children. **In-order** (left → root → right) visits a BST in sorted order. **Post-order** (left → right → root) is bottom-up — use it when a node needs information from both subtrees before it can compute its own answer (height, diameter, path sums).

**Template:**
```python
# Recursive DFS (move the append line for pre/in/post)
def inorder(node, result):
    if not node:
        return
    inorder(node.left, result)
    result.append(node.val)        # in-order position
    inorder(node.right, result)

# Post-order: height (classic bottom-up)
def max_depth(root):
    if not root:
        return 0
    return 1 + max(max_depth(root.left), max_depth(root.right))

# Post-order: diameter (global max updated inside recursion)
def diameter_of_binary_tree(root):
    best = [0]
    def height(node):
        if not node: return 0
        L, R = height(node.left), height(node.right)
        best[0] = max(best[0], L + R)   # path through this node
        return 1 + max(L, R)
    height(root)
    return best[0]
```

**Complexity:** All traversals O(n) time, O(h) space (h = height; O(log n) balanced, O(n) skewed).

**Representative problems:**
- **[Binary Tree Inorder Traversal (LC 94)](https://leetcode.com/problems/binary-tree-inorder-traversal/)** — foundational; implement both recursive and iterative versions.
- **[Maximum Depth of Binary Tree (LC 104)](https://leetcode.com/problems/maximum-depth-of-binary-tree/)** — single-line post-order recursion `1 + max(left, right)`.
- **[Diameter of Binary Tree (LC 543)](https://leetcode.com/problems/diameter-of-binary-tree/)** — post-order DFS tracking `left_height + right_height` as the candidate diameter; update a global best.

**Common pitfalls:**
- On a skewed tree, recursion depth hits Python's default limit (~1000); consider iterative or `sys.setrecursionlimit`.
- Returning height vs. diameter from the same helper — use a mutable/nonlocal variable for the global answer and return only height.
- Off-by-one in diameter: it's the number of **edges**, so `left_height + right_height`, not `+1`.
- Confusing "depth" (distance from root) with "height" (distance to farthest leaf).

### Binary Search Tree (BST) Properties

**When to reach for it:**
- "Validate BST," "kth smallest in BST," "lowest common ancestor in BST"
- "Convert sorted array/list to BST," "BST iterator," "range sum of BST"
- Any problem where sorted order of tree values is exploited (in-order gives sorted sequence)

**In plain English:**
A BST's defining invariant is that **every node's left subtree contains only smaller values and the right subtree contains only larger values**. This means in-order DFS visits nodes in sorted ascending order — a fact that unlocks O(h) search, insert, and delete. For validation, carry an open interval `(min_val, max_val)` downward; every node must fall strictly inside it.

**Template:**
```python
# Validate BST (carry bounds downward)
def is_valid_bst(root, lo=float('-inf'), hi=float('inf')):
    if not root:
        return True
    if not (lo < root.val < hi):
        return False
    return (is_valid_bst(root.left,  lo, root.val) and
            is_valid_bst(root.right, root.val, hi))

# LCA in BST (exploit ordering)
def lca_bst(root, p, q):
    while root:
        if p.val < root.val and q.val < root.val:
            root = root.left
        elif p.val > root.val and q.val > root.val:
            root = root.right
        else:
            return root   # split point = LCA
```

**Complexity:** Search/insert/delete/LCA O(h). Validate/kth-smallest O(n) time, O(h) space.

**Representative problems:**
- **[Validate Binary Search Tree (LC 98)](https://leetcode.com/problems/validate-binary-search-tree/)** — pass `(min, max)` bounds down; reject any node outside its valid range.
- **[Kth Smallest Element in a BST (LC 230)](https://leetcode.com/problems/kth-smallest-element-in-a-bst/)** — iterative in-order traversal; decrement a counter and return when it hits zero.
- **[Lowest Common Ancestor of a BST (LC 235)](https://leetcode.com/problems/lowest-common-ancestor-of-a-binary-search-tree/)** — navigate left/right until p and q split to opposite sides — that node is the LCA.

**Common pitfalls:**
- Checking only `node.val > node.left.val` (local comparison) instead of propagating true bounds — fails for inputs like `[5,4,6,null,null,3,7]`.
- Using `<=` vs `<`: BST usually requires **strict** inequality.
- Confusing BST LCA (LC 235 — O(h), no extra space) with general binary tree LCA (LC 236 — full post-order DFS).

### Trie (Prefix Tree)

**When to reach for it:**
- "Autocomplete," "type-ahead search," "word suggestions"
- "Find all words with prefix X" or "does any word start with X?"
- "Word search on a board" where you need to prune impossible prefixes early

**In plain English:**
A Trie is a tree where each **edge** represents a character, so every path from root to a marked node spells a stored word. Because all words sharing a prefix converge on the same path, prefix lookups are O(L) where L is the prefix length — independent of how many words are stored. Think of it as a multi-way branching decision tree for spelling.

**Template:**
```python
class TrieNode:
    def __init__(self):
        self.children = {}
        self.is_end = False

class Trie:
    def __init__(self):
        self.root = TrieNode()

    def insert(self, word):
        node = self.root
        for ch in word:
            node = node.children.setdefault(ch, TrieNode())
        node.is_end = True

    def _find(self, prefix):
        node = self.root
        for ch in prefix:
            if ch not in node.children:
                return None
            node = node.children[ch]
        return node

    def search(self, word):
        node = self._find(word)
        return node is not None and node.is_end

    def starts_with(self, prefix):
        return self._find(prefix) is not None
```

**Complexity:** Insert/search/prefix O(L) time. Space O(total characters inserted).

**Representative problems:**
- **[Implement Trie / Prefix Tree (LC 208)](https://leetcode.com/problems/implement-trie-prefix-tree/)** — straightforward implementation; know it cold before harder variants.
- **[Design Add and Search Words Data Structure (LC 211)](https://leetcode.com/problems/design-add-and-search-words-data-structure/)** — extend search to handle `.` wildcards via DFS over all children at that level.
- **[Word Search II (LC 212)](https://leetcode.com/problems/word-search-ii/)** — insert all target words into a Trie, then DFS the board; prune as soon as the current path diverges from any Trie prefix.

**Common pitfalls:**
- Forgetting to set `is_end = True` after inserting — `search("apple")` returns `False` even after inserting "apple".
- Using a fixed-size array `[None]*26` limits you to lowercase ASCII; a dict is more general.
- In Word Search II, forgetting to prune already-found words (duplicate results / slow).
- Confusing `search` (full word must exist) with `starts_with` (only prefix).

### Heap / Priority Queue & Top-K (Including Two-Heaps for Median)

**When to reach for it:**
- "Kth largest," "kth smallest," "top K elements," "K most frequent"
- "Merge K sorted lists/arrays," "smallest range covering K lists"
- "Continuously arriving data — return median at any point"
- Any problem where you repeatedly need the current minimum or maximum

**In plain English:**
Python's `heapq` is a **min-heap** — the smallest element is always at the top. To simulate a max-heap, negate values before pushing and negate again when popping. For **Top-K largest**, maintain a min-heap of size K; if the heap grows beyond K, pop the smallest — what remains is the K largest. For the **running median**, split the stream into a max-heap (lower half) and a min-heap (upper half), keeping their sizes balanced within one.

**Template:**
```python
import heapq

# Top K Largest (min-heap of size K)
def top_k_largest(nums, k):
    heap = []
    for n in nums:
        heapq.heappush(heap, n)
        if len(heap) > k:
            heapq.heappop(heap)   # evict the current smallest
    return heap                    # remaining K are the largest

# Two-heaps: running median
class MedianFinder:
    def __init__(self):
        self.lo = []   # max-heap (negated) — lower half
        self.hi = []   # min-heap — upper half

    def addNum(self, num):
        heapq.heappush(self.lo, -num)
        if self.hi and (-self.lo[0]) > self.hi[0]:
            heapq.heappush(self.hi, -heapq.heappop(self.lo))
        if len(self.lo) > len(self.hi) + 1:
            heapq.heappush(self.hi, -heapq.heappop(self.lo))
        elif len(self.hi) > len(self.lo):
            heapq.heappush(self.lo, -heapq.heappop(self.hi))

    def findMedian(self):
        if len(self.lo) > len(self.hi):
            return -self.lo[0]
        return (-self.lo[0] + self.hi[0]) / 2.0
```

**Complexity:** Push/pop O(log n). Top-K O(n log k). Two-heaps addNum O(log n), findMedian O(1).

**Representative problems:**
- **[Top K Frequent Elements (LC 347)](https://leetcode.com/problems/top-k-frequent-elements/)** — frequency count then min-heap of size K on `(freq, element)`; or bucket sort for O(n).
- **[Kth Largest Element in an Array (LC 215)](https://leetcode.com/problems/kth-largest-element-in-an-array/)** — min-heap of size K; top of heap after full traversal is the answer (QuickSelect also O(n) average).
- **[Find Median from Data Stream (LC 295)](https://leetcode.com/problems/find-median-from-data-stream/)** — two-heap invariant: `lo` (max-heap) holds the lower half, `hi` (min-heap) the upper half, sizes differ by ≤ 1.
- **[Merge K Sorted Lists (LC 23)](https://leetcode.com/problems/merge-k-sorted-lists/)** — min-heap keyed on node value drives greedy selection of the next smallest node.

**Common pitfalls:**
- `heapq` is **always a min-heap**; forgetting to negate gives wrong results for max-heap problems.
- When heap elements are tuples `(priority, item)` and items are non-comparable (e.g., `ListNode`), include a tiebreaker index `(val, idx, node)` to avoid `TypeError`.
- For Top-K **smallest**, use a **max-heap** of size K and evict the largest — the opposite of Top-K largest.
- In the two-heaps median, forgetting to rebalance after the cross-check leads to size drift and wrong results.

---

## Graph & Dynamic Programming Patterns

### Graph Representation & DFS/BFS Traversal

**When to reach for it:** "Explore all reachable nodes," "find connected components," "count islands / regions," "shortest path in an unweighted grid," "detect a cycle," "flood fill." Any problem where you need to *visit every node or cell* and track what you've seen.

**In plain English:** Think of a city map. DFS follows one road as far as it goes before backtracking (stack, implicit via recursion). BFS sends ripples outward — every neighbor one step away before anything two steps away (queue, guarantees shortest path in unweighted graphs). Graphs are stored as an **adjacency list** (dict of lists — sparse, common in interviews) or a **2-D grid** (a cell's four neighbors are its edges).

**Template:**
```python
from collections import deque

def dfs(graph, start):
    visited = set()
    def _dfs(node):
        if node in visited:
            return
        visited.add(node)
        for neighbor in graph[node]:
            _dfs(neighbor)
    _dfs(start)

def bfs(graph, start):
    visited = {start}
    queue = deque([start])
    while queue:
        node = queue.popleft()
        for neighbor in graph[node]:
            if neighbor not in visited:
                visited.add(neighbor)
                queue.append(neighbor)
```

**Complexity:** Time O(V + E), Space O(V).

**Representative problems:**
- **[Number of Islands (LC 200)](https://leetcode.com/problems/number-of-islands/)** — launch a DFS from every unvisited `'1'` cell, sinking the island; count launches.
- **[Pacific Atlantic Water Flow (LC 417)](https://leetcode.com/problems/pacific-atlantic-water-flow/)** — reverse-BFS inward from each ocean's border; answer is the intersection of reachable cells.
- **[Clone Graph (LC 133)](https://leetcode.com/problems/clone-graph/)** — DFS/BFS with a `{original: copy}` hash map to avoid infinite loops on cycles.

**Common pitfalls:**
- Forgetting to mark a node visited *before* pushing it onto the queue (causes duplicates and TLE).
- In grid problems, bounds-checking after indexing instead of before.
- Using DFS for "shortest path" in unweighted graphs — always use BFS for that.
- Infinite recursion on cyclic graphs when the visited set is missing.

### Topological Sort (Kahn's / DFS) for Dependencies

**When to reach for it:** "Prerequisites / dependencies / ordering," "can all tasks be completed?", "find a valid build/compile order," "detect a cycle in a directed graph." Any problem with a directed relationship where some things must come before others.

**In plain English:** Imagine a course catalog where Course B requires Course A. Topological sort produces a linear ordering where every prerequisite appears before what depends on it. Kahn's algorithm is BFS-based: repeatedly take nodes with no remaining prerequisites (in-degree 0) and strip their edges. If you can't process every node, there's a cycle.

**Template:**
```python
from collections import deque, defaultdict

def topo_sort_kahn(num_nodes, edges):
    graph = defaultdict(list)
    in_degree = [0] * num_nodes
    for u, v in edges:          # u must come before v
        graph[u].append(v)
        in_degree[v] += 1

    queue = deque(i for i in range(num_nodes) if in_degree[i] == 0)
    order = []
    while queue:
        node = queue.popleft()
        order.append(node)
        for neighbor in graph[node]:
            in_degree[neighbor] -= 1
            if in_degree[neighbor] == 0:
                queue.append(neighbor)

    return order if len(order) == num_nodes else []  # [] means a cycle exists
```

**Complexity:** Time O(V + E), Space O(V + E).

**Representative problems:**
- **[Course Schedule (LC 207)](https://leetcode.com/problems/course-schedule/)** — build a prereq graph; if Kahn's produces all n courses, return `True` (no cycle).
- **[Course Schedule II (LC 210)](https://leetcode.com/problems/course-schedule-ii/)** — same graph, but return the actual topological order.
- **[Alien Dictionary (LC 269)](https://leetcode.com/problems/alien-dictionary/)** — derive character-ordering edges from adjacent words in the sorted list, then topo-sort.

**Common pitfalls:**
- Confusing edge direction (u → v means u is a prerequisite of v).
- Not checking `len(order) == num_nodes` to detect cycles.
- Forgetting isolated nodes (zero in-degree from the start) — they must seed the queue.

### Union-Find / Disjoint Set (Connectivity, Cycle Detection)

**When to reach for it:** "Connected components," "are X and Y in the same group?", "will adding this edge create a cycle?", "number of distinct groups," "minimum spanning tree." Any problem about *grouping* or *dynamic connectivity*.

**In plain English:** Union-Find is like an org-chart. Every element starts as its own boss (root). When two elements join the same group, one boss reports to the other (`union`). "Are A and B in the same group?" becomes "do A and B share the same ultimate boss?" (`find`). **Path compression** (make everyone point directly to the root) and **union by rank** (attach the shorter tree under the taller) make every operation nearly O(1).

**Template:**
```python
class UnionFind:
    def __init__(self, n):
        self.parent = list(range(n))
        self.rank = [0] * n
        self.components = n

    def find(self, x):
        if self.parent[x] != x:
            self.parent[x] = self.find(self.parent[x])  # path compression
        return self.parent[x]

    def union(self, x, y):
        """Returns False if x and y were already connected (cycle!)."""
        rx, ry = self.find(x), self.find(y)
        if rx == ry:
            return False
        if self.rank[rx] < self.rank[ry]:
            rx, ry = ry, rx
        self.parent[ry] = rx
        if self.rank[rx] == self.rank[ry]:
            self.rank[rx] += 1
        self.components -= 1
        return True
```

**Complexity:** ~O(α(n)) per operation (inverse Ackermann — effectively O(1)), Space O(n).

**Representative problems:**
- **[Redundant Connection (LC 684)](https://leetcode.com/problems/redundant-connection/)** — add edges one-by-one; the first edge where `union` returns `False` is redundant.
- **[Number of Connected Components in an Undirected Graph (LC 323)](https://leetcode.com/problems/number-of-connected-components-in-an-undirected-graph/)** — init components = n, decrement on each successful union; return final count.
- **[Graph Valid Tree (LC 261)](https://leetcode.com/problems/graph-valid-tree/)** — valid tree iff exactly n−1 edges and no cycle.

**Common pitfalls:**
- Forgetting path compression — O(n) per call without it.
- Off-by-one in node indexing (0-indexed vs 1-indexed inputs).
- Not decrementing the component count on union — then you can't read it off directly.

### Shortest Path Basics (BFS for Unweighted, Dijkstra for Weighted)

**When to reach for it:** "Minimum steps / hops," "minimum cost to reach," "shortest path in a grid," "network delay time." BFS when all edges cost 1; Dijkstra when edges have non-negative weights.

**In plain English:** BFS naturally finds the shortest path in an unweighted graph because it explores level by level — the moment you first reach the destination, that's the shortest route. For weighted graphs, BFS breaks (a two-hop path might cost less than a one-hop path). Dijkstra fixes this with a **min-heap**: always relax the cheapest unfinalized node next.

**Template:**
```python
import heapq

def dijkstra(graph, start, n):
    # graph[u] = [(weight, v), ...]
    dist = {i: float("inf") for i in range(n)}
    dist[start] = 0
    heap = [(0, start)]                     # (cost, node)
    while heap:
        cost, node = heapq.heappop(heap)
        if cost > dist[node]:               # stale entry
            continue
        for weight, neighbor in graph[node]:
            new_cost = cost + weight
            if new_cost < dist[neighbor]:
                dist[neighbor] = new_cost
                heapq.heappush(heap, (new_cost, neighbor))
    return dist
```

**Complexity:** BFS O(V + E). Dijkstra O((V + E) log V) with a binary heap.

**Representative problems:**
- **[Network Delay Time (LC 743)](https://leetcode.com/problems/network-delay-time/)** — Dijkstra from source; answer is `max(dist.values())`, or −1 if any node is unreachable.
- **[Cheapest Flights Within K Stops (LC 787)](https://leetcode.com/problems/cheapest-flights-within-k-stops/)** — Dijkstra/Bellman-Ford variant with a `stops` dimension.
- **[Word Ladder (LC 127)](https://leetcode.com/problems/word-ladder/)** — BFS on a word graph (edges between words differing by one letter); BFS level = transformation count.

**Common pitfalls:**
- Using DFS for shortest path — DFS finds *a* path, not the *shortest*.
- Skipping the stale-entry check in Dijkstra (`if cost > dist[node]: continue`) — causes TLE.
- Applying Dijkstra to graphs with negative weights — use Bellman-Ford instead.
- Not initializing all distances to infinity before relaxing.

### Dynamic Programming — 1D (Climbing Stairs / House Robber)

**When to reach for it:** "Count the ways," "minimum / maximum over a sequence of choices," "can you reach the end?", overlapping subproblems on a 1-D array. If a brute-force recursion recomputes the same arguments, DP is the tool.

**In plain English:** 1-D DP is like filling a row of boxes left to right, where each box's answer depends only on the one or two boxes to its left. For Climbing Stairs, the ways to reach step *i* is the sum of ways to reach *i−1* and *i−2* — Fibonacci in disguise. For House Robber, at each house you choose between robbing it (plus the best from two back) or skipping it (the best from one back).

**Template:**
```python
def climb_stairs(n):
    if n <= 2:
        return n
    prev2, prev1 = 1, 2
    for _ in range(3, n + 1):
        prev2, prev1 = prev1, prev1 + prev2
    return prev1

def house_robber(nums):
    prev2, prev1 = 0, 0
    for num in nums:
        prev2, prev1 = prev1, max(prev1, prev2 + num)
    return prev1
```

**Complexity:** Time O(n), Space O(1) with rolling variables.

**Representative problems:**
- **[Climbing Stairs (LC 70)](https://leetcode.com/problems/climbing-stairs/)** — ways to reach step n taking 1 or 2 steps; `dp[i] = dp[i-1] + dp[i-2]`.
- **[House Robber (LC 198)](https://leetcode.com/problems/house-robber/)** — max loot from non-adjacent houses; `dp[i] = max(dp[i-1], dp[i-2] + nums[i])`.
- **[Coin Change (LC 322)](https://leetcode.com/problems/coin-change/)** — min coins to make amount; `dp[i] = min(dp[i], dp[i - coin] + 1)`.

**Common pitfalls:**
- Initializing base cases wrongly — always ground-truth `dp[0]`/`dp[1]` before the loop.
- Off-by-one: dp array of size `n` vs `n+1`.
- Forgetting to handle "amount unreachable" in Coin Change (check for infinity in the final answer).
- Over-space: allocating an O(n) array when two rolling variables suffice.

### Dynamic Programming — 2D & Grid Paths

**When to reach for it:** "Number of unique paths in a grid," "minimum path sum," "edit distance," "longest common subsequence," any problem with two varying indices (row + col, or two string positions). The tell: the state needs *two* dimensions to describe it fully.

**In plain English:** Picture a spreadsheet. Each cell's value is computed from the cell above and the cell to its left (or diagonally). You fill it row by row; the answer sits in the bottom-right corner. The 2-D DP grid is an explicit table of all intermediate answers so you never recompute them.

**Template:**
```python
def unique_paths(m, n):
    dp = [[1] * n for _ in range(m)]
    for r in range(1, m):
        for c in range(1, n):
            dp[r][c] = dp[r-1][c] + dp[r][c-1]
    return dp[m-1][n-1]

def lcs(text1, text2):
    m, n = len(text1), len(text2)
    dp = [[0] * (n + 1) for _ in range(m + 1)]
    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if text1[i-1] == text2[j-1]:
                dp[i][j] = 1 + dp[i-1][j-1]
            else:
                dp[i][j] = max(dp[i-1][j], dp[i][j-1])
    return dp[m][n]
```

**Complexity:** Time O(m × n), Space O(m × n) (often reducible to O(min(m, n))).

**Representative problems:**
- **[Unique Paths (LC 62)](https://leetcode.com/problems/unique-paths/)** — `dp[r][c] = above + left`; borders are all 1.
- **[Minimum Path Sum (LC 64)](https://leetcode.com/problems/minimum-path-sum/)** — `dp[r][c] = grid[r][c] + min(above, left)`.
- **[Longest Common Subsequence (LC 1143)](https://leetcode.com/problems/longest-common-subsequence/)** — match characters diagonally; mismatch takes max of left/above.

**Common pitfalls:**
- Forgetting to initialize the first row and column (base cases) before filling the interior.
- Indexing the original string at `i` vs `i-1` when using a 1-indexed DP table.
- Mixing up LCS (characters need not be adjacent) with Longest Common Substring (they must be).
- Not considering space optimization when the interviewer asks.

### Dynamic Programming — 0/1 Knapsack & Subset-Sum

**When to reach for it:** "Can you form a target sum using a subset of these numbers?", "maximum value under a weight limit," "partition array into equal subsets," "count subsets summing to X." The tell: a set of items where each is used *at most once* and you're optimizing or deciding feasibility.

**In plain English:** You have a backpack with a fixed capacity and items with weights and values. For each item you make a binary choice: take it or leave it. The trick is to process items one by one and iterate the capacity *backwards* so you don't accidentally use the same item twice in one pass.

**Template:**
```python
def knapsack(weights, values, capacity):
    dp = [0] * (capacity + 1)
    for w, v in zip(weights, values):
        for cap in range(capacity, w - 1, -1):  # iterate BACKWARDS (0/1)
            dp[cap] = max(dp[cap], dp[cap - w] + v)
    return dp[capacity]

def can_partition(nums):
    total = sum(nums)
    if total % 2:
        return False
    target = total // 2
    dp = {0}
    for num in nums:
        dp |= {s + num for s in dp}
        if target in dp:
            return True
    return target in dp
```

**Complexity:** Time O(n × W), Space O(W) where W is the target/capacity.

**Representative problems:**
- **[Partition Equal Subset Sum (LC 416)](https://leetcode.com/problems/partition-equal-subset-sum/)** — 0/1 subset-sum to `total/2`; early exit if target found.
- **[Coin Change 2 (LC 518)](https://leetcode.com/problems/coin-change-ii/)** — unbounded knapsack (each coin reusable), count combinations; iterate capacity **forwards**.
- **[Target Sum (LC 494)](https://leetcode.com/problems/target-sum/)** — assign +/− to each number; reducible to subset-sum with an algebraic transformation.

**Common pitfalls:**
- Iterating capacity *forwards* in 0/1 knapsack — that allows reuse of the same item (the unbounded variant).
- Odd-sum early exit in partition problems (can't split an odd total into two equal halves).
- Confusing "count combinations" (order doesn't matter) with "count permutations" (loop order flipped).
- Not initializing `dp[0] = 1` when counting ways (one way to make sum 0: pick nothing).

### Dynamic Programming — Longest Increasing Subsequence / Sequence DP

**When to reach for it:** "Longest increasing / non-decreasing subsequence," "minimum deletions to make sorted," "Russian Doll Envelopes." The key signal: a 1-D sequence where you seek an optimal *subsequence* (not subarray) with a monotonic property.

**In plain English:** LIS is like patience sort (stacking cards into piles). You scan left to right and for each element ask "what's the longest increasing run ending here?" The O(n²) approach checks all previous elements. The O(n log n) approach maintains a "tails" array — the smallest possible tail for every subsequence length — and binary-searches to place each new element.

**Template:**
```python
import bisect

def lis_n2(nums):                     # O(n^2), easy to derive live
    dp = [1] * len(nums)              # dp[i] = LIS ending at i
    for i in range(1, len(nums)):
        for j in range(i):
            if nums[j] < nums[i]:
                dp[i] = max(dp[i], dp[j] + 1)
    return max(dp) if nums else 0

def lis_nlogn(nums):                  # O(n log n), patience sort
    tails = []
    for num in nums:
        pos = bisect.bisect_left(tails, num)
        if pos == len(tails):
            tails.append(num)
        else:
            tails[pos] = num
    return len(tails)
```

**Complexity:** O(n²) or O(n log n) time; O(n) space.

**Representative problems:**
- **[Longest Increasing Subsequence (LC 300)](https://leetcode.com/problems/longest-increasing-subsequence/)** — canonical; O(n log n) required for the follow-up.
- **[Russian Doll Envelopes (LC 354)](https://leetcode.com/problems/russian-doll-envelopes/)** — sort by width ascending, height *descending*, then LIS on heights.
- **[Number of Longest Increasing Subsequences (LC 673)](https://leetcode.com/problems/number-of-longest-increasing-subsequences/)** — augment dp with a `count` array; tricky but classic.

**Common pitfalls:**
- Using `bisect_left` vs `bisect_right` — `bisect_left` for strictly increasing, `bisect_right` for non-decreasing.
- The `tails` array is *not* the actual LIS — it only gives the correct *length*.
- In Russian Doll, sorting heights descending at equal widths prevents using two envelopes of the same width.
- Initializing `dp[i] = 0` instead of `1` (every element is an LIS of length 1 by itself).

### The DP Recipe

A repeatable 5-step method for any DP problem — the single most valuable thing to internalize:

1. **Define the state.** What does `dp[i]` (or `dp[i][j]`) *mean* in English? Write it as a sentence. Example: "dp[i] = maximum money robbed from the first i houses."
2. **Write the recurrence (transition).** How does `dp[i]` depend on smaller subproblems? Draw the decision tree: at each state, what are the choices and their costs? Example: `dp[i] = max(dp[i-1], dp[i-2] + nums[i])`.
3. **Identify base cases.** The smallest subproblems you can answer directly. Example: `dp[0] = nums[0]`, `dp[1] = max(nums[0], nums[1])`.
4. **Determine the order of computation.** Smaller subproblems before larger ones — left to right for 1-D, row by row for 2-D, backwards for 0/1 knapsack.
5. **Locate the answer.** Is it `dp[n]`? `dp[n-1]`? `max(dp)`? This is often where bugs hide (for LIS it's `max(dp)`, not `dp[n-1]`).

### Memoization vs. Tabulation (How to Convert)

**In plain English:** Memoization is a *lazy* top-down approach — add a cache (`@lru_cache` or a dict) to your recursion so you never recompute the same call. Tabulation is a *proactive* bottom-up approach — fill a table iteratively from the base cases up. Memoization is easier to derive; tabulation is faster (no recursion overhead) and dodges Python's recursion-depth limit.

```python
from functools import lru_cache

# 1) Naive recursion — O(2^n)
def fib_naive(n):
    if n <= 1: return n
    return fib_naive(n-1) + fib_naive(n-2)

# 2) Memoization (top-down) — O(n)
@lru_cache(maxsize=None)
def fib_memo(n):
    if n <= 1: return n
    return fib_memo(n-1) + fib_memo(n-2)

# 3) Tabulation (bottom-up) — O(n) time, O(1) space
def fib_tab(n):
    a, b = 0, 1
    for _ in range(n):
        a, b = b, a + b
    return a
```

**Conversion checklist:** the recursion's parameters become the table dimensions; the cache key `(i, j)` becomes `dp[i][j]`; the base case of the recursion becomes the table's initialization; the recursive call pattern dictates fill order; `return memo[args]` becomes `return dp[final_state]`.

---

## Cracking the Coding Round — Strategy

### The 7-Step Approach to Any Interview Problem

Skipping steps — especially the first two — is the most common reason candidates fail a problem they actually know how to solve.

1. **Clarify (2–3 min).** Ask about input constraints (size, range, negatives?), return type, and edge cases. "Can I assume the input is valid?" "Should I handle duplicates?" Do not start coding yet — interviewers reward this.
2. **Work concrete examples.** Trace the given example manually, then construct your *own* edge case (empty, single element, all duplicates, already sorted). This prevents assumption errors.
3. **State the brute force first.** Say it out loud: "The naive approach is X, O(Y) time." It shows understanding and gives a baseline to optimize from.
4. **Optimize with pattern recognition.** Ask: "What's the bottleneck? Can a hash map remove repeated work? Is there sorted order for two pointers or binary search? Do subproblems overlap (DP)?" Announce the target complexity before coding.
5. **Code with narration.** Clean, readable code; descriptive names; narrate each block as you write it.
6. **Test and dry-run.** Before saying "done," trace your code against your edge case line by line — using the actual variables, not your mental model.
7. **Analyze complexity.** State time and space explicitly. If asked to optimize, reason about the lower bound and whether it's achievable.

### Signal → Pattern Cheat Table

Match the problem's phrasing to a pattern *before* writing a line of code.

| Problem wording / signal | Pattern to reach for |
|---|---|
| "Find pair/triplet summing to target in sorted array" | Two Pointers |
| "Longest / smallest subarray / substring satisfying condition" | Sliding Window |
| "Subarray sum equals K / range sum queries" | Prefix Sum + Hash Map |
| "Contains duplicate / anagram / count frequency" | Hashing |
| "Maximum contiguous subarray sum" | Kadane's |
| "Overlapping intervals / meeting rooms" | Merge Intervals |
| "Numbers in range 1..n, find missing/duplicate, O(1) space" | Cyclic Sort |
| "Cycle in linked list / find middle" | Fast & Slow Pointers |
| "Search in sorted or rotated array, O(log n)" | Binary Search |
| "Minimize the maximum / can we do it in X?" | Binary Search on the Answer |
| "All permutations / combinations / subsets" | Backtracking |
| "Top K elements / K closest / K most frequent" | Heap (min/max) |
| "Find median of a data stream" | Two Heaps |
| "Balanced parentheses / next greater element" | Stack / Monotonic Stack |
| "Prerequisites / dependencies / ordering of tasks" | Topological Sort |
| "Number of islands / connected components / flood fill" | Graph DFS / BFS |
| "Are X and Y in the same group? Merge groups dynamically" | Union-Find |
| "Minimum cost/steps from A to B (weighted graph)" | Dijkstra |
| "Shortest path in unweighted graph / fewest transformations" | BFS |
| "Autocomplete / words with a given prefix" | Trie |
| "Count ways / min-max over choices with overlapping subproblems" | Dynamic Programming |
| "Is this a valid BST / kth smallest in BST" | BST in-order traversal |

### Communication Tips

Interviewers can't assess your thinking if you go silent. Treat the interview as a **pair-programming session**, not an exam.

- **State intent before coding:** "I'll use a hash map to store frequencies as I scan left to right." This lets the interviewer redirect you early.
- **Narrate trade-offs:** "A sorted set gives O(log n) insertion, but a heap is simpler here — I'll use the heap." You score points for awareness even without the optimal path.
- **State assumptions explicitly:** "I'm assuming the graph is connected; I'll handle the disconnected case after the main logic."
- **Invite feedback:** after stating the approach, "Does that make sense before I code it?" — professional, not weak.
- **When stuck, say so constructively:** "I know I need O(n). I've got a hash-map idea — let me sketch it." Silence over ~90 seconds with no narration is a red flag.

### Testing & Edge Cases

Dry-running is part of the solution, not optional.

- **Always consider:** empty input (`[]`, `""`, `n=0`); single element; all identical / same sign; already sorted / reverse-sorted; negatives or zero; graph with no edges or fully connected; target unreachable.
- **Dry-run technique:** pick your simplest self-made example, write out the key variables (indices, pointers, stack, dp table) and advance them step by step through your *code* — bugs hide in the gap between "what the algorithm should do" and "what the code actually does."
- **Bugs you find during dry-run are not penalized** the way bugs the interviewer finds are.

### Time Management in a 45-Minute Round

| Phase | Time budget |
|---|---|
| Clarify + examples | 3–5 min |
| Brute force + optimization discussion | 5–8 min |
| Coding | 15–20 min |
| Testing / dry-run | 5–7 min |
| Complexity + follow-up | 3–5 min |

- If you're still optimizing at ~15 minutes, code the best solution you have. **A working O(n²) beats an unfinished O(n).**
- Reserve at least 5 minutes for testing.
- Treat a follow-up ("can you do it in O(1) space?") as a new mini-problem: state the new approach before touching code.

### What Interviewers Actually Score

Structured rubrics ([Tech Interview Handbook](https://www.techinterviewhandbook.org/best-practice-questions/)) assess four independent dimensions — you can be weak on one and still pass:

1. **Problem-solving / analytical ability** — did you find the right pattern and optimize? (highest weight)
2. **Coding / implementation quality** — clean, readable, idiomatic; meaningful names; no tangled duplication.
3. **Verification** — did you test and catch your own bugs? (self-caught bugs rate far higher than interviewer-caught ones)
4. **Communication** — thinking out loud, stating assumptions, asking questions. A clear communicator often outscores a silent expert.

**What does not score:** memorizing solutions verbatim. Interviewers vary parameters and ask follow-ups specifically to distinguish understanding from memorization.

---

## Curated Study Plan

A pattern-ordered path (roughly easy → hard). Do 2–3 per pattern; if you can re-derive the template from the intuition, move on.

| # | Pattern | Warm-up | Core | Stretch |
|---|---|---|---|---|
| 1 | Hashing | Two Sum (1) | Group Anagrams (49) | Longest Consecutive Sequence (128) |
| 2 | Two Pointers | Valid Palindrome (125) | 3Sum (15) | Trapping Rain Water (42) |
| 3 | Sliding Window | Best Time to Buy/Sell (121) | Longest Substring w/o Repeat (3) | Minimum Window Substring (76) |
| 4 | Prefix Sum | Range Sum Query (303) | Subarray Sum Equals K (560) | Contiguous Array (525) |
| 5 | Stack | Valid Parentheses (20) | Daily Temperatures (739) | Largest Rectangle in Histogram (84) |
| 6 | Binary Search | Binary Search (704) | Search Rotated Array (33) | Koko Eating Bananas (875) |
| 7 | Linked List | Reverse Linked List (206) | Linked List Cycle (141) | Merge k Sorted Lists (23) |
| 8 | Trees (DFS) | Max Depth (104) | Validate BST (98) | Binary Tree Max Path Sum (124) |
| 9 | Trees (BFS) | Level Order (102) | Right Side View (199) | Word Ladder (127) |
| 10 | Heap | Kth Largest (215) | Top K Frequent (347) | Find Median from Data Stream (295) |
| 11 | Backtracking | Subsets (78) | Combination Sum (39) | Word Search II (212) |
| 12 | Graphs | Number of Islands (200) | Course Schedule (207) | Pacific Atlantic (417) |
| 13 | Union-Find | Number of Provinces (547) | Redundant Connection (684) | Accounts Merge (721) |
| 14 | 1-D DP | Climbing Stairs (70) | House Robber (198) | Coin Change (322) |
| 15 | 2-D DP | Unique Paths (62) | Longest Common Subsequence (1143) | Edit Distance (72) |
| 16 | Knapsack / LIS | Partition Equal Subset Sum (416) | Longest Increasing Subsequence (300) | Target Sum (494) |
| 17 | Trie | Implement Trie (208) | Add & Search Word (211) | Word Search II (212) |
| 18 | Intervals | Merge Intervals (56) | Insert Interval (57) | Meeting Rooms II (253) |
| 19 | Bit Manipulation | Single Number (136) | Number of 1 Bits (191) | Sum of Two Integers (371) |
| 20 | Greedy | Maximum Subarray (53) | Jump Game (55) | Gas Station (134) |

---

## Glossary

| Term | Simple explanation | Purpose |
|---|---|---|
| **Big-O notation** | How an algorithm's time or memory grows as input size grows | Compare efficiency; the headline metric interviewers ask for |
| **O(1) / O(n) / O(log n) / O(n log n)** | Constant / linear / logarithmic / linearithmic growth | Rank solutions from fastest to slowest |
| **Amortized** | Average cost per operation over a long sequence | Explains why dynamic-array append is "O(1) amortized" |
| **In-place** | Uses O(1) extra space, mutating the input directly | Space-constrained problems (cyclic sort, reversal) |
| **Invariant** | A condition that stays true throughout a loop/algorithm | The correctness backbone of two-pointers, binary search, sliding window |
| **Two pointers** | Two indices moving through data to avoid nested loops | Pair/partition problems in O(n) |
| **Sliding window** | An expanding/contracting range over contiguous data | Longest/shortest subarray/substring under a constraint |
| **Prefix sum** | Cumulative totals enabling O(1) range-sum queries | Subarray-sum and range-query problems |
| **Hash map** | Key→value store with average O(1) lookup | Membership, counting, complement lookup |
| **Monotonic stack** | A stack kept in increasing or decreasing order | Next-greater/smaller-element problems |
| **BFS (breadth-first search)** | Explore level by level using a queue | Shortest path in unweighted graphs; level-order |
| **DFS (depth-first search)** | Explore as deep as possible before backtracking | Connectivity, tree traversal, path enumeration |
| **Backtracking** | Build candidates incrementally and undo on failure | Enumerate all subsets/permutations/valid configs |
| **Divide and conquer** | Split into halves, solve each, merge results | Merge sort, quickselect, O(n log n) algorithms |
| **Greedy** | Take the locally best choice at each step | Interval scheduling, jump games (when provably optimal) |
| **Exchange argument** | Proof that swapping to the greedy choice never worsens the result | Justifies that a greedy algorithm is correct |
| **Binary search on the answer** | Binary-search over the value range, not the array | "Minimize the maximum" / monotonic-feasibility problems |
| **Memoization** | Cache results of a top-down recursion | Turn exponential recursion into polynomial DP |
| **Tabulation** | Fill a DP table bottom-up iteratively | Faster DP; avoids recursion-depth limits |
| **DP state** | The variables that fully describe a subproblem | The first (and hardest) step of any DP |
| **Recurrence** | How a DP state is built from smaller states | The transition equation of a DP |
| **Topological sort** | Linear ordering respecting directed dependencies | Prerequisites/build-order; cycle detection in DAGs |
| **In-degree** | Number of incoming edges to a node | Kahn's algorithm seeds nodes with in-degree 0 |
| **Union-Find (Disjoint Set)** | Structure tracking group membership with near-O(1) ops | Connectivity, cycle detection, components |
| **Path compression** | Flattening the union-find tree on lookup | Keeps find/union nearly constant time |
| **Dijkstra's algorithm** | Greedy shortest path using a min-heap | Weighted graphs with non-negative edges |
| **Adjacency list** | `node → list of neighbors` graph representation | Memory-efficient for sparse graphs |
| **Heap / Priority queue** | Structure giving O(1) access to min (or max) | Top-K, merge-K, running median, Dijkstra |
| **Trie (prefix tree)** | Tree where edges are characters | O(L) prefix lookup / autocomplete |
| **Dummy (sentinel) node** | A placeholder head node in a linked list | Removes head-edge special cases |
| **Recursion depth** | How many nested calls are on the stack | Python caps at ~1000; deep recursion needs iteration |

---

## References

- [NeetCode 150 — curated pattern-grouped problem set](https://neetcode.io/practice?tab=neetcode150)
- [Blind 75 — the original curated list](https://leetcode.com/discuss/post/460599/blind-75-leetcode-questions-by-krishnade-9xev/)
- [Grokking the Coding Interview — DesignGurus](https://www.designgurus.io/course/grokking-the-coding-interview) · [Educative version](https://www.educative.io/courses/grokking-coding-interview)
- [DesignGurus — Grokking the Coding Interview: 20 patterns](https://www.designgurus.io/blog/grokking-the-coding-interview-patterns)
- [Tech Interview Handbook — best-practice questions & what interviewers score](https://www.techinterviewhandbook.org/best-practice-questions/)
- [emre.me — coding-pattern write-ups (cyclic sort, fast & slow pointers)](https://emre.me/categories/#coding-patterns)
- [LeetCode — problem set](https://leetcode.com/problemset/)

---

*Previous: [DS Coding & SQL Practice](03-ds-coding-and-sql-practice.md) | Up: [Guide Home](../README.md)*
