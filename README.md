```java
# Bug Report: Order Execution Report Cross-Contamination

**Severity:** Critical  
**Component:** Order Processor - Order Management  
**Date:** October 19, 2025  
**Status:** ✅ Resolved

---

## Problem

Orders received incorrect execution reports due to shared mutable object references in cache.

**Symptoms:**
- Order `O7SCDDT00002H1P`: Received 0 execution reports (expected 3)
- Order `O7SCDDT00001H1P`: Received 6 execution reports (expected 3)
- Cached order data mutated after storage, causing wrong clOrdID lookups

---

## Root Cause

`OrderManager.addOrder()` stored direct references to mutable `DefaultNewOrderSingle` objects instead of copies.

```java
// ❌ BROKEN CODE
orderManifestCache.computeIfAbsent(
    orderKey, k -> new OrderManifest(defaultNewOrderSingle, null));
```

**Flow:**
1. Order stored in cache (by reference)
2. Same reference sent to exchange adapter
3. Exchange processing mutated the object's `clOrdID`
4. Cache now contained corrupted data
5. Lookups returned wrong order information

---

## Solution

Implemented defensive copying to isolate cached objects from external mutations.

```java
// ✅ FIXED CODE
var orderCopy = createOrderCopy(defaultNewOrderSingle);
orderManifestCache.computeIfAbsent(
    orderKey, k -> new OrderManifest(orderCopy, null));
```

**Why it works:**
- Cache stores independent copy
- External code can mutate original without affecting cache
- Each order maintains data integrity throughout lifecycle

---

## Impact

**Business:**
- Critical: Execution reports mismatched to wrong orders
- Risk: Incorrect trade confirmations and settlements

**Technical:**
- FIX gateway sent reports to wrong sessions
- Order state management corrupted
- Test failures from missing/duplicate reports

**Performance:**
- Memory: ~500 bytes per order (negligible)
- CPU: <1ms per order (acceptable)

---

## Verification

**Test Results:**
```
✓ Order1 receives exactly 3 execution reports with correct clOrdID
✓ Order2 receives exactly 3 execution reports with correct clOrdID  
✓ No cross-contamination between orders
✓ Cache data integrity maintained
```

---

## Key Takeaways

1. **Always use defensive copying** when storing mutable objects in caches
2. **Prefer immutable objects** to eliminate this class of bugs entirely
3. **Add diagnostic logging** before/after cache operations
4. **Test with mutation scenarios** to catch shared reference bugs

---

## Future Actions

- [ ] Refactor `DefaultNewOrderSingle` to be immutable (2-3 days)
- [ ] Add static analysis rules for mutable object storage
- [ ] Document defensive copying patterns for team
- [ ] Create standard test utilities for object isolation verification

---

**Fixed By:** Development Team  
**Reviewed By:** [Pending]
```
