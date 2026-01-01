# Clinic: appointments + diagnoses + prescriptions

## Scenario
- Patients book appointments, get diagnoses, receive prescriptions. 
- A prescription has drug, dosage, duration, and can be renewed.

## Relations
- Patient(patient_id, ...)
- Clinician(clinician_id, ...)
- Appointment(appt_id, patient_id, clinician_id, scheduled_at, status)
- Diagnosis(dx_id, appt_id, code, description)
- Prescription(rx_id, appt_id, drug_id, dose, frequency, start_date, end_date)
- Drug(drug_id, name, atc_code, ...)

Example: “Prescription belongs to an appointment” is a good provenance instance.

## Knowledge Graph + Contraints
Example triples for our case, in RDF style:
- (appt55, hasPatient, patient9)
- (appt55, hasClinician, doc3)
- (appt55, producedPrescription, rx77)
- ...

## FHIR
- Fast Healthcare Interoparability Resources
- global industry standard for passing healthcare data between systems
---

## 1) Competency Questions
Our model aims to answer these questions:

- For a given patient, what were all encounters and diagnoses over time?
- For a given encounter, which diagnoses were recorded and which prescriptions were issued?
- What exactly was prescribed (drug, dose, frequency, duration), and was it renewed?
- What was dispensed vs what was prescribed? (they can differ) 
- Who created/modified each record, and when? (audit/provenance)


## 2) Extracting Stable Entities and Events from the Domain

**Stable Entities (there is a persisting identity):**
- Patient, person receiving care
- Clinician, person providing care
- Drug/Medication, the prescribed thing
- Clinic/Location, where it happens

**Events (has time and status and attributes):**
- Appointment, planned meeting
- Encounter/Visit, actual interaction
- Diagnosis, condition tied to the encounter
- Prescription, request of medication
- Dispense, what was supplied by the pharmacy

In our case, a lot of "Verbs" needs to be modeled as "Events" rather than remaining as simple "Relations". Because a lot of verbs have timestamps, status changes and other similar attributes in this clinic scenario.

## 3) Relational model

### A) Core Tables

**Entities:**
- Patient(patient_id, ...)
- Practitioner(practitioner_id, ...) 
- Medication(med_id, name, code_system, code)

**Planned vs actual**
- Appointment(appt_id, patient_id, practitioner_id, scheduled_start, scheduled_end, status, location)
- Encounter(enc_id, patient_id, practitioner_id, start_time, end_time, type, location, appt_id)
  - appt_id is optional: not all encounters come from appointments (walk-ins)

**Clinical facts**
- Diagnosis(dx_id, enc_id, code_system, code, description, recorded_at)
  - For example: ICD is a common diagnosis coding system. WHO (World Health Organization) says ICD is the basis for coded health recording and statistics. 
- Prescription(rx_id, enc_id, prescriber_id, authored_at, status)
- PrescriptionLine(rx_line_id, rx_id, med_id, dose, dose_unit, frequency, route, start_date, end_date, instructions)
  - Need to split header/lines so one prescription can include multiple meds.

**Dispensing** 
- Dispense(dispense_id, rx_line_id, dispensed_at, quantity, days_supply, dispenser_org_id)

### B) Constraints
Things we should enforce with Foreign Keys, Uniqueness, and Checks. For example:
- An **Encounter** must reference exactly one **Patient**.
- A **Diagnosis** must reference exactly one **Encounter**.
- A **Prescription** must reference exactly one **Encounter** and one **Practitioner**.
- A **Prescription** must have at least 1 **PrescriptionLine**.

### C) Traceability tables
Required to see "who changed what and when":
- Provenance(record_type, record_id, who, when, why, source_system, signature_hash?)
- AuditEvent(event_id, actor, action, resource_type, resource_id, ts, ...)

According to FHIR: Provenance describes origin and influence of a resource; meanwhile AuditEvent records security, operations, and privacy-relevant events.

## 4) Knowledge Graph 
Example triples:
- (Encounter hasPatient Patient)
- (Encounter hasPractitioner Doctor)
- (Encounter recordedCondition Condition)

- (Condition codeSystem ICD)
- (Condition code [some code])
- (Condition recordedAt Timestamp)

- (Prescription authoredDuring Encounter)
- (Prescription prescribedBy Doctor)
- (Prescription hasMedication Medication)
- (Prescription hasDosage DrugDose)
- (Prescription hasFrequency [x times a day])
- (Prescription validFrom [starting date])
- (Prescription validTo [ending date])
- (Prescription renewalOf oldPrescription)

