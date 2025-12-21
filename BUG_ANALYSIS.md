# Critical Bug Found in tempo-apps

## Bug: Unsafe Address.checksum() Return Value Check

### Location
[apps/explorer/src/lib/server/account.server.ts](apps/explorer/src/lib/server/account.server.ts#L169-L172)

### Severity
**HIGH** - This is a runtime bug that can cause crashes

### The Problem

In the `fetchTransactions` function (lines 169-172):

```typescript
const from = Address.checksum(row.from)
if (!from) throw new Error('Transaction is missing a "from" address')

const to = row.to ? Address.checksum(row.to) : null
```

### Why This Is a Bug

According to the `ox` library (which provides the `Address` module), the `Address.checksum()` function **never returns null or undefined**. It always returns a valid checksummed address string or **throws an error** if the input is invalid.

The current code assumes that `Address.checksum()` can return a falsy value (`null` or `undefined`), but it cannot. This means:

1. **The null check is dead code** - it will never catch the error condition
2. **Invalid addresses will not be caught** - if somehow an invalid address is passed, `Address.checksum()` will throw, but the try-catch is missing
3. **Runtime crash instead of graceful error** - if an invalid address comes from the database, the handler will crash instead of returning a proper error response

### Example of the Problem

```typescript
// If row.from is an invalid address like "0xinvalid":
const from = Address.checksum(row.from)  // Throws error!
if (!from) throw new Error(...)  // This line never executes - exception is unhandled

// Expected behavior:
try {
  const from = Address.checksum(row.from)
  // ... rest of code
} catch (error) {
  throw new Error(`Transaction has invalid "from" address: ${row.from}`)
}
```

### Impact

- **User-facing**: API endpoint crashes when encountering transactions with invalid addresses
- **Data reliability**: Invalid transaction data in the database will cause the entire request to fail with a 500 error instead of gracefully handling the issue
- **Maintainability**: Dead code makes the function confusing and harder to maintain

### The Fix

Remove the dead null check and wrap the `Address.checksum()` calls in a try-catch block to handle potential validation errors:

```typescript
let from: Address.Address
try {
  from = Address.checksum(row.from)
} catch {
  throw new Error(`Transaction has invalid "from" address: ${row.from}`)
}

let to: Address.Address | null = null
if (row.to) {
  try {
    to = Address.checksum(row.to)
  } catch {
    throw new Error(`Transaction has invalid "to" address: ${row.to}`)
  }
}
```

### Why This Matters for the Project

1. **Data validation**: The database might contain corrupted or invalid addresses from upstream sources
2. **User experience**: Better error messages help developers debug issues
3. **Production reliability**: Proper error handling prevents crashes
4. **Type safety**: The code should match what the API actually does

---

## Additional Context

This pattern appears in multiple places in the codebase (5 occurrences found), indicating a systematic misunderstanding of the `Address.checksum()` API.

All occurrences should be audited and fixed:
- `apps/explorer/src/lib/domain/receipt.ts` (lines 61, 81, 333)
- `apps/explorer/src/lib/server/account.server.ts` (lines 169, 172)

