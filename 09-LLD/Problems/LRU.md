# LRU Cache

## 1. Requirements

**Functional**

- Support `get(key)` — return the value if present, else a sentinel (e.g. -1 / null); accessing a key marks it as most recently used
- Support `put(key, value)` — insert or update a key's value; marks it as most recently used
- Fixed **capacity** — when inserting a new key would exceed capacity, evict the **least recently used** entry first
- Updating an existing key's value should also refresh its recency (counts as a "use")
- Support querying current size / capacity (optional, for debugging/monitoring)

**Non-Functional**

- Both `get()` and `put()` must run in **O(1)** time — this is the entire point of the problem; an O(n) scan-based "least recently used" search defeats the purpose
- Thread-safety if used in a concurrent/multi-threaded context (e.g. shared cache in a server)
- Memory overhead should be reasonable — avoid duplicating large values across auxiliary structures
- Should be generic/reusable across different key/value types

---

## 2. Actors

- **Client** — calls `get()`/`put()` on the cache (e.g. an application layer caching expensive DB/API results)
- **LRUCache** — the component maintaining bounded storage with eviction policy

---

## 3. Use Cases

- Client puts a new key-value pair into the cache
- Cache is full → evict the least recently used entry before inserting the new one
- Client gets a value by key → cache hit updates recency; cache miss returns not-found
- Client updates an existing key's value → recency refreshed, no eviction needed
- Client repeatedly accesses the same "hot" keys → those keys should never be evicted while colder keys are

---

## 4. Classes

- `Node` — doubly linked list node holding `key`, `value`, `prev`, `next` (key is stored in the node too, not just the value, so eviction can remove the corresponding hash map entry in O(1))
- `LRUCache` — owns:
    - a `HashMap<Key, Node>` for O(1) lookup of any node by key
    - a doubly linked list (with dummy `head`/`tail` sentinels) ordered by recency — most-recently-used near `head`, least-recently-used near `tail`
    - a `capacity` field
- (Optional) `EvictionPolicy` (interface) — abstracts the eviction strategy so LRU could be swapped for LFU/FIFO without changing the cache's public API

---

## 5. Relationships

- `LRUCache` **has-a** `HashMap<Key, Node>` (composition) — O(1) existence/lookup
- `LRUCache` **has-a** doubly linked list of `Node` (composition) — O(1) reordering (move-to-front) and O(1) eviction (remove from tail)
- The **same `Node` objects** are referenced by both the hash map and the linked list — this dual-structure design is _the_ key insight of the whole problem: hash map gives O(1) lookup, linked list gives O(1) reordering; neither alone achieves both

---

## 6. Design Patterns

- Not primarily a "GoF pattern" problem — it's a **data-structure composition** problem. The core technique is combining a **hash map + doubly linked list** to get O(1) on both lookup and recency-reordering
- **Strategy** (optional extension) — if you want to support swappable eviction policies (LRU vs LFU vs FIFO) behind a common interface, wrap the eviction logic in an `EvictionPolicy` strategy so `LRUCache`'s public API doesn't change when the policy does
- **Decorator** (optional extension) — a thread-safe wrapper (`SynchronizedLRUCache`) around a plain `LRUCache` is a natural Decorator if you want to keep the core implementation lock-free and add synchronization as an separate concern

---

## 7. Class Diagram

```text
┌─────────────────────────┐
│        LRUCache           │
│ - capacity: int              │
│ - map: HashMap<K, Node>        │
│ - head: Node (dummy, MRU end)   │
│ - tail: Node (dummy, LRU end)     │
│ + get(key): V                       │
│ + put(key, value)                     │
│ - moveToFront(node)                     │
│ - removeNode(node)                        │
│ - evictLRU()                                │
└───────────┬─────────────┘
            │ manages
            ▼
┌─────────────────────────┐
│           Node              │
│ - key: K                       │
│ - value: V                       │
│ - prev: Node                       │
│ - next: Node                         │
└─────────────────────────┘

head.next ⇄ [MRU] ⇄ ... ⇄ [LRU] ⇄ tail.prev
map[key] → Node   (O(1) lookup, same Node objects as in the list)
```

---

## 8. Sequence Flow

```text
Client → LRUCache.put("A", 1)   [capacity = 2]
  │  map doesn't contain "A" → create Node("A",1), insert at front (after head)
  │  map["A"] = node
  ▼
Client → LRUCache.put("B", 2)
  │  create Node("B",2), insert at front → list: head ⇄ B ⇄ A ⇄ tail
  ▼
Client → LRUCache.get("A")
  │  map contains "A" → found, moveToFront(A) → list: head ⇄ A ⇄ B ⇄ tail
  │  return 1
  ▼
Client → LRUCache.put("C", 3)
  │  map.size == capacity (2) → evictLRU():
  │      lru = tail.prev  (= B, the least recently used)
  │      removeNode(lru); map.remove("B")
  │  create Node("C",3), insert at front → list: head ⇄ C ⇄ A ⇄ tail
  │  map["C"] = node
```

---

## 9. Code

```cpp
#include <iostream>
#include <unordered_map>

template <typename K, typename V>
class LRUCache {
    struct Node {
        K key;
        V value;
        Node* prev;
        Node* next;
        Node(K k, V v) : key(k), value(v), prev(nullptr), next(nullptr) {}
    };

    int capacity;
    std::unordered_map<K, Node*> map;
    Node* head; // dummy — head->next is MRU
    Node* tail; // dummy — tail->prev is LRU

    void removeNode(Node* node) {
        node->prev->next = node->next;
        node->next->prev = node->prev;
    }

    void insertAtFront(Node* node) {
        node->next = head->next;
        node->prev = head;
        head->next->prev = node;
        head->next = node;
    }

    void moveToFront(Node* node) {
        removeNode(node);
        insertAtFront(node);
    }

public:
    explicit LRUCache(int cap) : capacity(cap) {
        head = new Node(K(), V());
        tail = new Node(K(), V());
        head->next = tail;
        tail->prev = head;
    }

    ~LRUCache() {
        Node* curr = head;
        while (curr) {
            Node* next = curr->next;
            delete curr;
            curr = next;
        }
    }

    // Returns true and sets outValue if found; false if miss
    bool get(const K& key, V& outValue) {
        auto it = map.find(key);
        if (it == map.end()) return false;

        Node* node = it->second;
        moveToFront(node); // access refreshes recency
        outValue = node->value;
        return true;
    }

    void put(const K& key, const V& value) {
        auto it = map.find(key);
        if (it != map.end()) {
            // key exists — update value, refresh recency
            it->second->value = value;
            moveToFront(it->second);
            return;
        }

        if ((int)map.size() >= capacity) {
            // evict LRU (node just before tail)
            Node* lru = tail->prev;
            removeNode(lru);
            map.erase(lru->key);
            delete lru;
        }

        Node* node = new Node(key, value);
        insertAtFront(node);
        map[key] = node;
    }
};

int main() {
    LRUCache<std::string, int> cache(2);

    cache.put("A", 1);
    cache.put("B", 2);

    int val;
    if (cache.get("A", val)) std::cout << "A = " << val << "\n"; // hit, A now MRU

    cache.put("C", 3); // evicts B (LRU, since A was just accessed)

    if (!cache.get("B", val)) std::cout << "B evicted (miss)\n";
    if (cache.get("C", val)) std::cout << "C = " << val << "\n";

    return 0;
}
```

---

## 10. Trade-offs

- **HashMap + doubly linked list (the standard answer) vs a simpler HashMap + ordered structure like `LinkedHashMap`** — languages with a built-in ordered hash map (Java's `LinkedHashMap` with access-order mode, Python's `OrderedDict`/`dict` since 3.7) let you implement LRU in a few lines by relying on the library's internal doubly-linked-list-backed implementation; doing it manually (as above) is what's expected in an interview to prove you understand _why_ it's O(1), not just that a library does it
- **Storing the key inside `Node`** — seems redundant (key is already the hash map's key) but is necessary: when evicting via the tail pointer, you only have a `Node*`, and need the key to remove the corresponding entry from the hash map in O(1) — without it you'd need an O(n) reverse lookup
- **Dummy head/tail sentinels vs null-checked boundaries** — sentinels eliminate special-casing "is this the first/last real node?" in `insertAtFront`/`removeNode`, trading a small constant memory overhead (2 extra nodes) for simpler, bug-resistant pointer logic
- **Single global lock (coarse-grained) vs sharded/striped locking for thread safety** — a single mutex around `get`/`put` is simple and correct but serializes all cache access; sharding the cache into N independently-locked sub-caches (hash key → shard) reduces contention at the cost of a slightly imperfect global LRU ordering (each shard is only locally LRU-accurate)

---

## 11. Interview Questions

- **Why does this need both a hash map and a linked list — why not just one or the other?** A hash map alone gives O(1) lookup but has no notion of order, so finding the least-recently-used item would require an O(n) scan. A linked list alone gives O(1) reordering (move a node to the front) but O(n) lookup to find a node by key in the first place. Combining them — hash map for lookup, linked list for order — gets O(1) on both operations, which is the entire insight of this problem.
- **Why store the key inside the `Node`, not just the value?** Because eviction only has direct access to the LRU `Node` (via `tail->prev`), not its key — without storing the key in the node, removing the corresponding hash map entry during eviction would require an O(n) reverse lookup, breaking the O(1) guarantee.
- **How would you make this thread-safe?** Wrap `get`/`put` in a mutex (simplest, fully correct, but serializes access) — or, for higher throughput, shard the cache into N independent LRU caches, each with its own lock, and route each key to a shard via a hash function; this trades perfectly-global LRU ordering for reduced lock contention.
- **How would you extend this to LFU (Least Frequently Used) instead of LRU?** LFU needs to track access _frequency_, not just recency, which changes the data structure: typically a hash map of key→node plus a second hash map of frequency→doubly-linked-list-of-nodes-at-that-frequency, plus tracking the current minimum frequency — more complex, but same core idea of combining hash maps with linked lists for O(1) operations.
- **How would you add TTL (time-to-live) expiration on top of LRU?** Store an expiry timestamp in each `Node`; check it on `get()` and treat an expired entry as a miss (optionally removing it lazily at that point) — for proactive cleanup rather than only lazy/on-access expiry, a background sweep or a min-heap ordered by expiry time would be needed alongside the existing structures.
- **What's the time and space complexity?** Both `get()` and `put()` are O(1) time (hash map lookup + constant pointer operations). Space is O(capacity) — one `Node` and one hash map entry per cached item, plus the two dummy sentinel nodes.

---

## Related Notes

- [[strategy]]
- [[decorator]]