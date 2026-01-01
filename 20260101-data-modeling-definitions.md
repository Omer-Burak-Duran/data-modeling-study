## 1) Modeling

### Competency Questions

**Definition:** The concrete questions our model must answer; they define scope + required detail.

**Example:** “Which prescriptions were issued in Encounter #E123?”

**Why:** If our model cannot answer its competency questions, it has bad structure.

### Knowledge level vs operational (implementation) level

**Knowledge level:** describe behavior in terms of what is known/wanted (goals/knowledge).

**Operational/symbol level:** the actual data structures, algorithms, and storage that realize it.

**Why:** First model meaning + constraints, then map to DB tables/graph storage.

---

## 2) Core building blocks (ER / relational thinking)

### Entity

**Definition:** A “thing” with identity that persists.

**Example:** `Patient`, `Practitioner`, `Medication`.

### Attribute

**Definition:** A property of an entity or relationship.

**Example:** `Patient.dateOfBirth`, `Medication.name`.

### Relationship

**Definition:** A connection between entities (often with cardinality).

**Example:** `(Patient hasAppointment Appointment)`.

### Cardinality

**Definition:** How many on each side (1–1, 1–N, N–M).

**Example:** Teacher(1) => Courses(N) refer to Teacher in the table of Course; Student(N) <=> Course(N) => use join/bridge table.

### Reify the verb (Relationship => Event)

**Idea:** If the “relationship” needs its own data (time, status, evidence, dosage...), make it an **event/entity**.

**Example:** “prescribed” becomes `Prescription` with `authored_at`, `dose`, `status`.

### Primary key (PK)

**Definition:** Column(s) that uniquely identify a row.

### Foreign key (FK) + referential integrity

**Definition:** An FK requires values to match an existing row in another table; this maintains referential integrity.

**Example:** `Prescription(encounter_id)` references `Encounter(encounter_id)`.

---

## 3) Functional dependencies & normalization (avoid redundancy bugs)

### Functional Dependency (FD)

**Definition (informal):** If you know X, there is only one possible Y (X => Y).

**Example:** `student_id => student_name` (if names do not change per record).

### Normalization (why)

**Goal:** Reduce redundancy and update anomalies by decomposing tables using FDs.

### 3NF / BCNF 

* **3NF:** eliminate most redundancy due to non-key dependencies.

* **BCNF:** stricter, every determinant must be a candidate key (handles tricky edge cases).

---

## 4) Time, state, and change

### Event vs State

* **Event:** something that happens at a time (appendable).

* **State:** current snapshot derived from events.

### Event Sourcing

**Definition:** Store every change as an event; rebuild current state by replaying events.

**Example:** `PrescriptionCreated`, `PrescriptionRenewed`, `PrescriptionCancelled`.

**Why:** Auditability + history become natural (for example, very relevant in clinic systems).

---

## 5) Knowledge graphs (RDF / OWL)

### RDF triple

**Definition:** (subject, predicate, object). Asserting a triple states the predicate holds between subject and object.

**Example:** `(Prescription, prescribedTo, Patient)`.

### RDF graph

**Definition:** A set of triples; visualizable as nodes and directed edges.

### IRIs (Internationalized Resource Identifiers)

**Definition:** used as “names”, or an equivalent of “IDs”, for graph nodes.

**Why:** Global, unambiguous identifiers for resources (subjects/predicates/objects often use IRIs). 

### N-ary relations (when triples are not enough)

**Fact:** RDF properties are binary; for relations with more than 2 participants or properties of a relation, need to use patterns (“reify” into a node).

**Example:** A `Prescription` node connects `doctor`, `patient`, `drug`, and stores `dose`.

### OWL: Class / Individual / Property

**Definition:** Individuals = objects; Classes = categories; Properties = relations (object => object or object => literal).

**Example:** `Patient` is a class; `patient123` is an individual; `hasEncounter` is an object property.

### OWL restrictions (cardinality/constraints in semantics)

**Idea:** We can state things like “exactly 1 primary physician” (cardinality), “only hasChild HappyPerson” etc.

### SHACL (Shapes Contraint Language)

**Definition:** A language to validate RDF data graphs against constraint “shapes”.

**Use case:** “Every `Prescription` must have at least 1 `medication` and exactly 1 `patient`.”

**Shortcut:** OWL is often about *meaning/inference*, SHACL is about *data validation*.

---

## 6) Property graphs (Neo4j-style)

### Node / relationship / property

**Definition:** Nodes are entities; relationships connect nodes; both can carry key-value properties; nodes can have labels, relationships have types.

**Example:** `(Patient)-[:HAS_ENCOUNTER {role:"subject"}]->(Encounter)`.

---

## 7) Causality layer (cause/effect, not just “related”)

### Causal DAG

**Definition:** A directed acyclic graph where arrows represent causal influence.

**Example:** `Severity => (TreatmentChoice, Outcome)`.

### Intervention / do-operator

**Idea:** To model “what if we set X to x?”, we use an intervention operator `do(x)` (conceptually: force X and cut its normal causes).

### Confounder

**Definition:** A variable that influences both cause and effect (can create `fake` or `artificial` associations).

**Example:** `Severity` confounds `Treatment => Outcome` if sicker patients are more likely to receive treatment.

**Key separation:** The DB/KG stores observations; the causal DAG is our *mechanism hypothesis*.

---

## 8) Micro-patterns

### M:N relationship => bridge table (relational)

**Example:** `Enrollment(student_id, course_id, term, grade)`.

### “Must keep history” => event log / event sourcing

**Example:** prescription renewals as new orders referencing prior, or as `PrescriptionEvent`s.

### “Relation has properties / more than 2 participants” => reify (graph/RDF)

**Example:** `Prescription` node connects doctor–patient–drug and stores dose/frequency.

---