

## Changes to DealExpandedPanel.tsx - History & Action Items Tables

### 1. History Table Updates

**a. Column reorder: Move "Type" before "Time"**
- Current: Checkbox | Changes | By | Time | Type | Actions
- New: No | Changes | By | Type | Time | Actions

**b. Type column: Show text label instead of color dot**
- Replace the colored dot + Tooltip with a plain text `<span>` showing the action type (e.g., "Note", "Call", "Update")
- Use `capitalize` on the text, styled as `text-[10px] text-muted-foreground`

**c. "By" column: Make username fully visible**
- Remove `max-w-[80px] truncate` from the By cell
- Use `whitespace-nowrap` so names don't wrap

**d. Remove Delete action, replace dropdown with eye icon only**
- Remove the entire `DropdownMenu` in the actions column
- Replace with a simple `Button` (ghost, icon-only) with `Eye` icon that calls `setDetailLogId(log.id)`
- Also remove the bulk delete bar (selectedLogIds bulk actions), checkbox column, and `handleDeleteLog`/`handleBulkDeleteLogs` references since we're recording history and shouldn't allow deletion

### 2. Add Type Filter Dropdown to History Section

The type filter dropdown already exists in the header (lines 458-472) with options: All, Note, Call, Meeting, Email, System. This matches the uploaded image exactly -- no changes needed here.

### 3. Add Sortable Column Headers to History Table

- Make "Changes", "By", "Type", and "Time" column headers clickable for sorting
- Add state: `historySortField` (string, default `'created_at'`) and `historySortDirection` (`'asc' | 'desc'`, default `'desc'`)
- Show sort direction icons (ArrowUpDown / ArrowUp / ArrowDown) similar to ActionItemsTable
- Sorting logic:
  - **Changes**: Sort alphabetically by the change summary text
  - **By**: Sort alphabetically by display name
  - **Type**: Sort alphabetically by action type
  - **Time**: Sort by `created_at` timestamp (already exists, just wire to header click)
- Remove the standalone sort toggle button from the History section header (lines 474-486) since sorting moves to column headers

### 4. Add Sortable Column Headers to Action Items Table

- Add state: `actionItemSortField` (string, default `'due_date'`) and `actionItemSortDirection` (`'asc' | 'desc'`, default `'asc'`)
- Make "Task", "Assigned To", "Due Date", "Status", "Priority" headers clickable
- Show sort direction icons matching ActionItemsTable module pattern
- Update `sortedActionItems` to use the new field/direction state
- Remove the standalone sort toggle button from the Action Items section header (lines 634-645)

### Technical Details

**File:** `src/components/DealExpandedPanel.tsx`

**State changes:**
- Replace `historySortDirection` with `historySortField` + `historySortDirection` (field-based sorting)
- Replace `actionSortBy` with `actionItemSortField` + `actionItemSortDirection`
- Remove `selectedLogIds` state (no more history selection/deletion)

**History header (lines 534-546) -- new order:**
```
Checkbox removed | Changes (sortable) | By (sortable) | Type (sortable, text) | Time (sortable) | Eye icon column
```

**Action Items header (lines 684-698) -- add sort icons:**
```
Checkbox | Task (sortable) | Assigned To (sortable) | Due Date (sortable) | Status (sortable) | Priority (sortable) | Module | Actions
```

**Removals:**
- `handleDeleteLog` function (line 267-271)
- `handleBulkDeleteLogs` function (line 273-279)
- `selectedLogIds` state and related toggle functions (lines 163, 298-311)
- Bulk delete bar in history (lines 515-522)
- History row checkbox cells
- History actions dropdown (lines 594-612), replaced with simple eye icon button

**Sort icon helper** (reuse pattern from ActionItemsTable):
```typescript
const getHistorySortIcon = (field: string) => {
  if (historySortField !== field) return <ArrowUpDown className="w-3 h-3 text-muted-foreground/60" />;
  return historySortDirection === 'asc' 
    ? <ArrowUp className="w-3 h-3 text-foreground" /> 
    : <ArrowDown className="w-3 h-3 text-foreground" />;
};
```

