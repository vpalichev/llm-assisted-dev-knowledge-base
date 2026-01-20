# SharePoint CAML queries and the 5000-item threshold: A technical guide

The SharePoint Online list view threshold blocks CAML queries not based on results returned, but on **rows scanned** during query execution. A query returning 100 items can still fail if SharePoint must scan more than 5,000 rows to identify those results. This fundamental distinction explains why `Get-PnPListItem -PageSize` works without filtering but fails with CAML queries on large lists—and reveals the precise requirements for threshold compliance.

## Why the threshold exists and what it actually measures

Microsoft implemented the 5,000-item threshold to prevent SQL Server from escalating row-level locks to table-level locks. When database operations exceed 5,000 rows, SQL Server temporarily locks the entire table rather than individual rows, degrading performance for all users on shared tenants. The threshold cannot be modified in SharePoint Online.

**The critical technical point**: Threshold enforcement occurs during query execution, not result aggregation. SharePoint must determine which items match your filter criteria before applying pagination. If your CAML `<Where>` clause filters on a non-indexed column, SharePoint scans every row in the list to evaluate matches—triggering the threshold even if only 50 items ultimately match your criteria.

Consider a list with 60,000 items. A query filtering on an unindexed "Status" column for "Active" items must scan all 60,000 rows to identify matches. Even with `<RowLimit>100</RowLimit>`, the scan exceeds the threshold and fails. However, the same query filtering on an **indexed** Status column uses the index to locate matching rows directly, avoiding a full table scan.

## Get-PnPListItem behaves fundamentally differently with and without -Query

When you execute `Get-PnPListItem -List "Documents" -PageSize 1000` without a `-Query` parameter, the cmdlet internally constructs a minimal CAML query:

```xml
<View>
  <RowLimit Paged="TRUE">1000</RowLimit>
</View>
```

This query implicitly orders by the ID column—which is **always indexed**—and uses `ListItemCollectionPosition` for pagination. Each page request retrieves the next 1,000 items by ID, never scanning more than 1,000 rows per request. The threshold is never approached.

When you add `-Query` with a CAML filter, the behavior changes fundamentally. The CAML `<Where>` clause is evaluated **server-side before pagination occurs**. SharePoint must identify all matching items before it can paginate results. If your filter uses unindexed columns or returns more than 5,000 matches, the query fails before pagination even begins.

GitHub issues #651 and #1550 in the PnP PowerShell repository confirm this behavior: `-PageSize` does not prevent threshold errors when `-Query` is specified because the filtering operation precedes the pagination operation in SharePoint's query execution pipeline.

## Requirements for CAML queries to work on large lists

Threshold-compliant CAML queries must satisfy three mandatory conditions:

**First, the initial filter must use an indexed column.** The leftmost condition in your `<Where>` clause must reference a column with an index. SharePoint evaluates filters left-to-right in nested `<And>` blocks, and the first filter determines whether an index can be used.

**Second, that first filter must reduce results to under 5,000 items.** Even with an indexed column, if your filter matches 10,000 items, the threshold blocks the query. The index accelerates the scan but doesn't eliminate it.

**Third, indexes must be created before the list exceeds 5,000 items.** Creating an index requires accessing all items, which itself can trigger the threshold on large lists. Plan indexing strategy before lists grow large.

The `<RowLimit Paged="TRUE">` element enables pagination but does **not** prevent threshold errors. It only controls how many results are returned per page after filtering succeeds.

## Filter order in CAML directly affects threshold behavior

The position of filter conditions within nested `<And>` elements determines query execution order. Consider this structure:

```xml
<Where>
  <And>
    <Geq>
      <FieldRef Name='Modified' />  <!-- FIRST: Must be indexed -->
      <Value Type='DateTime'>2025-01-01T00:00:00Z</Value>
    </Geq>
    <Eq>
      <FieldRef Name='Status' />  <!-- SECOND: Evaluated after first filter -->
      <Value Type='Choice'>Active</Value>
    </Eq>
  </And>
</Where>
```

SharePoint evaluates the Modified filter first. If Modified is indexed and the date range returns fewer than 5,000 items, the query succeeds regardless of whether Status is indexed. Reversing the order—placing the unindexed Status filter first—causes a full table scan and threshold error.

**Range queries on the same indexed column combine as a single filter.** You can use both `<Geq>` and `<Leq>` on the Modified column within one `<And>` block, and SharePoint treats this as one indexed filter operation:

```xml
<And>
  <Geq>
    <FieldRef Name='Modified' />
    <Value Type='DateTime'>2025-01-01T00:00:00Z</Value>
  </Geq>
  <Leq>
    <FieldRef Name='Modified' />
    <Value Type='DateTime'>2025-01-31T23:59:59Z</Value>
  </Leq>
</And>
```

## Correct approach for querying lists with 60,000+ items

For large lists requiring date-based filtering, implement this pattern with PnP PowerShell:

**Option 1: Batched date ranges with indexed column**

Index your date column (Modified, Created, or custom), then query in date ranges small enough to return under 5,000 items per batch:

```powershell
$Query = @"
<View Scope='RecursiveAll'>
  <Query>
    <Where>
      <And>
        <Geq>
          <FieldRef Name='Modified' />
          <Value Type='DateTime' IncludeTimeValue='TRUE'>2025-01-01T00:00:00Z</Value>
        </Geq>
        <Leq>
          <FieldRef Name='Modified' />
          <Value Type='DateTime' IncludeTimeValue='TRUE'>2025-01-15T23:59:59Z</Value>
        </Leq>
      </And>
    </Where>
    <OrderBy>
      <FieldRef Name='ID' Ascending='TRUE'/>
    </OrderBy>
  </Query>
  <RowLimit Paged="TRUE">500</RowLimit>
</View>
"@

$items = Get-PnPListItem -List "LargeList" -Query $Query
```

If each two-week batch exceeds 5,000 items, narrow the date range further.

**Option 2: Retrieve all items, filter client-side**

When server-side filtering is impractical, retrieve all items using pagination without a query, then filter locally:

```powershell
$allItems = Get-PnPListItem -List "LargeList" -PageSize 2000
$filtered = $allItems | Where-Object { 
    $_["Modified"] -ge [DateTime]"2025-01-01" -and 
    $_["Modified"] -le [DateTime]"2025-01-31" 
}
```

This approach works reliably but transfers more data and processes filtering in PowerShell.

**Option 3: ID-based range batching**

Query in ID ranges to ensure each batch stays under threshold:

```powershell
$batchSize = 4999
$currentId = 0
$results = @()

do {
    $Query = @"
    <View Scope='RecursiveAll'>
      <Query>
        <Where>
          <And>
            <Gt><FieldRef Name='ID'/><Value Type='Number'>$currentId</Value></Gt>
            <Leq><FieldRef Name='ID'/><Value Type='Number'>$($currentId + $batchSize)</Value></Leq>
          </And>
        </Where>
        <OrderBy><FieldRef Name='ID' Ascending='TRUE'/></OrderBy>
      </Query>
      <RowLimit Paged="TRUE">500</RowLimit>
    </View>
"@
    $batch = Get-PnPListItem -List "LargeList" -Query $Query
    $results += $batch
    $currentId += $batchSize
} while ($batch.Count -gt 0)
```

## Column indexing requirements and limitations

Indexable column types include: Single line of text, Choice (single value), Number, Currency, Date and Time, Yes/No, Lookup (single value), Person or Group (single value), and Managed Metadata.

**Lookup columns have a critical limitation**: While you can index lookup columns, Microsoft explicitly warns that indexed lookup columns **do not prevent threshold errors**. Use non-lookup column types as primary filters for threshold compliance.

Non-indexable types include: Multiple lines of text, multi-valued Choice/Lookup/Person columns, Calculated columns, and Hyperlink fields.

SharePoint permits **20 indexes per list**. For compound conditions, SharePoint supports compound indexes combining two columns, queryable via the `<WithIndex>` CAML element with the index GUID.

## Alternative API: RenderListDataAsStream

The `RenderListDataAsStream` method offers more flexibility with threshold constraints. Microsoft documentation indicates this API "removes the requirement of your first filter parameter to limit the result set to 5000 items or less." While still subject to performance considerations, this endpoint handles large list queries more gracefully than standard CSOM/REST approaches.

## Conclusion

The fundamental issue with `Get-PnPListItem -Query` on large lists stems from SharePoint's query execution order: CAML filters evaluate server-side before pagination applies. The `-PageSize` parameter cannot prevent threshold errors because filtering precedes pagination in the execution pipeline.

For 60,000+ item lists, success requires either indexed columns with filters narrow enough to return under 5,000 items, or client-side filtering after retrieving all items via pagination. Date range queries work when the date column is indexed and each range spans few enough items. The filter order in CAML matters—place your indexed, selective filter first in nested `<And>` blocks. Most importantly, create indexes proactively before lists grow large, as retroactive indexing may itself be blocked by the threshold.