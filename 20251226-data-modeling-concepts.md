## 1) Knowledge Level vs Operational Level

### A) Knowledge Level (meaning / intent / why):
- What concepts exist in the domain?
- What **must be true** about them? (constraints)
- What goals, beliefs, causes, and effects exist?

The knowledge level describes a system in terms of what it knows/wants and what it does, **independent of implementation details**.

### B) Operational level (how it's stored / computed):
- SQL tables, primary keys, foreign keys, indexes
- Graph DB nodes/edges, property choices
- API payload shapes

---

## 2) Choosing the representation type based on the questions we want to answer

### A) Relational tables:
Good for:
- strong constraints, (keys, uniqueness)
- clear transactions
- reporting and aggregation

Core ideas: entities/relations, keys, functional dependencies, normalization to reduce redundancy/anomalies.

### B) Knowledge Graphs
Good for:
- first-class relationships
- evolving schemas, flexible linking
- inference, semantics

Examples:
- Resource Description Framework (RDF) and 
- Web Ontology Language (OWL)

RDF:
- consists of triples: (subject, predicate, object)
- simple, flexible, graph-based

OWL:
- adds "meaningful schema" constructs on top of RDF
- classes, properties, restrictions ("functional" properties)

### C) Casual graph
- Directed Acyclic Graph (DAG)
- Structural Casual Model

Good for:
- cause vs correlation
- intervention questions (what if we change x?)
- explicit mechanisms

Key point: 
- a causal graph is not just “data relationships” 
- it has claims about the world

---

## 3) Step-by-step thought process when modeling data

### Step 0 - Write down "competency questions" (5-15)
These are the queries our model must answer

Examples (for story teller murder story):
- Who intended to kill whom?
- Which weapon was used in which attempt?
- What evidence supports the detective's suspicion?
- When did the butler acquire the gun?

Asking questions helps us judge the model.

### Step 1 - Extract nouns, verbs, adjectives
General Entity-Relationship (ER) modeling:
- nouns -> entity types
- verbs -> relationships or events
- adjectives/adverbs -> attributes

Note: verbs like “enrolls”, “kills”, “transforms” often deserve special handling

### Step 2 - Decide relatinships vs events
- Reifying the verb when needed is important.
- If a “relationship” needs its own attributes (time, place, probability, evidence, intent, duration), it should be an EVENT.
- For example, instead of Butler -> Kills -> Lord
- We create event: Killing. 
- With fields (agent, victim, weapon, time, location, outcome, evidence…)

### Step 3 - Identity and Keys
- What makes one of these the SAME thing?
- People: stable ID
- Objects: (gun for example) serial number / inventory ID
- Events: often need a surrogate ID since "same event" is blurry
- In relational tables these become Primary Keys (PK)
- In graphs these become stable node IDs.

### Step 4 - Cardinalities and Constraints
Our model's "physics laws"

Examples:
- a course can be thought by only one teacher (1-to-1)
- one teacher can teach many courses (1-to-many)

These cardinalities are important in:
- table strucutre
- foreign keys
- whether we enforce some constraints in graphs

### Step 5 - Normalize (or intentionally Denormalize)

In relational tables:
- use functional dependencies (so that updating is not difficult)
- if there are repeated facts, join them in separate tables 
- Third Normal Form (3NF) / Boyce Codd Normal Form (BCNF) logic

In graphs:
- AVOID duplicating "truth"
- instead of repeated properties, try to make a shared node

### Step 6 - Model states explicitly if they change over time

For example:
- state snapshots (time indexed state records)
- Event sourcing (states derived from events)

If we ignore time, our model would become inconsistent.

---

## 4) Practical checklist for practicing

1. What are the stable “things” (entities)?
2. What are the changing “happenings” (events)?
3. Which verbs are actually events because they need attributes (time/place/evidence/duration)?
4. What are the constraints? (1–1, 1–many, many–many, mandatory vs optional)
5. What’s the identity rule? (what makes two records “the same”?)
6. Where does time live? (state snapshots vs events)
7. Can I answer my competency questions with this model?