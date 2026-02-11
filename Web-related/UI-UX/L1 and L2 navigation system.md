## Navigation Positioning and Spatial Hierarchy

**Primary Navigation (L1)**: Positioned in the left sidebar panel, occupying the vertical edge of the application viewport. This constitutes the primary navigation container with fixed positioning relative to the viewport boundary.

**Secondary Navigation (L2)**: Positioned horizontally at the superior boundary of the content area. This represents the secondary navigation container with contextual rendering based on active primary navigation state.

**Content Area**: The main content region (also termed content panel or content viewport) represents the displayable space where page content renders. The content area exists as a distinct DOM region separate from navigation controls, bounded by the primary navigation panel on the left and extending to the right viewport edge.

**Spatial Relationship**: Secondary navigation occupies the top boundary of its parent content area, positioned below the application-level primary navigation controls but above the content payload. This creates a clear visual hierarchy: primary navigation at the left edge, secondary navigation at the content area's top edge, and content body below the secondary navigation.

**State-Dependent Rendering**: When the active primary navigation element changes, the system performs a content area swap operation. This operation simultaneously updates the visible content area and replaces the secondary navigation container with the navigation set associated with the newly active primary element. Each primary navigation element maintains an independent secondary navigation configuration, creating a one-to-many relationship between primary elements and secondary navigation sets.

**Coordinate System**: Using standard web coordinate terminology, primary navigation anchors to the left (x-axis origin), while secondary navigation anchors to the top (y-axis origin) of its containing content area. This orthogonal positioning (vertical versus horizontal) provides clear visual differentiation between navigation hierarchy levels.


------------------------------------------------------

## Navigation Element Terminology

### Canonical Term: Navigation Item

**Definition**: An individual interactive control within a navigation container that triggers state changes in the navigation system and controls content visibility.

### Terminological Analysis

**Navigation Item** is the canonical term because it accurately describes both the structural position (item within a navigation set) and functional role (control element) without implying specific interaction modalities.

**Navigation Element** is functionally equivalent and interchangeable with navigation item. Both terms indicate a discrete interactive component within a navigation structure.

### Level-Specific Terminology

**Primary Navigation Item** (also: L1 navigation item, primary element)

- Individual interactive control within the primary navigation container
- Triggers primary navigation switching operation
- Controls which content section and secondary navigation set becomes visible

**Secondary Navigation Item** (also: L2 navigation item, secondary element)

- Individual interactive control within the secondary navigation container
- Triggers secondary navigation switching operation
- Controls which content panel displays within the current primary context

### Alternative Terminology Evaluation

**"Switch"** - Semantically problematic. In user interface terminology, a switch specifically denotes a binary toggle control (on/off state). Navigation items are selector controls, not toggles. Using "switch" creates terminology conflict with standard UI component classification.

**"Button"** - Technically accurate from an HTML implementation perspective (often implemented as `<button>` elements), but functionally generic. "Button" describes the interface element type without conveying navigation-specific semantics.

**"Tab"** - Common in conversational usage but ambiguous without context. "Tab" can refer to browser tabs, tabbed interfaces, keyboard tab navigation, or tabular data. Requires qualifier (e.g., "navigation tab") for precision.

**"Link"** - Implies hyperlink navigation semantics (href-based routing). Navigation items in single-page applications often trigger JavaScript state changes rather than URL navigation, making "link" semantically inaccurate.

**"Control"** - Accurate but excessively generic. All interactive UI elements are controls. The term lacks specificity for documentation purposes.

**"Option"** - Implies selection from a set but lacks the action-triggering semantics central to navigation behavior.

### Recommended Usage Pattern

**Unambiguous Specification**:

- "Add a primary navigation item"
- "Remove the third secondary navigation item"
- "The active navigation item triggers content display"
- "Each primary navigation item has an associated secondary navigation set"

**Function Parameter Naming**:

```
switchPrimaryNavigation(itemIdentifier)
switchSecondaryNavigation(primaryItemIdentifier, secondaryItemIdentifier)
```

**State Description**:

- "Active navigation item" (currently selected)
- "Inactive navigation items" (not currently selected)
- "Default navigation item" (initially selected item in a set)

### Structural Context

**Navigation Item Composition**:

- Visual representation (text label, icon, or combination)
- Interactive behavior (click/tap handler)
- State indicator (visual distinction for active/inactive states)
- Identifier (unique reference within navigation set)

**Navigation Item Relationships**:

- Member of: Navigation set (collection of items at same hierarchy level)
- Container: Navigation container (structural parent element)
- Controls: Content panel(s) (display regions affected by item selection)

### Implementation Terminology

When referring to navigation items in implementation context:

**DOM Context**: "Navigation item element" or "navigation item node" **Event Handling**: "Navigation item click handler" or "navigation item selection event" **State Management**: "Active navigation item state" or "navigation item activation" **Styling**: "Navigation item styling" or "navigation item visual state"

### Summary

**Navigation Item** is the precise, unambiguous term for individual controls within navigation structures. This terminology maintains semantic accuracy across both primary and secondary navigation levels while avoiding the functional implications of alternative terms like "switch" (binary toggle) or "link" (URL navigation).


