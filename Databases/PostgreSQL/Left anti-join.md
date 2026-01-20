## Formal Terminology

The `-notin` comparison operation is formally referred to as **set difference** in set theory and mathematical notation, represented as A \ B or A - B.

In relational database terminology, this operation is called a **left anti-join** or **anti-semi-join**. The anti-join returns rows from the left operand that have no corresponding match in the right operand based on the join predicate.

## Domain-Specific Terms

**Set Theory:**

- **Set Difference**: A \ B = {x ∈ A | x ∉ B}
- **Relative Complement**: Elements in set A that are not in set B

**Relational Algebra:**

- **Anti-join**: R ▷ S (returns tuples from R with no match in S)
- **Left Anti-semi-join**: Specific variant that preserves left operand schema

**SQL Database Systems:**

- Implemented as `WHERE NOT IN` subquery
- Alternatively expressed as `WHERE NOT EXISTS` correlated subquery
- Sometimes called **exclusion join** in query optimization contexts

## PowerShell Context

In PowerShell, the `-notin` operator performs **membership negation testing**. The expression `$_.Path -notin $currentPaths` evaluates to true when the left operand does not exist as an element within the right operand collection.

The filtering operation `Where-Object { $_.Path -notin $currentPaths }` implements a **filter predicate** that selects elements from the input collection based on the anti-membership condition.

## Practical Nomenclature

When documenting this operation in technical specifications:

- **Set difference operation**: When emphasizing mathematical precision
- **Anti-join filtering**: When discussing data processing pipelines
- **Exclusion filter**: When describing the functional purpose
- **Missing element detection**: When focusing on the business logic

The PowerShell `-notin` operator is the inverse complement of the `-in` operator, implementing set non-membership testing with O(n) complexity for array collections.