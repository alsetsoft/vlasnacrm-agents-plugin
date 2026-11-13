---
name: document-lines
description: Documents with a header and line items — orders, invoices, receipts, quotes, protocols, sample batches, any parent record edited together with child rows. Atomic save, computed columns, line editor, totals, snapshot semantics, "create from this document". Load together with crud-form whenever a form edits a parent with child rows.
---

# Documents with line items

Builds on `crud-form` (routes, shell, actions, submit lock). This skill adds what is specific to header + lines.

## Data model

- `<doc>` (header) + `<doc>_items` (`doc_id` FK `on delete cascade`, `position` int for order, `item_id`, `qty`, `price`, optional `discount`, `vat_rate`, `comment`).
- **Computed columns live in the DB**: `amount numeric generated always as (round(qty * price * (1 - coalesce(discount,0)/100), 2)) stored` (or a view). The client **never sends** generated columns in insert/update (Postgres rejects them).
- Header totals (`sum`, `vat_sum`, line count) are computed by the save function, not trusted from the client.
- Invariants as constraints: `qty >= 0`, `price >= 0` CHECKs; cross-row rules (e.g. "returned ≤ received") in the save function under `FOR UPDATE`.

## Saving — one transaction

```sql
create or replace function save_<doc>(p_id uuid, p_header jsonb, p_items jsonb)
returns uuid language plpgsql security invoker as $$
declare v_id uuid;
begin
  if p_id is null then
    insert into <doc> (...) select ... from jsonb_to_record(p_header) as h(...) returning id into v_id;
  else
    update <doc> set ... from jsonb_to_record(p_header) as h(...) where id = p_id returning id into v_id;
    if v_id is null then raise exception 'not found' using errcode = 'P0002'; end if;
  end if;
  -- replace lines inside the same transaction (or diff by line id to keep audit history)
  delete from <doc>_items where doc_id = v_id;
  insert into <doc>_items (doc_id, position, item_id, qty, price, discount)
  select v_id, ord, (x->>'item_id')::uuid, (x->>'qty')::numeric, (x->>'price')::numeric, coalesce((x->>'discount')::numeric, 0)
  from jsonb_array_elements(p_items) with ordinality as t(x, ord);
  update <doc> set sum = (select coalesce(sum(amount), 0) from <doc>_items where doc_id = v_id) where id = v_id;
  return v_id;
end $$;
```

- The client calls **only** this RPC: `supabase.rpc("save_<doc>", { p_id, p_header, p_items })`. Never `delete` + `insert` lines from the browser — a failure in between leaves a document without lines, and a double click duplicates them.
- Validation failures raise with a SQL state the UI maps to a message (23514 check, 23505 duplicate, 42501 forbidden); the whole save rolls back.

## Editor

- Header in react-hook-form; lines in local state (`useState<Line[]>` or `useFieldArray`) with a client-side `key` per row (not the index).
- Dense table: header row 32px (`h-8`), cells `py-1`, inputs `h-7`; columns: # · item · qty · (availability, if the domain has stock) · price · discount · amount · remove.
- **Add a line** — see *Lines toolbar* below: one shared add field, always in the row ABOVE the lines table, never a trailing row under it.
- Numeric cells follow the non-negative rule (parse/clamp on change, block `-`/`e`).
- Price on add = the item's price converted to the document currency once (shared helper); changing it later is the user's choice.
- **Totals** next to the comment, computed by ONE helper that mirrors the DB formula exactly (same rounding), e.g. `lineAmount(qty, price, discount)`; show subtotal · discount · tax · total · line count.

## Lines toolbar — one place, one field, one label (every document)

Users add lines dozens of times a day; if the field moves between documents (above the table here, a trailing row there) or changes its label, they hunt for it every time. Real QA bug: the same ERP had the add field above the table in 7 documents and below it in 4, with 5 different placeholders ("Add product (search by name)…", "Add item — enter name or SKU…") and barcode mentioned in only one.

- **One shared component** for the row above every lines table (e.g. `DocumentItemsToolbar`) with fixed slots, so the order cannot drift:
  `[start: line count · bulk line actions (reprice…)] [add: the add field] ··· [end, pushed right: tax toggles, document-level line actions]`.
  Root `flex flex-wrap items-center gap-2`; `end` has `ml-auto`. The lines table follows directly below.
- **One add field** (e.g. `AddItemField`) wrapping the shared entity picker: `h-8`, `min-w-64 max-w-xl flex-1`, value always empty (pick = append a line, field resets), optional "+" on its left for an empty/manual line where the document allows one.
- **Why above, not a trailing row:** it stays visible at any number of lines (no scrolling to the bottom of a 50-line document), sits next to the other line actions, and works the same on an empty document.
- **One label constant** shared by all documents, saying *line/item* (it adds products, services and bundles alike — not "product") and listing **every** search criterion the picker really supports, e.g. `"Add item — name, code, SKU or barcode…"` (localized). Never a per-document placeholder.
- **One search behaviour:** the same server search everywhere (all words across name/code/SKU + exact barcode). If the label lists a criterion, every document's search must honour it — test it.
- **Empty state** of the lines table points to the field: `"No items yet — add them with the «Add item» field above"` (one constant).
- **Variants keep the slot:** a restricted add (e.g. "Add a line from the source receipt" on a return) renders in the same `add` slot; read-only documents omit `add` but keep the toolbar.
- **Narrow windows:** the toolbar wraps, the field never shrinks below 16rem, no horizontal page scroll at 1024 px.
- **Regression test** (e2e, no save): for every document route — the field is visible above the empty-state text, has the shared label, finds one known item by name, code, SKU and barcode, picking adds a line, no horizontal overflow at 1024 px.

## Snapshot semantics

- A saved document is a historical snapshot: opening it never recalculates prices, rates, discounts or taxes from current reference data. Recalculation only through an explicit action ("Apply discount to all lines", "Refresh prices") that the user sees in the line values before saving.

## Related actions

- "Create X from this document" (invoice from order, receipt from purchase order, return from sale): **save first** if dirty → separate action lock → navigate to `/x/new?from=<id>`; the new form prefills from a server function that returns only what is still open (e.g. ordered − already shipped), and shows it as an unsaved draft the user can trim.
- Tabs on the card (Lines · Related documents · History) use `forceMount` (or keep state above the tabs): unmounting the lines editor loses unsaved edits.

## Audit (reviewer)

| Check | Severity |
|-------|----------|
| `.delete()` on `*_items` followed by `.insert(` in the same handler (client-side line replace) | BLOCKER |
| a generated/computed column (`amount`, totals) present in an insert/update payload | BLOCKER |
| totals computed in the client with a formula different from the DB's (rounding, discount) | WARN |
| line editor keyed by array index | WARN |
| opening a saved document recalculates prices/discounts/rates in an effect | BLOCKER |
| "create from" action without saving the dirty source first, or without its own lock | BLOCKER |
| tabs unmount the lines editor (no `forceMount`/lifted state) | WARN |
| a line-item entity picker that differs from the project's shared picker for that entity | WARN |
| add-line field below the lines table (trailing row) or outside the shared lines toolbar | BLOCKER |
| add-line placeholder written inline instead of the shared constant, or different across documents | BLOCKER |
| add-line label lists a search criterion (barcode, code…) the picker does not search, or omits one it does | WARN |
| lines-table empty state that points "below" / differs from the shared constant | WARN |
