## Excel VBA: Fixed-Width Field Formatter

Below is a VBA implementation providing a `FixedWidthFormat` function that concatenates an arbitrary number of cell values into a single fixed-width record, with per-field width, alignment, and padding specifications.

### Module code

```vba
Option Explicit

' ============================================================================
' Enumeration: Field alignment modes
' ============================================================================
Public Enum FwAlign
    fwLeft = 0      ' Data on left, padding on right  (rpad)
    fwRight = 1     ' Data on right, padding on left  (lpad)
    fwCenter = 2    ' Padding distributed on both sides
End Enum

' ============================================================================
' Core helper: Format a single value into a fixed-width field
' ----------------------------------------------------------------------------
' value      - The input value (any type; coerced to String via CStr)
' width      - Target field width in characters
' align      - Alignment mode (fwLeft / fwRight / fwCenter)
' padChar    - Single-character pad (default: space)
' truncate   - If True, values longer than width are cut; else returned as-is
' ============================================================================
Public Function PadField(ByVal value As Variant, _
                         ByVal width As Long, _
                         Optional ByVal align As FwAlign = fwLeft, _
                         Optional ByVal padChar As String = " ", _
                         Optional ByVal truncate As Boolean = True) As String

    Dim s As String
    Dim deficit As Long
    Dim leftPad As Long, rightPad As Long

    s = CStr(value)

    ' Handle oversize input
    If Len(s) >= width Then
        If truncate Then
            PadField = Left$(s, width)
        Else
            PadField = s
        End If
        Exit Function
    End If

    deficit = width - Len(s)

    Select Case align
        Case fwLeft
            PadField = s & String$(deficit, padChar)
        Case fwRight
            PadField = String$(deficit, padChar) & s
        Case fwCenter
            leftPad = deficit \ 2
            rightPad = deficit - leftPad
            PadField = String$(leftPad, padChar) & s & String$(rightPad, padChar)
    End Select
End Function

' ============================================================================
' Primary function: Build a fixed-width record from a range
' ----------------------------------------------------------------------------
' rng        - Range of source cells (single row or single column)
' widthsCSV  - Comma-separated widths, e.g. "10,12,8"
' alignsCSV  - Comma-separated alignment codes: L, R, or C (case-insensitive)
'              e.g. "L,C,R"
' separator  - String placed between fields (default: "|")
' padChar    - Pad character (default: space)
' ============================================================================
Public Function FixedWidthFormat(ByVal rng As Range, _
                                 ByVal widthsCSV As String, _
                                 ByVal alignsCSV As String, _
                                 Optional ByVal separator As String = "|", _
                                 Optional ByVal padChar As String = " ") As String

    Dim widths() As String
    Dim aligns() As String
    Dim parts() As String
    Dim cell As Range
    Dim i As Long
    Dim mode As FwAlign

    widths = Split(widthsCSV, ",")
    aligns = Split(alignsCSV, ",")

    If rng.Cells.Count <> UBound(widths) - LBound(widths) + 1 Then
        FixedWidthFormat = "#ERR: width count <> cell count"
        Exit Function
    End If
    If rng.Cells.Count <> UBound(aligns) - LBound(aligns) + 1 Then
        FixedWidthFormat = "#ERR: align count <> cell count"
        Exit Function
    End If

    ReDim parts(0 To rng.Cells.Count - 1)
    i = 0
    For Each cell In rng.Cells
        Select Case UCase$(Trim$(aligns(i)))
            Case "L": mode = fwLeft
            Case "R": mode = fwRight
            Case "C": mode = fwCenter
            Case Else: mode = fwLeft
        End Select

        parts(i) = PadField(cell.value, CLng(Trim$(widths(i))), mode, padChar)
        i = i + 1
    Next cell

    FixedWidthFormat = Join(parts, separator)
End Function
```

### Installation on Windows

1. In Excel, press `Alt`+`F11` to open the VBA editor.
2. `Insert` → `Module`.
3. Paste the code above.
4. Save the workbook as `.xlsm` (macro-enabled workbook). The `.xlsx` format strips VBA.
5. If macros are blocked, `File` → `Options` → `Trust Center` → `Trust Center Settings` → `Macro Settings` → enable macros for this workbook or its folder.

### Usage examples

Given cells `A1:C1` containing `field01`, `3`, `f4`:

|Formula|Result|
|---|---|
|`=FixedWidthFormat(A1:C1, "10,12,8", "L,C,C")`|`field01 \| 3 \| f4`|
|`=FixedWidthFormat(A1:C1, "10,12,8", "L,R,L", " \| ")`|`field01 \| 3 \| f4`|
|`=FixedWidthFormat(A1:C1, "8,8,8", "R,R,R", "", "0")`|`0field01000000030000000f4`|

The function accepts a vertical range equally (`A1:A3`), since it iterates `rng.Cells` in row-major order.

### Pitfalls and quirks specific to this environment

**Font rendering.** Fixed-width alignment is visually meaningful only in a **monospaced font** (Consolas, Courier New, Lucida Console). Excel's default Calibri is proportional; the underlying string is correctly padded, but columns will not visually align in the cell. Apply a monospaced font to the output cell.

**Cell value coercion.** `CStr(cell.value)` uses Excel's internal representation. Dates become serial numbers (e.g., `45000`), not formatted dates. To preserve the displayed format, substitute `cell.Text` for `cell.value` — note that `cell.Text` returns `"###"` if the source column is too narrow to display the value, which is an Excel display artifact, not a data issue.

**Width parsing.** `Split(widthsCSV, ",")` is locale-sensitive in a subtle way: if the user enters `"10, 12, 8"` with spaces, the `Trim$` call handles it. However, entering the widths with a semicolon (common in locales where `;` is the list separator inside formulas) will fail — the VBA `Split` delimiter is independent of Excel's formula list separator. Keep the CSV argument comma-delimited regardless of Windows regional settings.

**Array-formula behavior.** The function returns a scalar string. To apply it to many rows, enter it per row rather than as a dynamic array; `FixedWidthFormat` is not vectorized over multiple rows.

**`String$` vs `String`.** The `$`-suffixed variants (`String$`, `Left$`, `CStr` — the latter has no `$` form) return `String` directly rather than `Variant`, which is marginally faster and avoids implicit conversions. This is a Windows-VBA convention worth preserving.

**Writing to a text file.** If the goal is to export fixed-width records to a `.txt` file for a mainframe or legacy system, write with `Open ... For Output` and terminate lines with `vbCrLf` (Windows line ending). Using `vbLf` alone produces Unix line endings, which some Windows tools render as a single long line.