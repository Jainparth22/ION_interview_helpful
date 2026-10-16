
**The ones most likely to come up**

- **LRU cache:** hashmap plus doubly linked list. The map gives O(1) lookup, and the list gives O(1) move-to-front and evict-from-back. She should be able to say why a singly linked list or an array doesn't work.
- **LFU cache:** hashmap plus frequency buckets (or a min-heap, at O(log n)).
- **TTL expiry:** a min-heap keyed on expiry time, or lazy expiry on access.
- **Undo/redo:** two stacks. Each action is a command object. Executing a new action clears the redo stack. A bounded deque caps the history size.
- **Text editor internals:** gap buffer, rope, or piece table, so inserting in the middle of a large document isn't O(n). Collaborative editing uses OT or CRDT.
- **Browser back/forward:** two stacks, or a doubly linked list.
- **Autocomplete and search suggestions:** trie.
- **Job scheduler and task queue:** priority queue (heap), which is what Celery and Redis-backed queues conceptually are.
- **Top-K, leaderboards, trending:** a min-heap of size k, or a Redis sorted set (skip list underneath).
- **Rate limiter:** a deque for a sliding window log, or a counter for a fixed window or token bucket.
- **Movie booking:** a hashmap from seat ID to status, plus a held-seats entry with a TTL. An interval tree or sorted structure works for show-timing conflicts.
- **Social graph, maps, and routing:** a graph with BFS for shortest hops and Dijkstra for weighted paths.
- **Dependency resolution and build order:** a DAG with topological sort. Deadlock detection is cycle detection on a wait-for graph.
- **Database indexes:** B+ tree, with LSM trees in write-heavy stores.
- **Membership checks at scale:** a bloom filter, which allows false positives but never false negatives.
- **Distributing keys across cache servers:** consistent hashing on a hash ring.
- **Network connectivity and Kruskal:** union-find.
- **Parentheses matching, expression evaluation, and the call stack:** a stack.
- **OS:** a ready queue for scheduling, a priority queue for priority scheduling, and page tables as multi-level trees or hash tables.

**How to drill it**

1. Cover the right-hand answers and have her say the structure for each use case out loud. She'll learn it faster by recalling it than by rereading it.
2. For the top five (LRU, undo/redo, autocomplete, scheduler, rate limiter), she should be able to name one alternative and why it loses.
3. She should list every data structure her own project uses, explicitly or inside Redis, Celery, or the DB, and why. This pairs well with the project deep dive.
4. For LRU specifically, she should be able to code it in C++ in 10 minutes, since it's both a DSA question and a design question.

If she's short on time, the first priority is LRU, undo/redo, trie, heap/priority queue, and graph. Those five cover most of the likely questions.


Here's caching in the form an interviewer will probe it. I'm assuming you want the "add caching to your existing project" question plus the data-structure side, since both came up.

## The answer structure for "how would you add caching to your project"

1. **Find the bottleneck first.** Don't say "I'd add Redis" straight away. Say which endpoints are read-heavy, slow, or hit the DB repeatedly with the same result.
2. **Pick what to cache.** Data that is read often, changes rarely, and is expensive to compute: dashboards, listings, user profiles, aggregate queries, computed results. Don't cache highly personalized or fast-changing data, or anything where stale data is dangerous (payments, seat availability at the moment of booking).
3. **Pick the pattern** (below), the key design, the TTL, and the invalidation strategy.
4. **Say how you'd measure it:** hit ratio, latency before and after, DB load.

## Caching patterns

- **Cache-aside (lazy loading):** the app checks the cache; on a miss it reads the DB and then fills the cache. This is the default answer and the one Redis is mostly used for. Downside: the first request is slow, and data can go stale.
- **Write-through:** every write goes to the cache and the DB together. Reads are always fresh, but writes are slower and you may cache data nobody reads.
- **Write-back (write-behind):** write to the cache and flush to the DB later. Very fast writes, but data is lost if the cache dies before flushing.
- **Write-around:** write to the DB only and let reads populate the cache. Good when written data is rarely read soon after.

## Eviction policies

LRU (most common), LFU (keeps frequently used items, resists one-off scans), FIFO, and TTL-based expiry. Redis lets you configure this with `maxmemory-policy`, for example `allkeys-lru`.

## Invalidation, the hard part

- **TTL:** simple, accepts a bounded staleness window.
- **Explicit delete on write:** update the DB, then delete the cache key. Delete rather than update, because updating can race with a concurrent reader and leave stale data.
- **Versioned keys:** put a version in the key (`user:42:v3`) so old entries just age out.

## Problems worth naming

- **Cache stampede:** a hot key expires and many requests hit the DB at once. Fixes: a lock so only one request rebuilds the value, TTL jitter, or refreshing early.
- **Cache penetration:** requests for keys that don't exist always miss. Fixes: cache the "not found" result briefly, or use a bloom filter.
- **Cache avalanche:** many keys expire together. Fix: randomize TTLs.
- **Consistency:** the cache and the DB can disagree. Say plainly that you're trading some freshness for speed, and say how much staleness is acceptable for the data you're caching.

## Key design and Redis specifics

- Keys should be namespaced and deterministic: `product:{id}`, `user:{id}:orders:page:2`. For query results, include the parameters in the key.
- Use Redis strings for serialized objects, hashes for objects with fields you update individually, and sorted sets for leaderboards.
- Redis is single-threaded for commands, in-memory, and optionally persistent. Local in-process caches (like Python's `lru_cache`) are faster but aren't shared across instances and can't be invalidated centrally, which is why you move to a distributed cache once you run multiple servers.

## LRU in C++ (expect to be asked to code this)

```cpp
class LRUCache {
    int cap;
    list<pair<int,int>> dll;                       // front = most recent
    unordered_map<int, list<pair<int,int>>::iterator> mp;
public:
    LRUCache(int c) : cap(c) {}
    int get(int key) {
        if (!mp.count(key)) return -1;
        dll.splice(dll.begin(), dll, mp[key]);     // move to front, O(1)
        return mp[key]->second;
    }
    void put(int key, int val) {
        if (mp.count(key)) {
            mp[key]->second = val;
            dll.splice(dll.begin(), dll, mp[key]);
            return;
        }
        if ((int)dll.size() == cap) {
            mp.erase(dll.back().first);
            dll.pop_back();
        }
        dll.emplace_front(key, val);
        mp[key] = dll.begin();
    }
};
```

She should be able to explain why it's a hashmap plus a doubly linked list: the map gives O(1) lookup, and the list gives O(1) reordering and eviction. For the design framing, she can add a mutex for thread safety.

## Twenty-minute practice

1. Five minutes: say the cache-aside flow out loud for one real endpoint in her project.
2. Five minutes: recite the three problems (stampede, penetration, avalanche) with one fix each.
3. Ten minutes: write the LRU from memory, then explain the complexity.

If you tell me her project, I can write out a specific caching answer for it: what she'd cache, the keys, the TTLs, and the invalidation plan.