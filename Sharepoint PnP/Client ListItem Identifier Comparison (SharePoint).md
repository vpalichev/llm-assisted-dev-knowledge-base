# Table of Contents

- [[#SharePoint Identifier Comparison: ID vs GUID vs UniqueId]]
- [[#Recommendation for Datomic Integration]]
- [[#ID (List Item ID)]]
- [[#GUID (List Item GUID)]]
- [[#UniqueId (File/Folder GUID)]]
- [[#Comparison Matrix]]
- [[#Practical Demonstration]]

---

## SharePoint Identifier Comparison: ID vs GUID vs UniqueId

[[#Table of Contents|Back to TOC]]

These three fields in `FieldValues` identify distinct entities within the SharePoint object model and have different scopes, persistence characteristics, and intended use cases.

Remeber that type returned by Get-PnPList is Microsoft.SharePoint.Client.ListItem

## Recommendation for Datomic Integration

[[#Table of Contents|Back to TOC]]

For storing SharePoint file references in Datomic, **UniqueId** is the correct identifier because:

1. It survives file renames and moves within the site collection
2. It identifies the actual file content, not the metadata wrapper
3. It aligns with the semantic meaning of "this document" rather than "this list entry"
4. SharePoint provides direct file retrieval via `GetFileById(guid)`

Store `GUID` (list item) only if you need to track metadata history separately from the file itself, such as when list column values and their change history are primary concerns.

---

## ID (List Item ID)

[[#Table of Contents|Back to TOC]]

**Type:** `Int32`

**Definition:** A sequential integer identifier assigned to each list item within a specific list or library. Represents the item's position in the list's internal item collection.

**Scope of Uniqueness:** Unique only within the containing list/library. Two items in different lists may share the same ID value.

**Persistence Behavior:**

- Remains stable while the item stays in the same list
- **Changes** when the file is moved to a different library
- **Changes** when the file is copied (new item receives new ID)
- Never reused within the same list after deletion

**Primary Use Cases:**

- List item retrieval via SharePoint REST API or CSOM
- Internal list operations and queries
- Human-readable reference within a single library context

**Access Pattern:**

```powershell
$item = Get-PnPListItem -List "Documents" -Id 42
$item.Id                      # Returns Int32
$item.FieldValues["ID"]       # Returns Int32
```

**API Usage:**

```
GET _api/web/lists/getbytitle('Documents')/items(42)
```

---

## GUID (List Item GUID)

[[#Table of Contents|Back to TOC]]

**Type:** `System.Guid`

**Definition:** A globally unique identifier assigned to the list item entity itself. Identifies the list item record in SharePoint's content database, independent of the file it may reference.

**Scope of Uniqueness:** Globally unique across the SharePoint farm/tenant.

**Persistence Behavior:**

- Remains stable while the item exists in the same list
- **Changes** when the file is moved to a different library (new list item created)
- **Changes** when the file is copied (new list item with new GUID)
- Represents the list item metadata container, not the binary file content

**Primary Use Cases:**

- Identifying list item metadata records
- Cross-list-item references within workflows
- Content database operations

**Access Pattern:**

```powershell
$item.FieldValues["GUID"]     # Returns System.Guid
```

**Important Distinction:** This GUID identifies the _list item wrapper_, not the underlying file. A file's metadata (columns, permissions at item level, version history) is associated with this GUID.

---

## UniqueId (File/Folder GUID)

[[#Table of Contents|Back to TOC]]

**Type:** `System.Guid`

**Definition:** A globally unique identifier assigned to the file or folder object itself within SharePoint's content storage layer. Represents the binary content entity rather than its list item metadata wrapper.

**Scope of Uniqueness:** Globally unique within the site collection. Guaranteed unique within the SharePoint tenant.

**Persistence Behavior:**

- **Remains stable** when file is renamed
- **Remains stable** when file is moved within the same site collection
- **Changes** when file is copied (new file entity created)
- **Changes** when file is moved to a different site collection
- Tied to the file blob, not the metadata wrapper

**Primary Use Cases:**

- Permanent file references in external systems (databases, integrations)
- File retrieval regardless of current path or name
- Cross-system synchronization where path may change
- Datomic entity identity (recommended approach)

**Access Pattern:**

```powershell
# Via list item
$item.FieldValues["UniqueId"]    # Returns System.Guid

# Via file object
$file = Get-PnPFile -Url "Shared Documents/file.docx"
$file.UniqueId                   # Returns System.Guid
```

**API Usage:**

```
GET _api/web/GetFileById('01234567-89ab-cdef-0123-456789abcdef')
```

---

## Comparison Matrix

[[#Table of Contents|Back to TOC]]

| Property                              | ID      | GUID   | UniqueId        |
| ------------------------------------- | ------- | ------ | --------------- |
| **Type**                              | Int32   | Guid   | Guid            |
| **Identifies**                        | List item position | List item entity | File/folder blob |
| **Scope**                             | Within list | Global | Site collection |
| **Survives rename**                   | Yes     | Yes    | Yes             |
| **Survives move (same library)**      | Yes     | Yes    | Yes             |
| **Survives move (different library)** | No      | No     | Yes             |
| **Survives move (different site)**    | No      | No     | No              |
| **Survives copy**                     | No      | No     | No              |

---

## Practical Demonstration

[[#Table of Contents|Back to TOC]]

```powershell
$item = Get-PnPListItem -List "Documents" -Id 5

[PSCustomObject]@{
    "ID (List Item Position)"    = $item.FieldValues["ID"]
    "GUID (List Item Entity)"    = $item.FieldValues["GUID"]
    "UniqueId (File Entity)"     = $item.FieldValues["UniqueId"]
} | Format-List
```

**Sample Output:**

```
ID (List Item Position) : 5
GUID (List Item Entity) : a1b2c3d4-e5f6-7890-abcd-ef1234567890
UniqueId (File Entity)  : 98765432-10fe-dcba-0987-654321fedcba
```