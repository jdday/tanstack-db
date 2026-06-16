# Ownership / subset-merge bug (on-demand) — findings

Branch: `backlog-refetch-revert`. This is a **different** bug from `ISSUE.md`
(which covers the `toArray` + `autoIndex` crash). Repro: `src/App.tsx` here —
flat per-filter live queries in nested components (no `toArray`), matching a
real app where each column re-derives from the collection.

## Symptom
Two `useLiveQuery`s read the **same on-demand collection** with different
`where`s (e.g. `eq(projectId, p1)` for a project column, `isNull(projectId)`
for the backlog). Move a row between them (`update` sets `projectId = null`):
it lands in the backlog **optimistically**, then after the refetch it **reverts**
to the project and disappears from the backlog.

Key point: the refetch responses are **correct** (`isNull → i1,i4`,
`parents → i2,i3`) — so it's not stale data. It's a **merge** bug.

## A/B (this repro, flat, no toArray)
| | optimistic | after refetch |
|---|---|---|
| published `query-db-collection` | ✅ moves to backlog | ❌ reverts |
| with the fix below | ✅ moves to backlog | ✅ stays |

## Root cause
`@tanstack/query-db-collection` → `applySuccessfulResult` in
`packages/query-db-collection/src/query.ts` (compiled: `dist/esm/query.js`).

Two loops run when a subset's result is applied:
- **previously-owned rows**: already writes an `update` when the value changed. ✅
- **newly-owned rows**: adds ownership but only writes an **`insert` if the row
  is absent** — it **never updates** an already-synced row's value. ❌

So when a row migrates subsets, the new subset returns it with the new value,
but it's already in the collection (stale value from the old subset) → the new
value is dropped. Then the old subset drops ownership but the new subset still
owns it (`removeRow` returns `false`), so the row stays stuck at the stale value.

Ownership is a `Set` (`rowToQueries`) — a row **can** belong to multiple
subsets; that's intended. The bug is purely the missing value update.

## The fix (PR-ready, ~+3 lines)
In the newly-owned-rows loop, mirror what the previously-owned loop already does:

```diff
 newItemsMap.forEach((newItem, key) => {
   const owners = getPersistedOwners(key);
   if (!owners.has(hashedQueryKey)) { owners.add(hashedQueryKey); setPersistedOwners(key, owners); }
   addRow(key, hashedQueryKey);
-  if (!currentSyncedItems.has(key)) {
-    write({ type: `insert`, value: newItem });
-  }
+  const existingItem = currentSyncedItems.get(key);
+  if (!existingItem) {
+    write({ type: `insert`, value: newItem });
+  } else if (!previouslyOwnedRows.has(key) && !deepEquals(existingItem, newItem)) {
+    write({ type: `update`, value: newItem });
+  }
 });
```
`deepEquals` is already imported in that file. Validated by patching the
installed `dist/esm/query.js` — the revert disappears (table above).

Open question for maintainers: there is a parallel `shouldUsePersistedBaseline`
path in the same function that likely needs the same treatment; and the
cross-subset "two subsets return the same row with different values" conflict
policy is theirs to bless (not hit by a migrating row, which is only ever in one
subset at a time).

## Second, separate bug (not fixed by the above)
With the ownership fix, the row correctly stays in the backlog — but if the
parent grouping uses **`toArray`**, the row also **lingers in the project**
(the `toArray`/includes engine doesn't drop a row when its grouping key
changes). That's a `@tanstack/db` bug (same family as `ISSUE.md`: silent dupes
without `autoIndex`, hard crash with it). A nested-component / flat structure
avoids it.

## Contributing the fix to TanStack DB
- Node `24.8.0` (`.nvmrc`), pnpm `11.1.0` (`corepack enable && corepack prepare pnpm@11.1.0 --activate`).
- `git clone` your fork of `TanStack/db`, `pnpm install`, `pnpm build`.
- Test the package: `pnpm --filter @tanstack/query-db-collection test` (vitest).
- Edit source `packages/query-db-collection/src/query.ts` (not `dist`).
- Add a test: two on-demand subsets, move a row between them, assert it stays.
- `pnpm changeset` (patch, `@tanstack/query-db-collection`), commit it.
- PR against `TanStack/db:main`.