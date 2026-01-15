

--------------------
Here's a concise, generic prompt template:

---

## Task: Add Table of Contents with Back Links to Markdown File**

For a given Markdown file with `##` headings:

1. **Create TOC at top:** Add `# Table of Contents` with nested list of `[[#Heading Name]]` links for all `##` headings
2. **Add back links:** After each `##` heading, add `[[#Table of Contents|Back to TOC]]` on new line

**Format:**
```
# Table of Contents
- [[#Table of Contents]]
  - [[#Heading 1]]
  - [[#Heading 2]]
---

## Heading 1
[[#Table of Contents|Back to TOC]]
Content...

## Heading 2  
[[#Table of Contents|Back to TOC]]
Content...
```

Use `replace_in_file` for each modification, matching exact heading text including special characters.

---

This is a reusable template that can be applied to any Markdown file needing TOC + back links.

-------------------------


Here's a generic prompt for cleaning any LLM chat transcript:

---

## Task: Clean LLM Chat Transcript for Knowledge Base**

**Input:** Verbatim copy of web-based chat with LLM containing valuable content mixed with conversational artifacts.

**Goal:** Extract clean, readable documentation by removing all chat-specific elements while preserving the core valuable information.

**Extraneous Elements to Remove:**

1. **Dates/Timestamps that look like LLM UI dialogue timestamps**
   - Look for date formats at places where it looks irrelevant: "Jan 14", "2024-01-15", "9:07 AM", "12:30 PM", etc.

2. **LLM Internal Processing Notes**
   - Meta-commentary: "Analyzed user requirements...", "Deconstructed the approach..."
   - Thinking aloud: "The user is asking about...", "This involves..."
   - Response planning: "Given the context, I should provide..."

3. **User Style Instructions & Preferences**
   - Response style guidelines
   - User preference configurations
   - Behavioral instructions for the LLM

4. **Conversational/Transitional Text**
   - Question fragments and follow-ups
   - Transitions: "Let's proceed to...", "Moving on to..."
   - Acknowledgments: "Good question", "Understood", "Ready for next"
   - Status messages: "Awaiting your response", "Processing..."

5. **Chat Interface Artifacts**
   - User/Assistant labels
   - System messages
   - UI elements and formatting artifacts

6. **Error Messages as Conversation**
   - Raw error outputs presented as dialogue

**Content to Preserve:**

1. **All Code Blocks** - Every fenced code block (```) exactly as-is
2. **Technical Explanations** - Clean explanations and instructions
3. **Question Headers** - Main technical or informational queries
4. **Inline Comments** - Comments within code blocks
5. **Structured Data** - Tables, lists, structured information
6. **Core Information** - The actual valuable content being discussed

**Output Structure:**

- **Organize Logically:** Group related content into coherent sections
- **Use Clean Markdown:** Headers, lists, code blocks without artifacts
- **Remove Duplication:** Eliminate redundant explanations
- **Maintain Technical Accuracy:** Preserve all technical details and examples
- **Create Flow:** Ensure content reads as continuous documentation
- **Be technical**: Avoid vacuous statements and pleasantries like "Sorting a list is crucial for having list organized"

------------


**Generic Example Transformation:**

**BEFORE:**
```
How to sort a list?

Jan 15

Analyzed sorting requirements for user query...

python
```python
list.sort()
```

That works. Now how to reverse sort?

Jan 15
```

**AFTER:**
```
## Sorting Lists

### Basic Sort
```python
list.sort()
```

### Reverse Sort
```python
list.sort(reverse=True)
sorted(list, reverse=True)
```
```

**Validation:**
- ✅ No dates or timestamps
- ✅ No meta-commentary about conversation
- ✅ No conversational transitions
- ✅ All code blocks preserved intact
- ✅ Technical content flows logically
- ✅ Organized into coherent sections

---

This generic prompt can clean any LLM chat transcript regardless of the technical domain or content type.