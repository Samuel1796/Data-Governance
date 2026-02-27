# Data Quality Assessment Report



## Executive Summary

A data quality audit was conducted on a 50-record sample of the MedTrack Ghana patient appointment CSV database. The audit identified violations across all six standard data quality dimensions: Accuracy, Completeness, Consistency, Timeliness, Validity, and Uniqueness. These issues are directly responsible for reported SMS reminder failures, inflated patient counts in reports, and billing breakdowns. This report documents each finding with specific record-level evidence, assesses the operational business impact, and recommends three prioritised corrective actions.

---

## Task 1: Data Quality Issues - Six Dimensions

| Dimension | Record(s) | Issue Found | Severity | Impact Area |
|---|---|---|---|---|
| Accuracy | P002 (row 2) | Phone number `244789012` missing leading zero - should be `0244789012` | High | Operations (SMS) |
| Completeness | P004 | PatientName field is empty - no name recorded for this patient | High | Clinical / Finance |
| Consistency | P002 vs P006 | Same patient has two different phone entries: `244789012` and `0244789012`. Doctor name `dr. osei` vs `Dr. Osei` - inconsistent casing | High | All Departments |
| Timeliness | P003 | Date `10/16/2025` uses MM/DD/YYYY format; suggests data from legacy/old system | Medium | Operations / Reporting |
| Validity | P002, P003 | AppointmentDate formats `15/10/2025` and `10/16/2025` violate the required YYYY-MM-DD standard | High | Operations / Billing |
| Uniqueness | P002 (rows 2 & 6) | Ama Serwa appears twice with same appointment date and doctor - a true duplicate record | High | Finance / Reporting |

### Dimension Notes

**Accuracy:**
Row 2 for patient P002 (Ama Serwa) records her phone number as `244789012`. This is a 9-digit number and is factually incorrect - all valid Ghanaian mobile numbers are 10 digits starting with 0. Sending an SMS to this number will fail at the telecom level.

**Completeness:**
Record P004 has a completely empty PatientName field. This violates the core requirement that every patient appointment must be associated with an identifiable patient. Without a name, billing, clinical records, and audit trails are all compromised.

**Consistency:**
Two types of inconsistency were identified: (1) patient P002's phone number is recorded two different ways (`244789012` in row 2 vs. `0244789012` in row 6), and (2) the same doctor is referenced as `dr. osei` (row 2) and `Dr. Osei` (rows 1 and 6), indicating no standardisation in data entry.

**Timeliness:**
Record P003 uses a MM/DD/YYYY format (`10/16/2025`), which suggests the data was entered from a legacy system or copied from an old template. This mismatch in format convention may also indicate that historical records were imported without proper transformation, making timely processing unreliable.

**Validity:**
Rows 2 and 3 use date formats (`15/10/2025` and `10/16/2025`) that do not conform to the expected ISO 8601 standard (YYYY-MM-DD). Most systems cannot automatically parse these alternative formats, causing them to be skipped or rejected in scheduling and billing workflows.

**Uniqueness:**
Patient P002 (Ama Serwa) appears in both row 2 and row 6 with the same appointment date (2025-10-15) and same doctor (Dr. Osei). While one has an incorrect phone number, they refer to the same appointment - making one a duplicate record that should not exist in the database.

---

## Task 2: Business Impact Assessment

| Quality Issue | Why It Causes Problems | Most Affected Function |
|---|---|---|
| Accuracy - Wrong phone number (P002) | SMS reminders sent to `244789012` will fail because the number is invalid. Patient misses appointment, resulting in a no-show and lost revenue. | Operations (SMS/Reminders) |
| Completeness - Missing patient name (P004) | Billing invoices cannot be generated without a patient name. Reports show unnamed patients, which is a compliance/audit risk. | Finance & Clinical |
| Consistency - Duplicate P002 records | Reports count Ama Serwa twice, inflating patient numbers. Billing may charge the patient twice. Appointment history is unreliable. | Finance & Operations |
| Validity - Invalid date formats | System parsers reject `15/10/2025` and `10/16/2025`. Scheduled SMS reminders and appointment slots will not be created, leading to missed care. | Operations & Billing |
| Timeliness - Legacy date format | Records entered in old format may represent stale or migrated data. If dates are wrong, appointment scheduling and clinical timelines are disrupted. | Clinical & Operations |
| Uniqueness - Duplicate appointment for P002 | Billing system may issue two invoices for the same visit. Resources are double-booked. Doctor workload calculations are skewed. | Finance |

**Summary:** The most cross-cutting issues are the invalid date formats (affecting scheduling, reminders, and billing simultaneously) and the duplicate P002 record (affecting Finance, Operations, and reporting integrity). The missing patient name (P004) is the most immediate compliance risk.

---

## Task 3: Recommended Solutions (Top 3 Critical Issues)

### Issue 1: Invalid Phone Numbers (Accuracy + Consistency)

**Technical Solution:**
Add a regex validation rule on the PhoneNumber field: must match `^0[0-9]{9}$` (10 digits, starting with 0). Run a one-time normalization script to prepend `0` to any existing 9-digit numbers.



**Responsible Party:** Backend Developer implements the validation rule; QA Engineer writes test cases; Database Admin runs the normalization script on existing data.

**Verification Method:** Run test cases with invalid numbers - system must reject them. Verify all phone numbers in the DB match the regex. Retest SMS delivery for previously failing patients and confirm delivery receipts.

---

### Issue 2: Invalid & Inconsistent Date Formats (Validity + Timeliness)

**Technical Solution:**
Enforce ISO 8601 (`YYYY-MM-DD`) as the only accepted date format at the data entry layer. Add a backend validator that rejects all other formats. Write a migration script to convert existing `DD/MM/YYYY` and `MM/DD/YYYY` entries.


**Responsible Party:** Backend Developer writes the validator; QA Engineer tests edge cases (e.g., Feb 30, ambiguous MM/DD vs DD/MM); Business Analyst confirms correct date values with data owners before migration.

**Verification Method:** Audit all AppointmentDate fields - zero records should exist outside `YYYY-MM-DD` format. Test that the UI and API reject alternative formats. Re-run the appointment reminder scheduler and confirm it picks up all records correctly.

---

### Issue 3: Duplicate Records & Missing Patient Name (Uniqueness + Completeness)

**Technical Solution:**
Add a composite unique constraint on `(PatientID, AppointmentDate, DoctorName)` to prevent duplicate bookings. Enforce `NOT NULL` on PatientName. Run a deduplication query to identify and merge existing duplicate records.


**Responsible Party:** Database Admin implements the constraints; Backend Developer updates the API to handle constraint violations gracefully; QA Engineer verifies deduplication results with clinical staff.

**Verification Method:** Query for duplicate `(PatientID + AppointmentDate + DoctorName)` combinations - result should return zero rows. Confirm PatientName is non-null for all records. Validate patient count in reports matches the confirmed patient list.

---

### Implementation Priority

These three fixes should be implemented in the following order based on business urgency:

1. **Date format standardisation first** - blocking the scheduling and SMS reminder system; affects the most records.
2. **Phone number validation second** - SMS delivery failures are a direct, ongoing operational loss.
3. **Deduplication and NOT NULL constraints third** - critical for financial accuracy and audit compliance.

---

## Task 4: QA Perspective - The Biggest Risk of Poor Data Consistency



In a healthcare system like MedTrack Ghana, consistency failures do not always throw visible errors. Unlike a system crash or an explicit validation failure, inconsistent data - such as a patient appearing under two different phone number formats - passes through data pipelines, gets stored successfully, and appears in reports. The damage is invisible until an SMS fails to deliver, a patient is double-billed, or a clinical summary is inaccurate.

### Why This Is Dangerous for QA Engineers

- **Test environments are often sanitised** with clean, consistent data - meaning the inconsistency bug may never be caught during QA testing.
- **Automated test scripts may pass** (no exception thrown), even though the business output is wrong.
- **Bug reports related to consistency failures are hard to reproduce and trace** - they appear as intermittent anomalies rather than clear defects.

### The Specific Risk for MedTrack Ghana

If the SMS reminder system queries by PatientID and selects the first phone number found, it may consistently pick the malformed `244789012` for Ama Serwa. Every reminder for her appointment silently fails - but the system logs show *"message sent successfully"*. No error, no alert, no fix - until the patient complains or misses a critical appointment.

This demonstrates why QA Engineers must advocate for:

- **Data consistency checks as first-class test cases** - not just functional tests.
- **End-to-end tests that verify outputs against known-clean data**, not just API responses.
- **Monitoring and alerting on data anomalies in production**, not just in test environments.

### Conclusion

Ultimately, poor data consistency erodes trust in the entire system. When reports, billing records, and patient histories can silently contradict each other, no stakeholder - clinical, financial, or operational - can rely on the data to make decisions. For a health tech company like MedTrack Ghana, this is not merely a technical problem: it is a **patient safety and organisational credibility risk**.
