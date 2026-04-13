# Lab 3: Observing Linked Lists

**CPTR 242 — Sequential and Parallel Data Structures and Algorithms**
Walla Walla University

---

## Overview

In this lab you will **run complete programs and observe their behavior**. There is no coding required. Your job is to read the output carefully, record what you see, and reason about what it means.

By the end you should be able to:

- Explain how pointer manipulation produces correct insert and remove operations on linked lists
- Describe the performance difference between linked lists and array-based lists for different operations
- Recognize O(1) and O(n) behavior in list operations from empirical data
- Explain what memory layout looks like for linked nodes versus contiguous arrays

---

## Background: Two Ways to Build a List

A list is an ordered sequence of elements. But the same ADT can be implemented in fundamentally different ways:

- **Linked list** — each element lives in its own heap-allocated node and holds a pointer to the next node. Elements can be scattered anywhere in memory.
- **Array-based list** — elements are stored contiguously in a single block of memory, like `std::vector`.

Today you will observe how each implementation behaves: how pointers change when you insert and remove, what addresses nodes live at, how operations scale, and how memory allocation patterns differ.

---

## Setup

All programs are self-contained. Each generates its own data and prints results directly to the terminal.

**Compiler command for all programs:**
```bash
g++ -O0 -o program program.cpp && ./program
```

> `-O0` disables compiler optimizations so you observe true behavior, not the compiler's shortcuts.

---

## Model 1: Pointers in Motion

The most important thing to understand about a linked list is that every insert and remove is really just a careful reassignment of pointers. This program traces every pointer change so you can see exactly what happens at each step.

### Program 1 — `list_pointer_trace.cpp`

```cpp
#include <iostream>
#include <iomanip>
using namespace std;

struct Node {
    int   data;
    Node* next;
    Node(int v) : data(v), next(nullptr) {}
};

void printList(Node* head, const string& label) {
    cout << label << ": ";
    for (Node* c = head; c; c = c->next)
        cout << "[" << c->data << "@" << c << "]";
    if (head) cout << " → null";
    cout << "\n";
}

void printAddresses(Node* head) {
    cout << "  Addresses: ";
    for (Node* c = head; c; c = c->next)
        cout << c << "(→" << c->next << ") ";
    cout << "\n";
}

// Insert newNode after prev. Pass nullptr for prev to insert at front.
void insertAfter(Node*& head, Node* prev, int val) {
    Node* n = new Node(val);
    cout << "\n  → Inserting " << val;
    if (!prev) {
        cout << " at FRONT\n";
        cout << "    Before: head=" << head << "\n";
        n->next = head;
        head = n;
        cout << "    After:  head=" << head
             << "  new->next=" << n->next << "\n";
    } else {
        cout << " after node [" << prev->data << "@" << prev << "]\n";
        cout << "    Before: prev->next=" << prev->next << "\n";
        n->next = prev->next;   // step 1
        prev->next = n;         // step 2
        cout << "    After:  new->next=" << n->next
             << "  prev->next=" << prev->next << "\n";
    }
}

void removeAfter(Node*& head, Node* prev) {
    if (!prev) {
        // Remove head
        Node* gone = head;
        cout << "\n  → Removing HEAD node [" << head->data
             << "@" << head << "]\n";
        cout << "    Before: head=" << head << "\n";
        head = head->next;
        delete gone;
        cout << "    After:  head=" << head << "\n";
    } else {
        Node* gone = prev->next;
        cout << "\n  → Removing [" << gone->data << "@" << gone
             << "] (after [" << prev->data << "])\n";
        cout << "    Before: prev->next=" << prev->next << "\n";
        prev->next = gone->next;
        delete gone;
        cout << "    After:  prev->next=" << prev->next << "\n";
    }
}

int main() {
    Node* head = nullptr;

    cout << "=== Building a singly-linked list ===\n";
    printList(head, "Initial");

    // Insert 30 at front
    insertAfter(head, nullptr, 30);
    printList(head, "After insert 30 at front");

    // Insert 10 at front
    insertAfter(head, nullptr, 10);
    printList(head, "After insert 10 at front");

    // Insert 20 after 10 (head)
    insertAfter(head, head, 20);
    printList(head, "After insert 20 after 10");

    // Insert 40 after 30 (last node)
    Node* node30 = head->next->next;
    insertAfter(head, node30, 40);
    printList(head, "After insert 40 after 30");
    printAddresses(head);

    cout << "\n=== Removing nodes ===\n";
    printList(head, "Before removes");

    // Remove 20 (second node, after head)
    removeAfter(head, head);
    printList(head, "After removing 20 (after head)");

    // Remove head (10)
    removeAfter(head, nullptr);
    printList(head, "After removing head (10)");

    printAddresses(head);

    // Clean up
    while (head) { Node* t = head->next; delete head; head = t; }
    cout << "\n(All nodes freed.)\n";
}
```

### Observation Table 1 — List State After Each Operation

Run the program and record the list contents printed after each labeled step:

| Step | List contents (data values in order) |
|---|---|
| Initial | |
| After insert 30 at front | |
| After insert 10 at front | |
| After insert 20 after 10 | |
| After insert 40 after 30 | |
| After removing 20 (after head) | |
| After removing head (10) | |

---

### Critical Thinking Questions — Model 1

**Q1.** Every `new Node(val)` call allocates memory on the heap. Every `delete` call frees it. What would happen to a program that builds a large linked list, removes all nodes using `removeAfter`, but forgets to call `delete` on the removed nodes?

> Your answer:

---

## Model 2: Operation Costs — Linked List vs. Array

Theory says linked lists give O(1) insert at front but O(n) access by index, while arrays give O(1) access but O(n) insert at front. This program measures those claims empirically across a range of sizes.

### Program 2 — `list_operation_costs.cpp`

```cpp
#include <iostream>
#include <vector>
#include <chrono>
#include <iomanip>
#include <numeric>
using namespace std;
using Clock = chrono::high_resolution_clock;
using us    = chrono::duration<double, micro>;

struct Node {
    int data; Node* next;
    Node(int v) : data(v), next(nullptr) {}
};

// --- Linked list operations ---
void ll_pushFront(Node*& head, int val) {
    Node* n = new Node(val);
    n->next = head; head = n;
}

int ll_getAt(Node* head, int index) {
    Node* c = head;
    for (int i = 0; i < index; i++) c = c->next;
    return c->data;
}

void ll_clear(Node*& head) {
    while (head) { Node* t = head->next; delete head; head = t; }
}

// --- Array-based list (std::vector) operations ---

template<typename Fn>
double timeUs(Fn fn, int reps = 200) {
    auto t0 = Clock::now();
    for (int i = 0; i < reps; i++) fn();
    return us(Clock::now() - t0).count() / reps;
}

int main() {
    vector<int> sizes = {1000, 5000, 10000, 50000, 100000};

    // ---- INSERT AT FRONT ----
    cout << "\n--- Insert at Front: " 
         << "Linked list pushFront vs vector insert(begin) (microseconds) ---\n\n";
    cout << setw(10) << "n"
         << setw(20) << "Linked pushFront"
         << setw(20) << "Vector insert front"
         << "\n" << string(50, '-') << "\n";

    for (int n : sizes) {
        // Linked list: push n elements to front
        double ll_time = timeUs([&]{
            Node* head = nullptr;
            for (int i = 0; i < n; i++) ll_pushFront(head, i);
            ll_clear(head);
        }, 20);

        // Vector: insert n elements at front (index 0)
        double vec_time = timeUs([&]{
            vector<int> v;
            v.reserve(n);
            for (int i = 0; i < n; i++) v.insert(v.begin(), i);
        }, 5);

        cout << setw(10) << n
             << setw(20) << fixed << setprecision(1) << ll_time
             << setw(20) << vec_time << "\n";
    }

    // ---- INSERT AT BACK ----
    cout << "\n--- Insert at Back: "
         << "Linked list (no tail) walk-to-end vs vector push_back (microseconds) ---\n\n";
    cout << setw(10) << "n"
         << setw(22) << "Linked (walk to back)"
         << setw(20) << "Vector push_back"
         << "\n" << string(52, '-') << "\n";

    for (int n : sizes) {
        // Linked list without tail pointer: walk to end each time
        double ll_back = timeUs([&]{
            Node* head = nullptr;
            for (int i = 0; i < n; i++) {
                Node* newN = new Node(i);
                if (!head) { head = newN; continue; }
                Node* c = head;
                while (c->next) c = c->next;  // walk to end — O(n)
                c->next = newN;
            }
            ll_clear(head);
        }, 5);

        double vec_back = timeUs([&]{
            vector<int> v;
            for (int i = 0; i < n; i++) v.push_back(i);
        }, 20);

        cout << setw(10) << n
             << setw(22) << fixed << setprecision(1) << ll_back
             << setw(20) << vec_back << "\n";
    }

    // ---- ACCESS BY INDEX ----
    cout << "\n--- Access by Index: "
         << "Linked list walk vs vector [] (nanoseconds, single access) ---\n\n";
    cout << setw(10) << "n"
         << setw(22) << "Linked getAt(n/2)"
         << setw(22) << "Vector v[n/2]"
         << "\n" << string(54, '-') << "\n";

    for (int n : sizes) {
        // Build both structures once
        Node* head = nullptr;
        vector<int> vec(n);
        for (int i = n-1; i >= 0; i--) ll_pushFront(head, i);
        iota(vec.begin(), vec.end(), 0);

        using ns = chrono::duration<double, nano>;
        int midIdx = n / 2;
        volatile int sink = 0;  // prevent optimization

        double ll_acc = timeUs([&]{
            sink = ll_getAt(head, midIdx);
        }, 5000);

        double vec_acc = timeUs([&]{
            sink = vec[midIdx];
        }, 500000);

        // Report in nanoseconds for readability
        cout << setw(10) << n
             << setw(22) << fixed << setprecision(3) << ll_acc * 1000
             << setw(22) << vec_acc * 1000 << "\n";

        ll_clear(head);
    }

    cout << "\n";
}
```

### Observation Table 2a — Insert at Front (μs)

| n | Linked `pushFront` | Vector `insert(begin)` |
|---|---|---|
| 1,000 | | |
| 5,000 | | |
| 10,000 | | |
| 50,000 | | |
| 100,000 | | |

### Observation Table 2b — Insert at Back (μs)

| n | Linked (walk to end) | Vector `push_back` |
|---|---|---|
| 1,000 | | |
| 5,000 | | |
| 10,000 | | |
| 50,000 | | |
| 100,000 | | |

### Observation Table 2c — Access by Index (ns, single access at n/2)

| n | Linked `getAt(n/2)` | Vector `v[n/2]` |
|---|---|---|
| 1,000 | | |
| 5,000 | | |
| 10,000 | | |
| 50,000 | | |
| 100,000 | | |

---

### Critical Thinking Questions — Model 2

**Q2.** In Table 2a, how does linked list `pushFront` time change as n grows from 1,000 to 100,000 (100×)? How does vector `insert(begin)` time change? Which operation's growth rate matches O(1) and which matches O(n)?

> Your answer:

**Q3.** In Table 2b, the linked list version walks to the end of the list before each insertion, making it O(n²) total. When n grows from 1,000 to 10,000 (10×), by approximately what factor does the linked list time increase? Does this match O(n²)?

> Your answer:

**Q4.** In Table 2c, the vector access time should be nearly the same for all values of n. The linked list access time grows with n. At n = 100,000 accessing the middle element, roughly how many times slower is the linked list than the vector? What does this tell you about why `std::vector` is preferred for random-access workloads?

> Your answer:

**Q5.** The linked list in Table 2b is slow because it has no tail pointer — it walks the whole list on every back-insertion. If you added a `tail` pointer (as in `std::list`), back insertion would be O(1). Would that make the linked list competitive with `vector::push_back` for building a list from scratch? What other factor still disadvantages the linked list?

> Your answer:

**Q6.** Based on Tables 2a, 2b, and 2c together, describe a real application or data structure where a linked list would be clearly the better choice over a vector, and justify your answer using the specific operation costs you observed.

> Your answer:

---

## Model 3: Memory Layout and the Cost of Cache Misses

One reason vector outperforms linked lists so dramatically on access is **cache locality** — the CPU loads nearby memory in bulk. Linked list nodes can be scattered anywhere in the heap. This program makes that visible by printing node addresses and measuring how memory layout affects traversal speed.

### Program 3 — `list_memory_layout.cpp`

```cpp
#include <iostream>
#include <vector>
#include <chrono>
#include <iomanip>
#include <numeric>
#include <algorithm>
#include <random>
using namespace std;

using Clock = chrono::high_resolution_clock;
using us    = chrono::duration<double, micro>;

struct Node {
    int data;
    Node* next;
    Node(int v) : data(v), next(nullptr) {}
};

void clearList(Node*& h) {
    while (h) {
        Node* t = h->next;
        delete h;
        h = t;
    }
}

// Build a RANDOMIZED (scattered) linked list
Node* buildRandomList(int n) {
    vector<Node*> nodes;
    nodes.reserve(n);

    // allocate nodes
    for (int i = 0; i < n; i++)
        nodes.push_back(new Node(i));

    // shuffle pointers to destroy locality
    random_device rd;
    mt19937 g(rd());
    shuffle(nodes.begin(), nodes.end(), g);

    // link them
    for (int i = 0; i < n - 1; i++)
        nodes[i]->next = nodes[i + 1];
    nodes[n - 1]->next = nullptr;

    return nodes[0]; // head
}

template<typename Fn>
double timeUs(Fn fn, int reps = 1000) {
    // warm-up (important!)
    for (int i = 0; i < 100; i++) fn();

    auto t0 = Clock::now();
    for (int i = 0; i < reps; i++) fn();
    return us(Clock::now() - t0).count() / reps;
}

int main() {
    cout << "\n=== Cache Locality Experiment: Vector vs Linked List ===\n\n";

    cout << setw(12) << "n"
         << setw(20) << "Vector (us)"
         << setw(22) << "Linked List (us)"
         << setw(14) << "Slowdown"
         << "\n" << string(70, '-') << "\n";

    vector<int> sizes = {100000, 500000, 1000000};

    for (int n : sizes) {
        // contiguous array
        vector<int> v(n);
        iota(v.begin(), v.end(), 0);

        // scattered linked list
        Node* ll = buildRandomList(n);

        volatile long long sink = 0;

        double vec_t = timeUs([&] {
            long long s = 0;
            for (int x : v) s += x;
            sink = s;
        }, 500);

        double ll_t = timeUs([&] {
            long long s = 0;
            for (Node* c = ll; c; c = c->next)
                s += c->data;
            sink = s;
        }, 500);

        cout << setw(12) << n
             << setw(20) << fixed << setprecision(2) << vec_t
             << setw(22) << ll_t
             << setw(13) << setprecision(1) << (ll_t / vec_t) << "x\n";

        clearList(ll);
    }

    cout << "\n";
}
```

### Observation Table 3a — Node Address Gaps

Record the first 8 node addresses and their gaps from the program output:

| Node value | Address (hex) | Gap from previous node (bytes) |
|---|---|---|
| (head) | | — |
| | | |
| | | |
| | | |
| | | |
| | | |
| | | |
| | | |

### Observation Table 3b — Traversal Speed

| n | Vector sum (μs) | Linked list sum (μs) | Slowdown |
|---|---|---|---|
| 1,000 | | | |
| 5,000 | | | |
| 10,000 | | | |

### Observation Table 3c — Node Sizes

| Type | `sizeof` (bytes) | Notes |
|---|---|---|
| `int` | | |
| Pointer (`Node*`) | | |
| `Node` (singly) | | |
| `DNode` (doubly) | | |
| 10,000 singly-linked nodes | | |
| 10,000 doubly-linked nodes | | |

---

### Critical Thinking Questions — Model 3

**Q7.** Look at the address gaps in Table 3a. Are the nodes contiguous (gap ≈ `sizeof(Node)`)? Or are they spread out? What does this tell you about how `new` allocates memory and why linked lists are "pointer-chasing" structures?

> Your answer:

**Q8.** In Table 3b, both the vector and the linked list perform the same total number of additions (n additions). Yet the linked list is significantly slower. The work is identical — why is the time different? What is the CPU doing between each addition in the linked list case that it doesn't have to do for the vector?

> Your answer:

**Q9.** Look at Table 3c. A doubly-linked node stores one extra pointer compared to a singly-linked node. How many extra bytes does that add per node? For 10,000 nodes, how many extra bytes total does that cost? Is this a meaningful difference in practice?

> Your answer:

**Q10.** The traversal slowdown in Table 3b is consistently larger than 1× — linked list traversal is slower than vector traversal even though both do O(n) work with the same loop structure. As n grows from 1,000 to 10,000, does the slowdown factor stay roughly constant, grow, or shrink? What does the trend suggest about how cache effects scale with list size?

> Your answer:

**Q11.** A cache line is typically 64 bytes. Given `sizeof(Node)` from Table 3c, how many singly-linked nodes fit in one cache line? When the CPU loads a node from memory, it loads the whole cache line. For a linked list whose nodes are scattered in memory, how many cache lines must be loaded to traverse 10,000 nodes? For a vector of 10,000 ints (each 4 bytes), how many cache lines?

> Your answer:

---

## Model 4: Doubly-Linked Traversal and the Circular List

This final program demonstrates two structural variations: doubly-linked list traversal in both directions, and a circular list where the last node points back to the first.

### Program 4 — `list_variants.cpp`

```cpp
#include <iostream>
#include <string>
using namespace std;

// --- Doubly-linked list ---
struct DNode {
    int   data;
    DNode* next;
    DNode* prev;
    DNode(int v) : data(v), next(nullptr), prev(nullptr) {}
};

DNode* buildDoubly(initializer_list<int> vals) {
    DNode* head = nullptr;
    DNode* tail = nullptr;
    for (int v : vals) {
        DNode* n = new DNode(v);
        if (!tail) { head = tail = n; }
        else { n->prev = tail; tail->next = n; tail = n; }
    }
    return head;
}

void printForward(DNode* head) {
    cout << "Forward:  ";
    for (DNode* c = head; c; c = c->next)
        cout << c->data << (c->next ? " → " : " → null\n");
}

void printBackward(DNode* head) {
    // Walk to tail first
    DNode* tail = head;
    while (tail->next) tail = tail->next;
    cout << "Backward: ";
    for (DNode* c = tail; c; c = c->prev)
        cout << c->data << (c->prev ? " → " : " → null\n");
}

void clearDoubly(DNode* h) {
    while (h) { DNode* t = h->next; delete h; h = t; }
}

// Remove a specific node — O(1) given a pointer
void removeNode(DNode*& head, DNode* node) {
    cout << "  Removing [" << node->data << "]\n";
    cout << "    Before: ";
    if (node->prev) cout << "prev=[" << node->prev->data << "] ";
    else            cout << "prev=null ";
    if (node->next) cout << "next=[" << node->next->data << "]\n";
    else            cout << "next=null\n";

    if (node->prev) node->prev->next = node->next;
    else            head = node->next;
    if (node->next) node->next->prev = node->prev;
    delete node;
}

// --- Circular singly-linked list ---
struct CNode {
    int    data;
    CNode* next;
    CNode(int v) : data(v), next(nullptr) {}
};

CNode* buildCircular(initializer_list<int> vals) {
    CNode* head = nullptr;
    CNode* tail = nullptr;
    for (int v : vals) {
        CNode* n = new CNode(v);
        if (!tail) { head = tail = n; }
        else { tail->next = n; tail = n; }
    }
    if (tail) tail->next = head;   // make it circular
    return head;
}

void printCircular(CNode* head, int steps) {
    cout << "Circular traversal (" << steps << " steps from head): ";
    CNode* c = head;
    for (int i = 0; i < steps; i++) {
        cout << c->data;
        if (i < steps - 1) cout << " → ";
        c = c->next;
    }
    cout << "\n";
}

void printCircularOnce(CNode* head) {
    if (!head) return;
    cout << "One full pass: ";
    CNode* c = head;
    do {
        cout << c->data << " ";
        c = c->next;
    } while (c != head);
    cout << "(back to head)\n";
}

int main() {
    // --- Doubly-linked list ---
    cout << "\n=== Doubly-Linked List ===\n\n";
    DNode* dl = buildDoubly({10, 20, 30, 40, 50});

    printForward(dl);
    printBackward(dl);

    cout << "\n--- Removing nodes by pointer (no traversal needed) ---\n";
    // Get pointers to specific nodes
    DNode* node30 = dl->next->next;     // third node
    DNode* node10 = dl;                 // head

    printForward(dl);
    removeNode(dl, node30);
    printForward(dl);

    removeNode(dl, node10);
    printForward(dl);

    clearDoubly(dl);

    // --- Circular list ---
    cout << "\n=== Circular Singly-Linked List ===\n\n";
    CNode* cl = buildCircular({1, 2, 3, 4, 5});

    cout << "The last node's next pointer: ";
    CNode* last = cl;
    while (last->next != cl) last = last->next;
    cout << last << " → " << last->next
         << " (same as head: " << cl << ")\n\n";

    printCircularOnce(cl);
    printCircular(cl, 12);    // traverse past the end — wraps around

    cout << "\n--- Round-robin simulation: 3 processes, 9 time slices ---\n";
    CNode* scheduler = buildCircular({1, 2, 3});
    CNode* curr = scheduler;
    for (int tick = 1; tick <= 9; tick++) {
        cout << "  Tick " << tick << ": Process " << curr->data << "\n";
        curr = curr->next;
    }

    // Clean up circular list manually
    CNode* start = scheduler;
    CNode* c = start;
    do { CNode* t = c->next; delete c; c = t; } while (c != start);

    cout << "\n";
}
```

### Observation Table 4 — Doubly-Linked Remove

Record the list state printed after each removal:

| Step | List contents (forward direction) |
|---|---|
| Initial | 10 → 20 → 30 → 40 → 50 |
| After removing 30 | |
| After removing 10 (head) | |

---

### Critical Thinking Questions — Model 4

**Q12.** Look at the `removeNode` function for the doubly-linked list. It contains no loop — just four pointer assignments. But in a singly-linked list, removing a given node requires finding the node *before* it first (a loop). What specific pointer in `DNode` enables the O(1) removal without a traversal?

> Your answer:

**Q13.** The `removeNode` function has two `if` conditions — one checking `node->prev` and one checking `node->next`. What are the two special cases these conditions handle, and what would go wrong if either check were removed?

> Your answer:

**Q14.** Look at the circular list output for `printCircular(cl, 12)` — it prints 12 values from a 5-element list, wrapping around past the end. How many complete loops around the list does 12 steps represent? Write the sequence of values you observed.

> Your answer:

**Q15.** The circular list program ends with a special cleanup loop: `do { CNode* t = c->next; delete c; c = t; } while (c != start)`. Why can't you just use the standard `while (head) { ... head = head->next; }` cleanup loop that works for non-circular lists? What would happen if you tried?

> Your answer:

**Q16.** The round-robin scheduler output shows 3 processes cycling across 9 ticks. What real-world OS feature does this demonstrate? If a fourth process joined mid-execution, what change to the circular list would allow it to participate starting from the next tick?

> Your answer:

---


## Submission Instructions

Create a new **public** Github Repository called `cs242`, upload your local `cs242` folder there including all answers from this lab.

Each answer should be 2-5 sentences that demonstrate your understanding of the concepts through the lens of the exercises you ran.

Email the GitHub repository web link to me at `chike.abuah@wallawalla.edu`

*If you're concerned about privacy* 

You can make a **private** Github Repo and add me as a collaborator, my username is `abuach`.

Congrats, you're done with the third lab!

---