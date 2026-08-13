# Data Models and Query Languages

**Source:** DDIA Ch. 2, p. 27.

The data model is the hardest decision to reverse and the one most often made by accident
(by whichever database the team already runs). Make it deliberately.

---

## 1. Relational vs. document

### 1.1 The object-relational mismatch
Application objects are trees; relational tables are flat. The gap is bridged by ORMs
(imperfectly) or by document stores (which store the tree directly). For a self-contained
tree loaded as a unit — a résumé, an order with line items, an invoice snapshot — the
document model has **better locality**: one query, one disk seek, no joins.

### 1.2 The decisive question: what are the relationships?

| Relationship | Winner | Why |
|---|---|---|
| One-to-many, loaded together, owned exclusively | Document | Locality; the tree *is* the record |
| Many-to-one (references to shared entities) | Relational | Normalize once; update in one place |
| Many-to-many | Relational | Joins are the correct primitive; emulating them in app code is slow and racy |
| Deeply interconnected, variable-depth traversal | Graph | Recursive traversal is native |

The historical argument matters: the network (CODASYL) model lost to the relational model
precisely because manual access-path traversal did not scale as a programming model.
Document databases that force applications to resolve references by hand are repeating
that mistake. **If your document store code contains a loop that fetches referenced
documents, you have hand-rolled a join — badly.**

### 1.3 Normalization
Duplicating human-meaningful values (a company name, a city name) is **denormalization**;
it requires write-time fan-out to keep copies consistent, and it goes wrong the moment one
of those writes fails. Normalizing to an ID means the value has no meaning to humans and
therefore never needs to change — which is exactly why IDs are good.

Denormalize deliberately, for a measured read-path reason, and then own the consistency
problem explicitly (see `11-stream-processing.md` on derived data).

### 1.4 Schema flexibility (schema-on-read)
Document databases are **not schemaless**. The schema is implicit, and it is enforced by
every reader. That is the whole trade:

- **Schema-on-write** (relational): explicit, enforced at write, changes require migration
  (`ALTER TABLE` — usually fast, but `UPDATE` backfills of large tables are not).
- **Schema-on-read** (document): implicit, interpreted at read, "migration" happens in
  application code that must handle every historical shape *forever*.

Choose schema-on-read when the structure is genuinely heterogeneous (many types, or an
external system determines the structure). Choose schema-on-write when the data is
uniform — the explicit schema is documentation and enforcement in one.

Practical rule: if you pick a document store, **still write down the schema and version
it**, and still handle old shapes explicitly rather than with defensive `?.` chains
scattered through the codebase.

### 1.5 Convergence
Modern relational databases support JSON/XML columns and index into them; modern document
databases support joins and some relational features. Use the hybrid deliberately:
relational for the entities and relationships, a JSON column for the genuinely
variable-shaped payload — but do not let the JSON column become the place where schema
discipline goes to die.

---

## 2. Query languages

**Declarative beats imperative** for data access, for reasons that matter operationally:

- The query says *what*, not *how*, so the optimizer is free to change the access path,
  add indexes, reorder joins, and parallelize — without you rewriting the query.
- Imperative traversal code hard-codes the access path and the ordering, which blocks both
  optimization and parallel execution.

Practical implication for application code: prefer pushing filtering, aggregation, and
joining into the database over pulling rows into the application and looping. Application-
side joins are the #1 source of N+1 queries.

**MapReduce** sits in between — a low-level programming model where you supply the
functions. Higher-level declarative layers (SQL-on-Hadoop, Spark SQL, aggregation
pipelines) exist because raw MapReduce is hard to write and harder to optimize.

---

## 3. Graph-like data models

Use when many-to-many relationships dominate and traversal depth is variable or unbounded:
social graphs, road/rail networks, dependency graphs, recommendation graphs, permission
hierarchies.

**Property graph model:** vertices with properties, edges with a label, head and tail
vertex, and properties. Any vertex can connect to any other; efficient traversal in both
directions (given indexes on head and tail). Query languages: Cypher, Gremlin. Equivalent
queries in SQL require recursive CTEs (`WITH RECURSIVE`) and become substantially harder
to write and read for variable-length paths.

**Triple-store / RDF model:** (subject, predicate, object). Query language: SPARQL.
Suited to heterogeneous, sparsely-attributed, externally-sourced data.

**Datalog:** rules that derive new predicates from existing ones, composable and
reusable; the theoretical foundation, and a good mental model for reasoning about
recursive queries.

**Practical signal:** if you find yourself writing SQL with an unknown number of joins,
or a loop in application code that walks references one hop at a time, evaluate a graph
model — or at least a recursive CTE.

---

## 4. Decision procedure

1. Enumerate the entities and the relationships between them. Write down the cardinality
   of each relationship.
2. Enumerate the actual access patterns: the top ~10 queries by frequency and by latency
   sensitivity. **The access patterns choose the model, not the entity diagram.**
3. Check for many-to-many. If present, default relational (or graph if traversal depth
   varies).
4. Check for genuine heterogeneity. If the shape varies unpredictably per record, consider
   document or a JSON column.
5. Check for analytics. If large scans over few columns are required, that is a **separate,
   derived** column-oriented store — not a reason to change the OLTP model (see
   `03-storage-and-retrieval.md`).
6. Write down the decision and the rejected alternative with its reason.

---

## 5. Review checklist

- [ ] Access patterns enumerated before the model was chosen
- [ ] Relationship cardinalities documented; many-to-many identified
- [ ] No hand-rolled joins (application loops resolving references)
- [ ] No N+1 query patterns on hot paths
- [ ] Denormalized fields have a named mechanism keeping them consistent
- [ ] If schema-on-read: the schema is documented and versioned anyway, and old shapes are
      handled in one place rather than defensively scattered
- [ ] Analytical queries are not running against the OLTP store
- [ ] Recursive/variable-depth traversals use a recursive query or graph model, not app loops
