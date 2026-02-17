---
name: optimize
description: Analyzes code for performance issues including slow algorithms, memory leaks, unnecessary re-renders, database query problems, and resource bottlenecks. Use when the user says "this is slow", "optimize this", "performance issue", "why is this taking so long?", or "make this faster".
allowed-tools: Read, Grep, Glob, Bash
---

# Performance Optimization Skill

When analyzing code for performance, follow this structured process:

## 1. Understand What's Slow
- Ask or determine: what operation is slow? (page load, API response, build time, query, render)
- What's the current performance? (response time, load time, memory usage)
- What's the expected/acceptable performance?
- How much data is involved? (10 rows vs 10 million rows changes everything)

## 2. Algorithm & Data Structure Analysis
- Identify time complexity of key operations (O(n), O(n²), O(n log n), etc.)
- Look for nested loops over large datasets
- Check if a different data structure would help:
  - Array lookups that should be Map/Set/Object for O(1) access
  - Linear searches that should use binary search or indexing
  - Repeated array filtering that should be pre-grouped
- Look for unnecessary sorting or repeated work

Flag pattern:
```
// 🔴 BAD — O(n²): nested loop for lookups
users.forEach(user => {
  const order = orders.find(o => o.userId === user.id);
});

// ✅ GOOD — O(n): pre-index with Map
const orderMap = new Map(orders.map(o => [o.userId, o]));
users.forEach(user => {
  const order = orderMap.get(user.id);
});
```

## 3. Database & Query Performance
- **N+1 queries**: Loading related data inside a loop instead of batch loading
- **Missing indexes**: Queries filtering/sorting on unindexed columns
- **Over-fetching**: SELECT * when only a few columns are needed
- **Missing pagination**: Loading entire tables into memory
- **Unoptimized joins**: Joining large tables without proper conditions
- **Missing connection pooling**: Opening new connections per request

Flag pattern:
```
// 🔴 BAD — N+1: one query per user
for (const user of users) {
  const posts = await db.query('SELECT * FROM posts WHERE user_id = ?', [user.id]);
}

// ✅ GOOD — single batch query
const posts = await db.query('SELECT * FROM posts WHERE user_id IN (?)', [userIds]);
const postsByUser = groupBy(posts, 'user_id');
```

## 4. Memory Analysis
- **Memory leaks**: Event listeners not cleaned up, intervals not cleared, subscriptions not unsubscribed
- **Large object retention**: Holding references to data no longer needed
- **Unbounded caches**: Caches that grow forever without eviction
- **String concatenation in loops**: Use array join or StringBuilder instead
- **Loading entire files into memory**: Stream large files instead

Flag pattern:
```
// 🔴 BAD — memory leak: listener never removed
useEffect(() => {
  window.addEventListener('resize', handleResize);
}, []);

// ✅ GOOD — cleanup on unmount
useEffect(() => {
  window.addEventListener('resize', handleResize);
  return () => window.removeEventListener('resize', handleResize);
}, []);
```

## 5. Frontend Performance (if applicable)
- **Unnecessary re-renders**: Missing React.memo, useMemo, useCallback
- **Large bundle size**: Importing entire libraries when only one function is needed
- **Missing code splitting**: Large pages that should be lazy loaded
- **Unoptimized images**: Missing compression, wrong format, no lazy loading
- **Layout thrashing**: Reading and writing DOM properties in a loop
- **Missing virtualization**: Rendering thousands of list items instead of virtualizing
- **Blocking main thread**: Heavy computation not offloaded to Web Worker

Flag pattern:
```
// 🔴 BAD — re-computes on every render
const sorted = items.sort((a, b) => a.name.localeCompare(b.name));

// ✅ GOOD — memoized
const sorted = useMemo(
  () => [...items].sort((a, b) => a.name.localeCompare(b.name)),
  [items]
);
```

## 6. API & Network Performance
- **Missing caching**: Responses that could be cached (HTTP headers, Redis, in-memory)
- **No request batching**: Multiple small requests that could be combined
- **Missing compression**: Large JSON responses without gzip/brotli
- **Synchronous operations**: Blocking calls that could be parallelized
- **No pagination**: Returning unbounded result sets
- **Missing timeouts**: External calls without timeout limits

Flag pattern:
```
// 🔴 BAD — sequential when independent
const users = await fetchUsers();
const products = await fetchProducts();
const orders = await fetchOrders();

// ✅ GOOD — parallel execution
const [users, products, orders] = await Promise.all([
  fetchUsers(),
  fetchProducts(),
  fetchOrders(),
]);
```

## 7. Concurrency & Async
- **Uncontrolled parallelism**: Firing 10,000 requests at once instead of batching
- **Missing debounce/throttle**: Search inputs firing on every keystroke
- **Blocking event loop**: CPU-heavy sync operations in Node.js
- **Missing queue**: Tasks that should be queued and processed in background

## 8. Build & Tooling (if applicable)
- Slow builds from unnecessary transpilation
- Missing tree shaking
- Duplicated dependencies in bundle
- Missing caching in CI/CD pipeline

## Output Format

For each issue found:

**[IMPACT] Category — File:Line**
- **Problem**: What's slow and why
- **Current complexity**: O(?) or estimated impact
- **Suggested fix**: How to improve it
- **Expected improvement**: What performance gain to expect
```
// before (slow)
...

// after (fast)
...
```

Impact levels:
- 🔴 **HIGH** — Causes noticeable slowdown or crashes at scale. Fix first.
- 🟡 **MEDIUM** — Degrades performance under load. Fix soon.
- 🟢 **LOW** — Minor inefficiency. Optimize when convenient.

## Summary

End every analysis with:
1. **Biggest bottleneck** — The single highest-impact issue
2. **Quick wins** — Changes that take <30 minutes and give noticeable improvement
3. **Estimated improvement** — What overall performance gain to expect after fixes
4. **Measurement plan** — How to verify the improvements (specific metrics to track, tools to use)
