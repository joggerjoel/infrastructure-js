## TanStack Grid Guidelines – Performance & Best Practices

This document provides guidelines for building performant TanStack grids. It documents lessons learned from real-world performance issues and establishes patterns that all grids must follow.

**Related Documentation**:
- **[TANSTACK_GRID_SPEC.md](../specs/TANSTACK_GRID_SPEC.md)** — Complete schema-driven architecture specification
- Module 6 of the spec covers the **Schema-Driven Grid Architecture** which handles most of these concerns automatically

---

### Recommended Approach: Schema-Driven Grids

For new grids, use the **schema-driven architecture** defined in `TANSTACK_GRID_SPEC.md`. The `SchemaGrid` component automatically handles:
- Column memoization
- Data memoization  
- Filter state management
- Loading/error states
- Pagination
- Performance optimizations

```tsx
// New grids should use this pattern:
import { SchemaGrid } from '@/components/SchemaGrid';
import { myGridSchema } from '@/schemas/my-grid.schema';

export function MyGridPage() {
  return <SchemaGrid schema={myGridSchema} />;
}
```

If you **must** build a custom grid (rare), follow the rules below.

---

### Common Performance Issues

**Problem:** Grids can appear frozen or unresponsive when:
- `columns` and `data` arrays are recreated on every render
- React Table recalculates internal models unnecessarily
- Large result sets are processed without proper memoization
- Filters trigger new fetches while non-memoized config causes recalculation

**Solution:** Always memoize heavy inputs (`columns`, `data`) and implement proper loading/error handling.

### Rules for all data grids

#### 1. Always memoize `columns`

**DO:**

```ts
const columnHelper = createColumnHelper<MyRow>();

const columns = useMemo(
  () => [
    columnHelper.accessor('id', { header: 'ID' }),
    // …other columns…
  ],
  [] // or [deps] if you truly need dynamic behavior
);
```

**DON'T:**

```ts
// ❌ New array every render – will cause table recalculation
const columns = [
  columnHelper.accessor('id', { header: 'ID' }),
  // …
];
```

If a column relies on callbacks/mutations, either:

- put those in `useCallback` and include them in the `useMemo` deps, or
- keep columns static and call mutations from the cell using values passed via `row.original`.

#### 2. Memoize table `data`

Even when the backend paginates, the in-memory `data` array can still be large or change often.

**DO:**

```ts
const tableData = useMemo(
  () => queryData?.data ?? [],
  [queryData?.data]
);

const table = useReactTable({
  data: tableData,
  columns,
  // …
});
```

Avoid passing `queryData?.data || []` inline into `useReactTable` – that creates a **new array each render**, invalidating React Table's memoization.

#### 3. Treat filters as part of the query key (but stable)

- Keep filters in a single `useState` object (e.g. `{ match_type, review_type, disable_type, search }`).
- Use that object in the React Query `queryKey`:

```ts
const [filters, setFilters] = useState<Filters>({
  status: 'all',
  category: 'active',
  search: '',
});

const query = useQuery({
  queryKey: ['myGrid', pageIndex, pageSize, filters],
  queryFn: ({ signal }) => service.getGridData(/* … */, filters, [], signal),
  retry: false,
  keepPreviousData: false,
});
```

Rules:

- Only call `setFilters` when the user **actually changes** a value.
- Don't recreate filter objects unnecessarily in renders – derive them from state, not from props each time.

#### 4. Show clear loading and error states

- Use `isLoading` **and** `isFetching` to avoid rendering stale rows during a refetch:

```tsx
{isLoading || isFetching ? (
  <tr><td colSpan={columns.length}>Loading…</td></tr>
) : isError ? (
  <tr><td colSpan={columns.length}>Failed to load: {String(error?.message)}</td></tr>
) : (
  rows.map(/* render rows */)
)}
```

- Don't try to "keep rendering old rows" while a heavy refetch is in flight unless you've profiled it and know it's safe.

#### 5. Be careful with heavy per-cell work

Components like `StringDiffMatcher` that do text normalization, tokenization, or complex layout per row can easily become a bottleneck.

Guidelines:

- Keep **expensive string/array work out of render** where possible (pre-compute on the backend or in a memoized selector).
- If you must do it in the cell, keep it:
  - purely functional (no side effects),
  - as simple as possible, and
  - reused across grids (so optimizations benefit multiple screens).

#### 6. Use fetch timeouts defensively

Network/proxy issues can cause fetches to hang longer than expected. Wrap service calls with a **timeout guard** so they don't appear to freeze the page forever.

Example pattern (simplified):

```ts
async function fetchWithTimeout(url: string, signal?: AbortSignal, ms = 30000) {
  const controller = new AbortController();
  const timeoutId = setTimeout(() => controller.abort(), ms);

  const finalSignal = signal ?? controller.signal;

  try {
    const res = await fetch(url, { signal: finalSignal });
    return res;
  } finally {
    clearTimeout(timeoutId);
  }
}
```

Then call that from your service layer so **all grids** benefit from the same behavior.

#### 7. Add planned debug hooks (but keep them cheap)

When building new grid pages:

- Log **one line** when the query runs:

```ts
console.log('[MyGrid] Fetching', { pageIndex, pageSize, filters });
```

- Optionally log **row counts** on render:

```ts
console.log('[MyGrid] Rendering table', {
  rowCount: table.getRowModel().rows.length,
  total: data?.total,
});
```

If you ever see:

- row counts wildly higher than `pageSize`, or
- logs firing hundreds of times per second,

stop and profile before shipping.

---

### Checklist for new grids

Before merging any new grid/table page:

#### Preferred: Schema-Driven Grid (see TANSTACK_GRID_SPEC.md)

- [ ] **Use `SchemaGrid` component** — handles all performance concerns automatically
- [ ] Grid schema defined in `schemas/{grid-name}.schema.ts`
- [ ] Schema validated (all required fields, valid column types)
- [ ] Custom cell renderers registered in `renderers/` (if needed)
- [ ] AI-friendly: flat properties, explicit defaults, searchable IDs

#### If Custom Grid Required (document justification):

- [ ] `columns` is wrapped in `useMemo`
- [ ] `data` passed to `useReactTable` is memoized
- [ ] Filters live in a dedicated state object and are part of the React Query `queryKey`
- [ ] Loading/error states are explicit and don't attempt to render massive stale datasets
- [ ] Heavy per-row work is minimized or pre-computed
- [ ] Service layer either:
  - uses a shared `fetchWithTimeout`, or
  - has a consciously chosen timeout/abort strategy
- [ ] Console debug logging (if any) is **bounded and cheap**

Following this checklist should prevent regressions where a single filter change makes the **entire SPA appear frozen**.

---

### Migration to Schema-Driven Grids

Existing custom grids should be migrated to the schema-driven architecture when:
- Major changes are needed
- Performance issues arise
- The grid needs new features (bulk actions, modals, export)
- AI needs to make frequent modifications

See `TANSTACK_GRID_SPEC.md` Section 6.9 for migration steps and Section 6.12 for AI-centric design principles.
