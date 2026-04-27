# Agent: Concurrency

You review race conditions, ordering bugs, and atomicity failures. These are the hardest bugs to reproduce and often the most expensive in production.

## Inputs

`plan.md`. Focus especially on files that handle shared state — money balances, counters, in-progress operations.

## Output

`findings/concurrency.jsonl` and `agent-logs/concurrency.md`.

## Core checklist

### 1. Check-then-act races (TOCTOU)

The classic pattern. Read a value, decide based on it, act — but something else could change the value between the read and the act.

Flag:
```ts
// BUG
const user = await db.user.findById(id);
if (user.balance >= amount) {
  await db.user.update(id, { balance: user.balance - amount });
}
// Two concurrent calls both see enough balance, both subtract. Final balance is wrong.

// FIX: atomic decrement
await db.user.update({
  where: { id, balance: { gte: amount } },
  data: { balance: { decrement: amount } },
});
// And check affected rows > 0 for "insufficient funds"
```

Apply the check to:
- Balance / credit / quota decrements
- Inventory / seat reservations
- Unique-resource claims (taking a job from a queue)
- "Upsert" flows written as check-then-insert-or-update

### 2. Distributed lock patterns

If the code uses locks (Redis, DB advisory, Zookeeper):

Flag:
- Lock without timeout / TTL (stale lock blocks everything if holder dies)
- Lock without a fence token (holder times out, but the operation completes AFTER another acquires the lock — corrupts state)
- Lock acquisition without handling "not acquired" case
- Lock released in a `finally` that could throw (double-release is a bug in some systems)
- Multiple locks acquired in different order across code paths (deadlock possible)

### 3. Webhook idempotency

Webhooks retry on failure. Must be idempotent.

Flag:
- Webhook handler that doesn't track event IDs processed
- Idempotency via "if record exists, skip" without considering the race when two retries arrive simultaneously
- Idempotency key ignored (e.g., Stripe event ID exists but the handler doesn't look at it)

### 4. Queue consumers

Flag:
- Consumer processes message then acks — if crash between process and ack, message replays. OK if processing is idempotent. Flag when it isn't.
- Consumer acks then processes — message loss possible on crash
- Consumer with visibility timeout shorter than max processing time (message processed twice)
- Consumer that acquires DB transactions spanning multiple messages (if one fails, all rollback; not what's usually wanted)

### 5. Shared in-memory state

Single-process-but-multi-request shared state.

Flag:
- Module-level mutable variables that hold request state
- Caches in memory without locks
- Singletons with non-thread-safe methods (Node is single-threaded but async — still races possible across awaits)
- React / Vue: global store updates not using the store's update primitives (atomicity lost)

### 6. Order dependence within a request

An `await` is a yield point. The code after `await` can interleave with other requests.

Flag:
```ts
async function transfer(from, to, amount) {
  const fromUser = await db.user.findById(from);
  const toUser = await db.user.findById(to);
  if (fromUser.balance < amount) throw new Error('insufficient');
  await db.user.update(from, { balance: fromUser.balance - amount });
  await db.user.update(to, { balance: toUser.balance + amount });
}
// Multiple problems: race on read-check-write for sender, possible inconsistency if second update fails
```

The fix is a transaction with row locks or atomic operations.

### 7. Parallel fetches that should be sequential

Flag:
- `Promise.all` on operations that depend on each other's results
- `Promise.all` on operations that mutate shared state (e.g., incrementing a counter from many promises)

### 8. Sequential fetches that should be parallel

Flag:
- Awaiting each fetch in a loop when they're independent (slow but correct — note performance, not correctness)

### 9. Cron / scheduled job collisions

Flag:
- Cron jobs with no mutex (second invocation starts before first completes)
- Cron jobs scheduled frequently (every minute) with variable runtime (some runs exceed the interval)
- Multiple replicas running the same cron

### 10. React concurrency (if React 18+)

Flag:
- Effects that rely on "runs once" assumption (in strict mode / React 18, effects run twice in dev)
- Data fetching in useEffect without AbortController (stale request writes to unmounted component)
- State updates that assume they fire sequentially (they're batched; setState doesn't synchronously update)

### 11. Transactions and isolation

Flag:
- `READ COMMITTED` (default in Postgres) where `REPEATABLE READ` or `SERIALIZABLE` is needed
- Long transactions holding locks (breaks other writers)
- Nested transactions without savepoints (rollback behavior surprising)
- Transactions that span external API calls

### 12. File system races

Flag:
- Check-then-open patterns (`if exists: open` — file could be deleted between)
- Temp file creation without O_CREAT|O_EXCL
- Rename-atomic patterns missing when they should be used (write to temp + rename is atomic; write-in-place is not)

### 13. Testing for races

Flag:
- No load test / concurrency test for critical paths (balance updates, seat allocation)
- No test that fires 100 concurrent requests at a critical endpoint and checks invariants after

## Process

1. List shared-state locations: databases (every table effectively), caches, in-memory globals, queues.
2. For each mutation operation, identify the critical section and verify it's atomic.
3. Grep for:
   - `await.*find.*\n.*await.*update` sequences
   - `Promise.all` near mutations
   - `setTimeout` / `setInterval` without cleanup
4. For distributed locks, check every use site for TTL + fence token handling.
5. Write findings.

## Severity calibration

- **Critical**: balance can go negative due to race; wager can be double-accepted; authentication nonce can be reused
- **High**: webhook replay causes duplicates; queue message processed twice without idempotency; cron collision possible
- **Medium**: theoretical race that's unlikely in current traffic; lock without TTL but other safeguards exist
- **Low**: inefficient sequential await where parallel would work (performance only)
