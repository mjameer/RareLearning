# 🌳 Binary Tree & BST – LeetCode Beginner Notes

A concise and practical guide to understand **Binary Trees**, **BSTs**, **Tree Traversals**, **Binary Search**, and **Heaps** — with clear examples and Python code.

---

## 🧩 1. Binary Tree

A **binary tree** is a data structure where **each node can have at most two children** — `left` and `right`.

### Example
```
    1
   / \
  2   3

```

### ✅ Representation in Code
```python
class BinaryTree:
    def __init__(self, val, left=None, right=None):
        self.val = val
        self.left = left
        self.right = right
```

---

## 🌱 2. Binary Search Tree (BST)

A **BST** is a special binary tree that follows the ordering rule:

> For any node: `left_child < node.value <= right_child`

### Example
```
        8
       / \
      3   10
     / \    \
    1   6    14

```
- All values in the left subtree of 8 are smaller.
- All values in the right subtree of 8 are greater.

---

## 🌿 3. Binary Tree vs. BST

| Concept | Binary Tree | Binary Search Tree |
|----------|--------------|--------------------|
| Ordering | No rule | Left < Root ≤ Right |
| Duplicates | Allowed | Usually placed on right |
| Use case | Hierarchical data | Searching / sorting |

✅ Every **BST** is a **binary tree**,  
but not every binary tree is a **BST**.

---

## 🌲 4. Balanced Binary Tree

A **balanced binary tree** maintains almost equal heights of left and right subtrees.

**Condition:**
> |height(left) - height(right)| ≤ 1

### Examples

✅ Balanced
```
        2
       / \
      3   4
     /
    1

        2
       / \
      3   4

```

❌ Not Balanced
```
        2
       /
      3
     /
    4
```

---

## 📏 5. Height vs. Depth

| Term | Meaning |
|------|----------|
| **Height of a node** | Distance from node → lowest leaf |
| **Depth of a node** | Distance from root → node |

### Example
```
        1
       / \
      2   3
     / \
    4   5

```

| Node | Height | Depth |
|------|---------|--------|
| 1 | 2 | 0 |
| 2 | 1 | 1 |
| 3 | 0 | 1 |
| 4 | 0 | 2 |
| 5 | 0 | 2 |

**Height of Tree = 2**  
**Depth of Tree = 2**

---

## 🔁 6. Tree Traversal

Traversal = visiting all nodes in a specific order.

There are **two main types**:

### (A) Depth First Search (DFS) — *Recursive*

1. **Preorder (Root → Left → Right)**  
2. **Inorder (Left → Root → Right)** — *Sorted order for BST*  
3. **Postorder (Left → Right → Root)**  

### Example Tree
```
           A
          / \
         B   C
        / \   \
       D   E   F

```

| Type | Order |
|------|--------|
| Preorder | A, B, D, E, C, F |
| Inorder | D, B, E, A, C, F |
| Postorder | D, E, B, F, C, A |

### Example Code (Postorder)
```python
def postorder(root):
    if not root:
        return []
    return postorder(root.left) + postorder(root.right) + [root.val]
```

---

### (B) Breadth First Search (BFS) — *Iterative using Queue*

Visit level by level → **Level Order Traversal**

### Example
```
        1
       / \
      2   3
     / \   \
    4   5   6

```

Output:
```
[[1], [2,3], [4,5,6]]
```

### Example Code
```python
from collections import deque

def levelOrder(root):
    if not root:
        return []
    queue = deque([root])
    result = []
    
    while queue:
        level = []
        for _ in range(len(queue)):
            node = queue.popleft()
            level.append(node.val)
            if node.left:
                queue.append(node.left)
            if node.right:
                queue.append(node.right)
        result.append(level)
    return result
```

---

### Example Problem: Right Side View

> Return the nodes visible from the right side of a tree.

```
        1
       / \
      2   3
           \
            7

```
Output:
```
[1, 3, 7]
```

---

## 🔍 7. Binary Search (Algorithm)

Used on **sorted arrays** to divide the search space by half each step.

**Time complexity:** `O(log n)`

### Example
Array = `[1, 3, 5, 7, 9, 11]`, Target = `7`

Steps:
- mid = 5 → target > mid → search right half  
- mid = 9 → target < mid → search left half  
✅ Found `7`

---

## ⚖️ 8. Linear Search vs Binary Search

| Type | Works On | Time Complexity |
|------|-----------|----------------|
| Linear | Any list | O(n) |
| Binary | Sorted list | O(log n) |

---

## 🧮 9. Heap (Priority Queue)

A **heap** is a binary tree stored as an **array**, allowing fast access to the smallest/largest element.

### Properties
- Left child index = 2*i + 1  
- Right child index = 2*i + 2  

### Example (Min Heap)
```
        2
       / \
      3   4
     / \
    8   9

```
Array = `[2, 3, 4, 8, 9]`

Root (smallest element) = 2

---

### Common Operations

| Operation | Description | Time |
|------------|--------------|------|
| `push()` | Insert and rebalance | O(log n) |
| `pop()` | Remove top element | O(log n) |
| `heapify()` | Build heap from array | O(n) |

### Example Code
```python
import heapq

nums = [9, 4, 7, 1, 2]
heapq.heapify(nums)          # Build min-heap
heapq.heappush(nums, 0)      # Add element
smallest = heapq.heappop(nums)  # Pop smallest
```

---

### Heap Use Cases

| Problem | Heap Type |
|----------|------------|
| Kth Largest | Min Heap |
| Kth Smallest | Max Heap |

**Sorting Complexity:** `O(n log n)`  
**Heap Approach:** `O(n log k)` — faster when `k << n`

✅ **Tip:** Whenever you see “Kth largest/smallest,” think **Heap**.

---

## 🌳 Bonus: Check if a Tree is Balanced

### Example Tree
```
        2
       / \
      3   4
     /
    1

```

### Analysis

| Node | Left Height | Right Height | Difference | Balanced? |
|------|--------------|---------------|-------------|------------|
| 1 | -1 | -1 | 0 | ✅ |
| 3 | 0 | -1 | 1 | ✅ |
| 4 | -1 | -1 | 0 | ✅ |
| 2 | 1 | 0 | 1 | ✅ |

✅ **This tree is balanced.**

If you add one more child under `1`, the height difference at node `3` becomes `2` → ❌ Not balanced.

---

### 🧠 Summary Cheat Sheet

| Concept | Key Idea | Time Complexity |
|----------|-----------|----------------|
| Binary Tree | Each node ≤ 2 children | – |
| BST | Left < Root ≤ Right | Search: O(log n) |
| DFS Traversal | Pre, In, Post | O(n) |
| BFS Traversal | Level order (Queue) | O(n) |
| Binary Search | Sorted array, divide half | O(log n) |
| Heap | Priority queue | Insert/Delete: O(log n) |

---

### 📘 Recommendation
If you’re starting LeetCode, practice these patterns first:
1. Binary Tree Traversal (DFS/BFS)
2. Level Order, Right View, Max Depth
3. Validate BST
4. Kth Smallest / Largest using Heaps
5. Binary Search on Arrays

---

> 💡 **Tip:** Always draw a small tree and trace your recursion manually.  
It’s the fastest way to understand any tree problem.

---

**Author:** MJ  
**Purpose:** For quick LeetCode reference and interview prep.
