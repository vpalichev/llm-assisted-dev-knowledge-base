This code builds two lookup hashtables for quick O(1) access to SharePoint files during the sync process:

**`$spFilesByUniqueId`** - Maps `UniqueId` (GUID) → file object

- Used to detect if the _same file_ (by identity) still exists at a path
- A file's UniqueId stays constant even if renamed

**`$spFilesByName`** - Maps `FileLeafRef` (filename) → file object

- Used to check if _any_ file exists at a given filename path
- Helps detect when a different file now occupies the same name

These lookups enable the sync logic to distinguish between:

1. **Same file, same version** → skip
2. **Same file, new version** → update
3. **Different file at same path** → supersede (old file replaced by new one with same name)
4. **File gone from path** → mark deleted

Without these hashtables, the code would need nested loops with O(n²) complexity to find matching files.