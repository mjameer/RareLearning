# 📘 Data Structures – Real-World Analogies & Complexities

## Big-O Complexity (Efficiency of Operations)
- **O(1) Constant time** → Like grabbing the first item on a grocery list.  
- **O(n) Linear time** → Like searching a list of names.  
- **O(n²) Quadratic time** → Like every student shaking hands with all others.  
- **O(log n) Logarithmic time** → Like looking up a word in a dictionary by halving the search.

---

## Array
- **Analogy:** Row of lockers, each with a number.  
- **Characteristics:** Fixed size, contiguous memory.  
- **Operations:**  
  - Access = **O(1)**.  
  - Insert/Delete in middle = **O(n)** (need to shift).  
  - Insert/Delete at end = **O(1)**.  
- **Limitations:** Cannot grow automatically — you must allocate a new array if it’s full.

---

## ArrayList (Dynamic Array)
- **Analogy:** A row of lockers where the building automatically constructs a new, bigger row when needed.  
- **Characteristics:** Grows dynamically by allocating larger space (usually 2× capacity) and copying elements.  
- **Operations:**  
  - Access = **O(1)** (direct index lookup).  
  - Append at end = **O(1) amortized**, but **O(n)** when resize occurs (copying over).  
  - Insert/Delete in middle = **O(n)** (shifting elements).  
- **Advantages over Array:** No manual resizing required, handles growth automatically.

---

## Linked List
- **Analogy:** Train cars connected one by one.  
- **Operations:**  
  - Access = **O(n)** (must traverse).  
  - Insert/Delete with reference = **O(1)**.  
  - Insert/Delete after searching = **O(n)**.  
- **Advantage:** No resizing issues; insertion/deletion doesn’t require shifting.

---

## Stack (LIFO – Last In, First Out)
- **Analogy:** Stack of chips — take the top one first.  
- **Operations:**  
  - Push = **O(1)**.  
  - Pop = **O(1)**.  
  - Search deeper = **O(n)**.  
- **Use cases:** Undo operations, function calls, DFS.

---

## Queue (FIFO – First In, First Out)
- **Analogy:** Standing in line at a store.  
- **Operations:**  
  - Enqueue (add to back) = **O(1)**.  
  - Dequeue (remove from front) = **O(1)**.  
  - Search = **O(n)**.  
- **Use cases:** Scheduling, BFS.

---

## Heap (Priority Queue)
- **Analogy:** Pyramid of boxes with smallest (min-heap) or largest (max-heap) on top.  
- **Operations:**  
  - Access root = **O(1)**.  
  - Insert/Delete = **O(log n)** (bubble up/down).  
- **Use cases:** Priority scheduling, shortest-path algorithms.

---

## HashMap (Key-Value Store)
- **Analogy:** Mailroom where each employee has a personal mailbox.  
- **Operations:**  
  - Lookup/Insert/Delete = **O(1)** average.  
  - Worst case (many **key collisions** at same bucket): **O(n)**.  
- **Collision handling:** Chaining (linked list/tree at bucket) or probing (find next slot).  
- **Python equivalent:** Dictionary.

---

## Binary Search Tree (BST)
- **Analogy:** Family tree with ordering rules (left < parent < right).  
- **Operations:** Access/Insert/Delete = **O(log n)** (halve search space).  
- **Worst case:** If tree is skewed (unbalanced) → **O(n)**.  

---

## Set
- **Analogy:** Thanos’s Infinity Gauntlet — only one of each stone, order doesn’t matter.  
- **Definition:** Unordered collection of **unique** elements (often implemented with hash tables).  
- **Operations:** Insert/Delete/Lookup = **O(1)** average, **O(n)** worst case with collisions.  
- **Use cases:** Duplicate removal, membership checking, set operations.

### reference 

- [youtube ref](https://www.youtube.com/watch?v=O9v10jQkm5c&list=PLqYhjyl3PN1xvuGBzKg2pPDv83m8Bl_3P)
