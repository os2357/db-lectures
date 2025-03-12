# Towards Sanity in Query Languages

URLs: [Presentation](https://www.youtube.com/watch?v=TBAf5l1RmcA) (Feb 18, 2025)

## Key Points

### Relational Model
- The relational model is excellent and should be retained.
- SQL, as a query language for the relational model, has many flaws despite being widely used.

---

## Problems with SQL

### Syntax Issues
- SQL syntax is arbitrary and difficult to extend.
- The syntax aims to be human-readable but fails for complex queries.
- SQL has an excessive number of keywords, which makes it confusing.
- Example: SQL-92 had 227 keywords; SQL-2023 has 409 keywords.
  
### Error-Prone Constructs
- Cross joins can be mistaken for valid queries without error messages.
- Common table expressions (CTEs) have inconsistent and unintuitive rules.
  
### Ambiguous Semantics
- SQL’s evaluation order is not intuitive.
- Different database systems often interpret SQL semantics differently, leading to inconsistent results.

---

## Problems with SQL Implementation

### Lack of Standardization
- SQL is not a true standard; different systems implement SQL differently.
- Commercial vendors have deviations, leading to incompatibilities.
- Even systems that try to emulate standards (like PostgreSQL) still have differences.
  
### Type Systems
- The type system in SQL is inconsistent across different databases.
- SQL lacks well-defined semantics for types, implicit casting, and errors.

### Examples of Incompatibilities
- Microsoft SQL Server disallows certain queries with ORDER BY in subqueries.
- PostgreSQL does not support certain types of full outer joins.
- Different behavior in aggregate scoping across systems (e.g., BigQuery).

---

## Proposed Solutions

### New Query Language: Simple and Expressive Query Language (S)
- A new query language called "S" is proposed.
- "S" is declarative and has multiset semantics, three-valued logic, and regular syntax.
- "S" has no keywords, explicit semantic ordering, and high-level constructs.

### Advantages of S Language
- Easier to learn, debug, and implement.
- Supports reusable query components (higher-order functions).
- A prototype implementation exists that transpiles S queries to SQL.

### Standardized Relational Intermediate Representation (S-IR)
- Proposal for a formal specification of relational operators, scalar expressions, built-in functions, data types, and implicit casting rules.
- Aims to provide a well-defined semantics for database queries across different systems.

---

## Goals and Benefits

### Interoperability
- Transpilation from PostgreSQL SQL to S-IR for initial interoperability.
- Longer-term goal: adoption of S-IR directly by database engines.
  
### Enabling Innovation
- Easier implementation of new query languages.
- Facilitates development of composable data systems.
- Promotes fair competition between vendors by removing semantic islands.

---

## Challenges

### Type System Standardization
- Standardizing type behavior is the hardest part of the proposal.
  
### Transpilation Complexity
- Transpiling queries to different SQL dialects may require extensive rewrites or even custom modules.

---

## Vision for the Future

- An IR (intermediate representation) for relational queries with clear semantics would lead to a better world.
- The success of the relational model is independent of SQL’s flaws.
- Improved interoperability would drive innovation in databases and enable new ideas like query optimization as a service and federated databases.

---

## Conclusion

- SQL’s syntax and lack of standardization hinder database innovation.
- A well-defined relational IR could address many issues with SQL and lead to a more interoperable and innovative database ecosystem.