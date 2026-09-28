# 🧠 LeetCode Pattern Master Notes

> Goal: Become highly proficient at LeetCode and programming competitions by learning to **recognize patterns, derive solutions, and reason about invariants** instead of memorizing solutions.

---

# 0. 🧭 Universal Problem-Solving Framework

Use this for **every** LeetCode problem.

## 1. INPUT
What am I given?

## 2. OUTPUT
What exactly must I return?

## 3. CONSTRAINTS
Look for:
- `n` size
- Value ranges
- Sorted/unsorted?
- Duplicates?
- Negative numbers?
- Graph/tree structure?
- Time-limit implications?

## 4. BRUTE FORCE
What is the most obvious solution?

## 5. BOTTLENECK
Why is the brute-force solution too slow?

## 6. OBSERVATION
What information is being recomputed or wasted?

## 7. PATTERN
Which known pattern removes that waste?

## 8. DATA STRUCTURE
What structure supports the operation efficiently?

## 9. INVARIANT
What must remain true throughout the algorithm?

## 10. ALGORITHM
Write the steps in plain English before coding.

## 11. COMPLEXITY
- Time: ?
- Space: ?

## 12. EDGE CASES
Check:
- Empty input
- One element
- Duplicates
- All same values
- Negative values
- Boundary indices
- Already sorted/reversed
- No valid answer
- Multiple valid answers

### 🧠 Master Mental Model

**Problem → Information to maintain → Invariant → Operation needed → Data structure → Algorithm**

---

# 1. 🗺️ Pattern Recognition Cheat Sheet

| Problem clue | Pattern |
|---|---|
| Duplicate / seen before | Hash Set |
| Frequency / count | Hash Map / Counter |
| Pair + sorted array | Two Pointers |
| Palindrome | Two Pointers |
| Contiguous subarray | Sliding Window / Prefix Sum |
| Longest substring | Sliding Window |
| Range sum | Prefix Sum |
| Subarray sum = K | Prefix Sum + Hash Map |
| Sorted search | Binary Search |
| Minimum X such that... | Binary Search on Answer |
| Parentheses / nested structure | Stack |
| Next greater/smaller | Monotonic Stack |
| Level by level | BFS |
| Shortest unweighted path | BFS |
| Explore connected things | DFS |
| All possible combinations | Backtracking |
| Top K | Heap |
| Repeated min/max | Heap |
| Overlapping ranges | Intervals |
| Linked-list middle/cycle | Fast & Slow Pointers |
| Hierarchical data | Tree |
| Connections / relationships | Graph |
| Same connected component | Union Find |
| Prerequisites / dependencies | Topological Sort |
| Weighted shortest path | Dijkstra |
| Local optimal choices | Greedy |
| Repeated subproblems | Dynamic Programming |
| Prefix matching | Trie |
| XOR / binary state | Bit Manipulation |

---

# 2. 🔑 Hash Map / Hash Set

## 🧠 Mental Model

> "I need to remember something I've already seen so I can look it up instantly."

Instead of repeatedly searching the previous elements, store information about them.

## 🔎 Recognize

Look for:
- Have I seen this before?
- Duplicate detection
- Frequency/counting
- Matching two pieces of information
- Grouping
- Lookup by key
- "Find the complement"

## 🧩 Generic Template — Set

```python
seen = set()

for x in nums:
    if x in seen:
        # already seen
        ...
    seen.add(x)
```

## 🧩 Generic Template — Map

```python
seen = {}

for x in nums:
    if x in seen:
        # use stored information
        ...
    seen[x] = information
```

## 🧩 Frequency Template

```python
from collections import Counter

count = Counter(nums)
```

## 👀 Visual

```text
Input stream
   ↓
[2, 7, 11, 15]
   ↓
Memory:
2 → seen
7 → seen
11 → seen
...
```

## 💡 Example — Two Sum

```python
seen = {}

for i, x in enumerate(nums):
    needed = target - x

    if needed in seen:
        return [seen[needed], i]

    seen[x] = i
```

### Why it works

For each `x`, ask:

```text
What number do I need?
needed = target - x
```

Then check whether that number was already seen.

## ⏱️ Complexity

- Time: `O(n)`
- Space: `O(n)`

## 🔒 Invariant

The hash structure accurately represents everything we have already processed.

## ⚠️ Common Mistake

Using a hash map without asking what information should be stored.

The key could map to:
- index
- frequency
- latest position
- earliest position
- another object

---

# 3. 👈👉 Two Pointers

## 🧠 Mental Model

> "I can solve this by maintaining two positions and moving them intelligently instead of checking every combination."

## 🔎 Recognize

- Sorted array
- Pair problems
- Palindrome
- Compare both ends
- Remove duplicates
- Partitioning
- Linked-list pointer manipulation

## 🧩 Generic Template

```python
left = 0
right = len(nums) - 1

while left < right:

    # evaluate current pair

    if condition:
        left += 1
    elif condition:
        right -= 1
    else:
        left += 1
        right -= 1
```

## 👀 Visual

```text
[ 1  2  4  6  8  9 ]
  ↑                 ↑
 left              right
```

Move a pointer only when you can prove the other side cannot produce the answer.

## 💡 Example — Sorted Two Sum

```python
left = 0
right = len(nums) - 1

while left < right:
    total = nums[left] + nums[right]

    if total == target:
        return True
    elif total < target:
        left += 1
    else:
        right -= 1
```

## ⏱️ Complexity

- Time: `O(n)`
- Space: `O(1)`

## 🔒 Invariant

Everything outside the active pointer region has already been correctly ruled out or processed.

## ⚠️ Common Mistake

Moving both pointers without proving why.

---

# 4. 🪟 Sliding Window

## 🧠 Mental Model

> "I need information about a contiguous region, so I'll maintain a window instead of recalculating every subarray."

## 🔎 Recognize

- Subarray
- Substring
- Contiguous
- Longest
- Shortest
- Maximum/minimum
- At most K
- Without duplicates

## 🧩 Generic Template

```python
left = 0
result = 0

for right in range(len(nums)):

    # add nums[right]

    while window_is_invalid():

        # remove nums[left]
        left += 1

    result = max(result, right - left + 1)
```

## 🧩 Fixed-Size Window

```python
left = 0

for right in range(len(nums)):

    # add nums[right]

    if right - left + 1 > k:
        # remove nums[left]
        left += 1

    if right - left + 1 == k:
        # process window
        ...
```

## 👀 Visual

```text
a b c d e f g
  └─────┘
  window

        → move →
```

## 💡 Example — Longest Substring Without Repeating Characters

```python
left = 0
seen = set()
answer = 0

for right in range(len(s)):

    while s[right] in seen:
        seen.remove(s[left])
        left += 1

    seen.add(s[right])

    answer = max(answer, right - left + 1)
```

## ⏱️ Complexity

- Time: `O(n)`
- Space: `O(k)` / `O(n)` depending on the window data.

## 🔒 Invariant

The current window satisfies the required constraint.

## ⚠️ Critical Distinction

Sliding Window generally works when the property can be maintained as the window expands/shrinks.

For many **sum** problems involving arbitrary negative numbers, Prefix Sum may be the better pattern.

---

# 5. ➕ Prefix Sum

## 🧠 Mental Model

> "Precompute cumulative information so future range queries become cheap."

## 🔎 Recognize

- Range sum
- Multiple sum queries
- Subarray sum
- Cumulative values
- Count subarrays with target sum

## 🧩 Generic Template

```python
prefix = [0]

for x in nums:
    prefix.append(prefix[-1] + x)

range_sum = prefix[R + 1] - prefix[L]
```

## 👀 Visual

```text
nums:
  2   4   1   5   3

prefix:
0   2   6   7   12  15
```

Range `[1,3]`:

```text
prefix[4] - prefix[1]
= 12 - 2
= 10
```

## 🧩 Advanced — Subarray Sum K

```python
prefix = 0
count = {0: 1}
answer = 0

for x in nums:
    prefix += x

    needed = prefix - k

    if needed in count:
        answer += count[needed]

    count[prefix] = count.get(prefix, 0) + 1
```

## ⏱️ Complexity

- Time: `O(n)`
- Space: `O(n)`

## 🔒 Invariant

`prefix[i]` represents the cumulative information through index `i - 1`.

---

# 6. 🔍 Binary Search

## 🧠 Mental Model

> "I have an ordered search space. Each decision lets me eliminate roughly half of it."

## 🔎 Recognize

- Sorted array
- Search
- First/last occurrence
- Boundary
- Minimum possible X
- Maximum possible X
- "Can X work?"

## 🧩 Generic Exact Search

```python
left = 0
right = len(nums) - 1

while left <= right:

    mid = left + (right - left) // 2

    if nums[mid] == target:
        return mid

    elif nums[mid] < target:
        left = mid + 1

    else:
        right = mid - 1

return -1
```

## 🧩 Binary Search on Answer

```python
left = minimum_possible
right = maximum_possible

while left < right:

    mid = (left + right) // 2

    if can_do_it(mid):
        right = mid
    else:
        left = mid + 1

return left
```

## 👀 Visual

```text
NO  NO  NO  NO | YES YES YES YES
                ↑
             boundary
```

## ⏱️ Complexity

- Time: `O(log n)` for ordinary binary search.
- Answer search: `O(log range × cost(can_do_it))`.

## 🔒 Invariant

The answer always remains inside the current search range.

## ⚠️ Recognition Trick

Whenever you see:

> "Find the minimum X such that..."

Ask:

> "Is `can_do_it(X)` monotonic?"

If yes, Binary Search on Answer may work.

---

# 7. 📚 Stack

## 🧠 Mental Model

> "The most recently unresolved item should be processed first."

LIFO:

```text
Last In → First Out
```

## 🔎 Recognize

- Parentheses
- Nested structures
- Undo
- Previous/next unresolved state
- Expression parsing
- Backtracking through recent states

## 🧩 Generic Template

```python
stack = []

for x in data:

    while stack and condition(stack[-1], x):
        stack.pop()

    stack.append(x)
```

## 🧩 Parentheses Template

```python
stack = []

for x in data:

    if opening(x):
        stack.append(x)

    else:
        if not stack:
            return False

        top = stack.pop()

        if not matches(top, x):
            return False

return len(stack) == 0
```

## 👀 Visual

```text
      ┌───┐
      │ C │ ← top
      ├───┤
      │ B │
      ├───┤
      │ A │
      └───┘
```

## 🔒 Invariant

The stack contains unresolved elements/states in the order they need to be resolved.

---

# 8. 📈 Monotonic Stack

## 🧠 Mental Model

> "Keep only candidates that can still become useful. Remove candidates that are permanently dominated."

## 🔎 Recognize

- Next greater element
- Next smaller element
- Previous greater/smaller
- Daily Temperatures
- Stock Span
- Largest Rectangle in Histogram

## 🧩 Generic Template

```python
stack = []

for i, x in enumerate(nums):

    while stack and nums[stack[-1]] < x:

        j = stack.pop()

        # x is the next greater element for j

    stack.append(i)
```

## 👀 Visual

```text
2, 1, 5, 3, 4

When 5 arrives:

5 resolves:
2 → 5
1 → 5
```

## ⏱️ Complexity

Usually:
- Time: `O(n)`
- Space: `O(n)`

Why `O(n)` despite the nested `while`?

Each element is pushed once and popped at most once.

## 🔒 Invariant

The stack maintains a monotonic order of unresolved candidates.

---

# 9. 🌊 BFS — Breadth-First Search

## 🧠 Mental Model

> "Explore the closest possibilities first."

Think of ripples spreading through water.

## 🔎 Recognize

- Shortest path in an unweighted graph
- Minimum number of moves
- Fewest steps
- Level-by-level traversal
- Grid problems
- Word Ladder

## 🧩 Generic Template

```python
from collections import deque

queue = deque([start])
visited = {start}

while queue:

    node = queue.popleft()

    for neighbor in graph[node]:

        if neighbor not in visited:
            visited.add(neighbor)
            queue.append(neighbor)
```

## 🧩 With Distance

```python
queue = deque([(start, 0)])
visited = {start}

while queue:

    node, distance = queue.popleft()

    for neighbor in graph[node]:

        if neighbor not in visited:
            visited.add(neighbor)
            queue.append((neighbor, distance + 1))
```

## 👀 Visual

```text
          A
       /  |  \
      B   C   D
     / \
    E   F

Level 0: A
Level 1: B C D
Level 2: E F
```

## 🔒 Invariant

When a node is processed, BFS has found its minimum distance from the start in an unweighted graph.

## ⏱️ Complexity

- Time: `O(V + E)`
- Space: `O(V)`

---

# 10. 🌲 DFS — Depth-First Search

## 🧠 Mental Model

> "Follow one possibility as deeply as possible before exploring another."

## 🔎 Recognize

- Explore a graph/tree
- Connected components
- Islands
- Reachability
- Recursive structures
- Flood fill

## 🧩 Generic Template

```python
def dfs(node):

    if node in visited:
        return

    visited.add(node)

    for neighbor in graph[node]:
        dfs(neighbor)
```

## 🧩 Tree Template

```python
def dfs(node):

    if not node:
        return

    dfs(node.left)
    dfs(node.right)
```

## 👀 Visual

```text
A
├── B
│   ├── D
│   └── E
└── C
```

DFS:

```text
A → B → D
      ↘ E
   backtrack
      → C
```

## 🔒 Invariant

Every visited node has been explored according to the DFS rule.

## ⏱️ Complexity

- Time: `O(V + E)`
- Space: `O(V)` worst case.

---

# 11. 🔙 Backtracking

## 🧠 Mental Model

> **Choose → Explore → Undo → Choose again**

Backtracking is DFS over a decision tree.

## 🔎 Recognize

- All combinations
- All permutations
- Subsets
- Sudoku
- N-Queens
- Word Search
- Generate all possibilities

## 🧩 Generic Template

```python
def backtrack(path):

    if is_complete(path):
        result.append(path.copy())
        return

    for choice in choices:

        if not valid(choice):
            continue

        path.append(choice)

        backtrack(path)

        path.pop()
```

## 👀 Visual

```text
          Start
         /     \
        A       B
       / \     / \
      AB AC   BA BB
```

Each branch represents a choice.

## 🔒 Invariant

`path` exactly represents the choices made along the current branch.

## ⚠️ Critical Rule

The `pop()` is not optional.

```python
path.append(choice)
backtrack(path)
path.pop()
```

That is the **undo** step.

---

# 12. 🏆 Heap / Priority Queue

## 🧠 Mental Model

> "I repeatedly need the most important candidate, but I don't need everything fully sorted."

## 🔎 Recognize

- Top K
- K smallest/largest
- Repeated minimum/maximum
- Scheduling
- Median
- Merge sorted sequences

## 🧩 Min Heap

```python
import heapq

heap = []

for x in nums:
    heapq.heappush(heap, x)

smallest = heapq.heappop(heap)
```

## 🧩 Top K Pattern

```python
heap = []

for x in nums:

    heapq.heappush(heap, x)

    if len(heap) > k:
        heapq.heappop(heap)
```

## 👀 Visual

```text
        smallest
           ↓
          [2]
        /     \
      [4]     [7]
```

The root gives the highest-priority item.

## 🔒 Invariant

The heap contains the candidates currently relevant to the answer.

## ⏱️ Complexity

Heap operation:
- Push: `O(log n)`
- Pop: `O(log n)`
- Peek: `O(1)`

---

# 13. 📅 Intervals

## 🧠 Mental Model

> "Sort intervals by their starting point, then sweep left to right while tracking overlap."

## 🔎 Recognize

- Meeting rooms
- Calendar
- Scheduling
- Overlapping ranges
- Merge intervals
- Insert interval

## 🧩 Generic Template

```python
intervals.sort()

result = []

for start, end in intervals:

    if not result or start > result[-1][1]:

        result.append([start, end])

    else:

        result[-1][1] = max(result[-1][1], end)
```

## 👀 Visual

```text
[1────3]
   [2────────6]

becomes

[1────────6]
```

## 🔒 Invariant

`result` contains the correctly merged intervals processed so far.

## ⏱️ Complexity

- Sorting: `O(n log n)`
- Sweep: `O(n)`
- Overall: `O(n log n)`

---

# 14. 🐢🐇 Fast & Slow Pointers

## 🧠 Mental Model

> "Different speeds reveal hidden structure."

## 🔎 Recognize

- Linked-list cycle
- Find middle
- Cycle entry
- Palindrome linked list
- Repeated-state problems

## 🧩 Generic Template

```python
slow = head
fast = head

while fast and fast.next:

    slow = slow.next
    fast = fast.next.next
```

If they meet:

```python
if slow == fast:
    # cycle exists
```

## 👀 Visual

```text
slow:  → → → →
fast:  → → → → → → →
                  ↘
                    ↖
```

## 🔒 Invariant

The relative speed difference lets the two pointers expose cycle structure.

---

# 15. 🌳 Trees

## 🧠 Mental Model

> "Every node is the root of its own smaller tree."

## 🔎 Recognize

- Hierarchy
- Parent/child relationships
- Binary tree
- BST
- Depth
- Height
- Paths
- Subtrees

## 🧩 Generic DFS Template

```python
def dfs(node):

    if not node:
        return

    left = dfs(node.left)
    right = dfs(node.right)

    # combine information
```

## 🔑 Key Question

> "What information should the child return to the parent?"

This question solves many tree problems.

## Traversals

### Preorder

```text
Root → Left → Right
```

### Inorder

```text
Left → Root → Right
```

For a BST, inorder traversal gives sorted order.

### Postorder

```text
Left → Right → Root
```

Useful when the parent depends on child results.

### Level Order

Use BFS.

## 🔒 Invariant

Each recursive call correctly solves the problem for that node's subtree.

---

# 16. 🕸️ Graphs

## 🧠 Mental Model

> "A graph represents relationships. My job is usually to explore, optimize, or reason about those relationships."

## 🔎 Recognize

- Connections
- Roads
- Networks
- Friendships
- Dependencies
- Paths
- Reachability

## 🧩 Generic Representation

```python
graph = {
    0: [1, 2],
    1: [0, 3],
    2: [0],
    3: [1]
}
```

## 🔑 First Questions

1. What does a node represent?
2. What does an edge represent?
3. Is the graph directed?
4. Is it weighted?
5. Do I need shortest path?
6. Do I need connected components?
7. Do I need dependency ordering?

## Choose:

```text
Unweighted shortest path → BFS
Explore/reachability → DFS
Same components → Union Find
Dependencies → Topological Sort
Weighted shortest path → Dijkstra
```

---

# 17. 🔗 Union Find / DSU

## 🧠 Mental Model

> "I need to efficiently maintain groups and determine whether two elements belong to the same group."

## 🔎 Recognize

- Connected components
- Dynamic connectivity
- Network merging
- Redundant connection
- Kruskal's algorithm

## 🧩 Generic Template

```python
parent = list(range(n))

def find(x):

    if parent[x] != x:
        parent[x] = find(parent[x])

    return parent[x]


def union(a, b):

    root_a = find(a)
    root_b = find(b)

    if root_a != root_b:
        parent[root_a] = root_b
```

## 👀 Visual

Initially:

```text
1   2   3   4   5
```

After unions:

```text
1──2──3      4──5
```

Two components.

## 🔒 Invariant

The parent structure correctly represents which elements belong to the same connected component.

## ⚡ Optimization

Use:
- Path compression
- Union by rank/size

This makes operations extremely efficient in practice.

---

# 18. 📋 Topological Sort

## 🧠 Mental Model

> "I need an ordering that respects dependencies."

## 🔎 Recognize

- Prerequisites
- Dependencies
- Course Schedule
- Build order
- Task ordering
- A before B

## 🧩 Kahn's Algorithm

```python
from collections import deque

queue = deque()

for node in graph:

    if indegree[node] == 0:
        queue.append(node)

while queue:

    node = queue.popleft()

    for neighbor in graph[node]:

        indegree[neighbor] -= 1

        if indegree[neighbor] == 0:
            queue.append(neighbor)
```

## ⚠️ Cycle Detection

If fewer than `n` nodes are processed:

```text
Cycle exists.
```

## 👀 Visual

```text
A → C
↓   ↓
B → D
```

Possible ordering:

```text
A → B → C → D
```

## 🔒 Invariant

Every node added to the result has all required prerequisites satisfied.

---

# 19. 🚗 Dijkstra's Algorithm

## 🧠 Mental Model

> "Always expand the currently cheapest known route."

## 🔎 Recognize

- Weighted graph
- Shortest path
- Edge weights are non-negative

## 🧩 Generic Template

```python
import heapq

dist = [float("inf")] * n
dist[start] = 0

heap = [(0, start)]

while heap:

    distance, node = heapq.heappop(heap)

    if distance > dist[node]:
        continue

    for neighbor, weight in graph[node]:

        new_dist = distance + weight

        if new_dist < dist[neighbor]:

            dist[neighbor] = new_dist

            heapq.heappush(
                heap,
                (new_dist, neighbor)
            )
```

## 👀 Visual

```text
A --2--> B
|        |
5        1
|        |
↓        ↓
C --2--> D
```

Dijkstra repeatedly expands the cheapest currently known route.

## 🔒 Invariant

When a node is finalized/processed with its minimum distance under the algorithm's conditions, no cheaper route will later be found.

## ⚠️ Important

Standard Dijkstra does **not** work correctly with negative edge weights.

---

# 20. 🧠 Greedy

## 🧠 Mental Model

> "Make the best local decision and prove this choice cannot hurt the optimal global solution."

## 🔎 Recognize

- Scheduling
- Resource allocation
- Maximize/minimize
- Choose earliest/latest
- Local choices
- Interval optimization

## 🧩 Generic Template

```python
items.sort(key=greedy_rule)

answer = initial_state

for item in items:

    if can_take(item):

        take(item)
        update(answer)

return answer
```

## 👀 Visual

```text
Choices:
A   B   C   D   E

Take the locally best choice
        ↓
Reduce problem
        ↓
Repeat
```

## 🔒 Invariant

Every greedy choice preserves the possibility of reaching an optimal global solution.

## ⚠️ Critical Warning

Greedy is **not**:

> "Take whatever looks best."

You need a reason/proof that the local choice is safe.

---

# 21. 🔄 Dynamic Programming

## 🧠 Mental Model

> "My problem contains smaller states whose answers repeat. Solve each state once and reuse it."

## 🔎 Recognize

- Overlapping subproblems
- "What is the best way to..."
- Current answer depends on previous states
- Choices + repeated states
- Optimization/counting problems

## 🧩 Top-Down Template

```python
memo = {}

def dp(state):

    if state in memo:
        return memo[state]

    if base_case(state):
        return base_value

    answer = ...

    for choice in choices:

        answer = combine(
            answer,
            dp(next_state)
        )

    memo[state] = answer

    return answer
```

## 🧩 Bottom-Up Template

```python
dp = [...]

# initialize base cases

for state in states:

    dp[state] = recurrence(...)

return dp[target]
```

## 👀 Visual

Without DP:

```text
        A
      /   \
     B     B
    / \   / \
   C   C C   C
```

Repeated `B` and `C` are recomputed.

With memoization:

```text
B → compute once
C → compute once
```

## 🔑 DP Derivation Questions

1. What is the state?
2. What does `dp[state]` mean?
3. What choices can I make?
4. What smaller states result?
5. What is the recurrence?
6. What are the base cases?
7. In what order should states be computed?

## 🔒 Invariant

`dp[state]` stores the correct answer for that state.

---

# 22. 🌳 Trie

## 🧠 Mental Model

> "Many strings share prefixes, so store shared prefixes only once."

## 🔎 Recognize

- Prefix
- Autocomplete
- Dictionary
- Word search
- Prefix matching
- Many string lookups

## 🧩 Generic Template

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

        for char in word:

            if char not in node.children:
                node.children[char] = TrieNode()

            node = node.children[char]

        node.is_end = True
```

## 👀 Visual

```text
        root
       /    \
      c      d
      |      |
      a      o
     / \     |
    t   r    g
```

Words can share the same prefix path.

## 🔒 Invariant

Every path from the root represents a prefix of one or more stored words.

## ⏱️ Complexity

For word length `L`:

- Insert: `O(L)`
- Search: `O(L)`
- Prefix search: `O(L)`

---

# 23. 🔢 Bit Manipulation

## 🧠 Mental Model

> "Treat numbers as collections of binary states instead of decimal values."

## 🔎 Recognize

- XOR
- Binary representation
- Powers of 2
- Unique number
- Missing number
- Bit states
- Subsets represented by masks

## 🧩 Check Bit

```python
if x & (1 << i):
    ...
```

## 🧩 Set Bit

```python
x |= (1 << i)
```

## 🧩 Clear Bit

```python
x &= ~(1 << i)
```

## 🧩 Toggle Bit

```python
x ^= (1 << i)
```

## 🧩 XOR Template

```python
answer = 0

for x in nums:
    answer ^= x

return answer
```

Important properties:

```text
x ^ x = 0
x ^ 0 = x
```

Therefore pairs cancel.

## 👀 Visual

```text
Decimal:  5
Binary:  101

bit positions:
  2 1 0
  1 0 1
```

## 🔒 Invariant

Each bit operation preserves or transforms exactly the binary state required by the problem.

---

# 24. 🧠 Pattern Selection Flowchart

When you see a new problem, ask:

```text
                    START
                      │
                      ▼
              Is it contiguous?
                /          \
              YES           NO
               │             │
        Sliding Window    Continue
        / Prefix Sum
                             │
                             ▼
                     Is it sorted?
                       /       \
                     YES        NO
                      │          │
                Two Pointers   Continue
                / Binary
                Search
                               │
                               ▼
                     Need fast lookup?
                       /       \
                     YES        NO
                      │          │
                 Hash Map/Set  Continue
                               │
                               ▼
                      Need previous
                      unresolved items?
                       /       \
                     YES        NO
                      │          │
                    Stack      Continue
                               │
                               ▼
                       Next greater/
                       smaller?
                          │
                          ▼
                   Monotonic Stack
                               │
                               ▼
                     Need shortest path?
                       /          \
                     YES           NO
                      │             │
               Unweighted?       Continue
                 /     \
               YES      NO
                │        │
               BFS    Dijkstra
                               │
                               ▼
                   Need all possibilities?
                          │
                          ▼
                      Backtracking
                               │
                               ▼
                   Repeated subproblems?
                          │
                          ▼
                          DP
```

---

# 25. 🧩 Data Structure Selection Guide

| Need | Use |
|---|---|
| Fast membership | Set |
| Key → value | Hash Map |
| First/last unresolved | Stack |
| Minimum/maximum repeatedly | Heap |
| Ordered search | Binary Search |
| Prefix queries | Trie |
| Connected components | Union Find |
| FIFO exploration | Queue |
| LIFO exploration | Stack |
| Range accumulation | Prefix Sum |
| Contiguous constraint | Sliding Window |
| Hierarchical relationships | Tree |
| Arbitrary relationships | Graph |

---

# 26. 🔒 Invariant Master List

| Pattern | Core invariant |
|---|---|
| Hash Map | Stored information accurately represents processed data |
| Two Pointers | Outside pointer region is already handled |
| Sliding Window | Current window satisfies the constraint |
| Prefix Sum | Prefix value represents cumulative information |
| Binary Search | Answer remains inside search range |
| Stack | Stack contains unresolved candidates |
| Monotonic Stack | Stack maintains monotonic ordering |
| BFS | Processed nodes have minimum distance in unweighted graph |
| DFS | Visited nodes have been explored |
| Backtracking | Path exactly represents current choices |
| Heap | Heap contains relevant candidates |
| Intervals | Result contains correctly merged intervals |
| Fast/Slow | Relative speed exposes cycle/position information |
| Tree DFS | Recursive result correctly solves subtree |
| Union Find | Parent structure represents components |
| Topological Sort | Processed nodes have satisfied prerequisites |
| Dijkstra | Known shortest distances are maintained |
| Greedy | Local choice remains globally safe |
| DP | `dp[state]` is correct for that state |
| Trie | Tree paths represent prefixes |
| Bit Manipulation | Binary state is transformed correctly |

---

# 27. 🚨 Common Pattern Confusions

## Sliding Window vs Prefix Sum

### Sliding Window
Use when you need to maintain a **valid contiguous window**.

### Prefix Sum
Use when you need **fast cumulative/range information**, especially when negative values make ordinary window logic difficult.

---

## BFS vs DFS

### BFS
Use when:
- Minimum number of steps
- Unweighted shortest path
- Level order

### DFS
Use when:
- Exploring everything
- Components
- Reachability
- Recursive structure

---

## Stack vs Monotonic Stack

### Stack
General LIFO behavior.

### Monotonic Stack
Stack additionally maintains increasing/decreasing order to answer next/previous greater/smaller questions efficiently.

---

## Heap vs Sorting

### Sorting
Use when you need the **entire ordering**.

### Heap
Use when you repeatedly need only the **most important candidate**.

---

## Greedy vs DP

### Greedy
One local decision is permanently safe.

### DP
Different choices can lead to different future consequences, so states must be compared/reused.

---

## DFS vs Backtracking

Backtracking is essentially:

> DFS + choices + undo

Regular DFS often explores a fixed graph/tree.

Backtracking explores a **decision space** and modifies the current path.

---

# 28. 🎯 How to Become Pro-Level

Don't memorize 500 solutions.

Instead, for every problem, force yourself to answer:

```text
1. What is the brute force?
2. Why is it too slow?
3. What information am I repeatedly recomputing?
4. What information should I maintain?
5. What data structure makes that cheap?
6. What invariant must remain true?
7. Why is the algorithm correct?
8. Why is the complexity better?
```

## The real progression

```text
Problem
   ↓
Recognize pattern
   ↓
Recall invariant
   ↓
Choose data structure
   ↓
Derive algorithm
   ↓
Implement
   ↓
Analyze complexity
   ↓
Review mistake
   ↓
Recognize faster next time
```

---

# 29. 📈 Suggested Learning Order

Don't learn all patterns randomly.

## Phase 1 — Foundations

1. Arrays
2. Strings
3. Hash Map / Set
4. Two Pointers
5. Sliding Window
6. Prefix Sum
7. Stack
8. Binary Search

## Phase 2 — Core Data Structures

9. Linked Lists
10. Fast & Slow Pointers
11. Trees
12. BFS
13. DFS
14. Heap
15. Intervals

## Phase 3 — Advanced Algorithms

16. Backtracking
17. Graphs
18. Union Find
19. Topological Sort
20. Dijkstra
21. Greedy
22. Trie
23. Bit Manipulation

## Phase 4 — Dynamic Programming

24. 1D DP
25. 2D DP
26. Knapsack
27. Subsequence DP
28. Grid DP
29. State-machine DP
30. Advanced DP

---

# 30. 📝 Problem Review Template

After solving every problem, record:

## Problem
**Name:**  
**Difficulty:**  
**Pattern:**  

## 1. What was the brute force?

## 2. Why was brute force too slow?

## 3. What clue revealed the pattern?

## 4. What data structure did I use?

## 5. What invariant did I maintain?

## 6. What was the key observation?

## 7. What was the algorithm?

## 8. Time Complexity

## 9. Space Complexity

## 10. What mistake did I make?

## 11. What would I recognize faster next time?

## 12. Can I implement it again without looking?

---

# 🏆 Final Mental Model

When you get stuck, don't immediately search for the solution.

Ask:

> **What am I maintaining?**

Then:

> **What must always remain true?**

Then:

> **What operation do I need to perform quickly?**

Then:

> **Which data structure makes that operation cheap?**

That is the foundation of becoming strong at LeetCode and programming competitions.
