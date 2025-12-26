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

