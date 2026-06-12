Sometimes Outlook mangles text in the Drafts folder. Messages that arrive in the Inbox can be fixed via **Actions → Other Actions → Encoding**, but drafts are "our" messages — written by us — so Outlook offers no encoding override for them. To recover the original text, we must understand this:

**1. Original** — "Коллеги, добрый день!" in three encodings:

```
UTF-8:   D0 9A D0 BE D0 BB D0 BB D0 B5 D0 B3 D0 B8 2C 20 D0 B4 D0 BE D0 B1 D1 80 D1 8B D0 B9 20 D0 B4 D0 B5 D0 BD D1 8C 21
CP1251:  CA EE EB EB E5 E3 E8 2C 20 E4 EE E1 F0 FB E9 20 E4 E5 ED FC 21
KOI8-R:  EB CF CC CC C5 C7 C9 2C 20 C4 CF C2 D2 D9 CA 20 C4 C5 CE D8 21
```

The sender's system stored the **KOI8-R** bytes.

**2. Mangled** — those KOI8-R bytes read as CP1251:

```
EB→л  CF→П  CC→М  CC→М  C5→Е  C7→З  C9→Й  2C→,  20→sp
C4→Д  CF→П  C2→В  D2→Т  D9→Щ  CA→К  20→sp
C4→Д  C5→Е  CE→О  D8→Ш  21→!
```

Displayed: `лПММЕЗЙ, ДПВТЩК ДЕОШ!`

**3. Redemption** — take the CP1251 bytes of those displayed letters:

```
л=EB  П=CF  М=CC  М=CC  Е=C5  З=C7  Й=C9  ,=2C  sp=20
Д=C4  П=CF  В=C2  Т=D2  Щ=D9  К=CA  sp=20
Д=C4  Е=C5  О=CE  Ш=D8  !=21
→ EB CF CC CC C5 C7 C9 2C 20 C4 CF C2 D2 D9 CA 20 C4 C5 CE D8 21
```

Identical to the KOI8-R row in step 1. Reinterpret as KOI8-R → "Коллеги, добрый день!"

The fix is a no-op on bytes; only the label changes.

**How to do step 3 in practice:** paste the mangled string into an online decoder like **2cyr.com**:

- **"I see"** (input encoding) → **Windows-1251**
- **"Instead of"** (intended encoding) → **KOI8-R**

Hit decode → readable Russian appears. The tool takes each displayed letter, looks up its CP1251 byte, then prints the letter that byte represents in the KOI8-R table.