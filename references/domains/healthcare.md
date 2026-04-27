# Domain Pack: Healthcare / PHI / Clinical

Extends the `security`, `compliance`, and `logic-math` agents. Load when the stack handles PHI (Protected Health Information), patient records, clinical data, or medical device integration.

The stakes: HIPAA violations start at $100/record and escalate. Patient-safety bugs in clinical software can harm people. Both are why this pack exists.

## Disclaimer

You are not a clinician and not a HIPAA compliance officer. This pack covers code-level patterns that create exposure. The user should pair any serious findings with a formal HIPAA risk assessment and (for clinical software) medical domain expert review.

## PHI handling

### What counts as PHI

HIPAA's 18 identifiers:
1. Names
2. Geographic subdivisions smaller than a state (including zip code in some contexts)
3. Dates directly related to an individual (birth date, admission date, etc.) — except year for persons under 90
4. Phone numbers
5. Fax numbers
6. Email addresses
7. SSNs
8. Medical record numbers
9. Health plan beneficiary numbers
10. Account numbers
11. Certificate / license numbers
12. Vehicle identifiers
13. Device identifiers and serial numbers
14. Web URLs
15. IP addresses
16. Biometric identifiers
17. Full-face photos and comparable images
18. Any other unique identifying number, characteristic, or code

Any combination of these + health information = PHI.

### PHI at rest

Flag:
- PHI columns not encrypted at rest (application-level or column-level encryption; disk-level is insufficient alone)
- Backups containing PHI without encryption
- Logs that contain PHI (every error log reference that includes patient ID + any symptom / diagnosis is PHI)
- Cache layers (Redis, memcached) holding PHI without encryption or TTL discipline

### PHI in transit

Flag:
- Non-TLS connections between services handling PHI
- Internal service-to-service traffic unencrypted (zero-trust architecture: always TLS)
- PHI sent via email (most email is untrusted in transit; requires encrypted patient-portal pattern)
- PHI in URLs (logged in server/CDN/proxy logs; referer headers)
- PHI in query parameters

### PHI in logs and error reports

Flag:
- Sentry / error reporter configured without PHI scrubbing
- `console.log(user)` where `user` is a patient record
- Debugging logs that dump request bodies
- Access logs that include request bodies or response bodies
- Developer tools / admin dashboards that show PHI to users who don't need it

### Minimum necessary

HIPAA's "minimum necessary" rule requires that only the data needed for the task be accessed.

Flag:
- API endpoints that return entire patient record when only specific fields are needed
- `SELECT *` on patient tables
- Indexed fields exposed in APIs without access controls
- Batch export endpoints without a rationale

## Authentication and authorization

### Patient authentication

Flag:
- Weak password requirements on patient portals (should be NIST-aligned)
- MFA not offered for patient accounts
- Account recovery flows that leak whether an account exists ("No account found for this email")
- Patient-portal session timeout too long (HIPAA environment typically wants 15-30 min)

### Clinician authentication

Flag:
- Shared accounts between clinicians ("Dr. Smith's office" as a user)
- No audit trail of individual clinician actions
- No periodic re-authentication for high-risk actions (viewing records of VIP patients, deleting records)

### Role-based access

Healthcare roles vary widely: primary physician, specialist, nurse, billing, admin, patient themselves, legal guardian.

Flag:
- Flat role model (admin / user) insufficient for clinical context
- No "break-glass" access pattern (emergency access with extra audit)
- Patient-to-clinician relationships not explicitly modeled (a specialist at the same hospital doesn't automatically have access to every patient)

## Audit logs

HIPAA requires audit trails for PHI access.

Flag:
- No audit log for PHI access
- Audit log missing fields: who, what data, when, from where (IP), why (reason / purpose-of-access attestation)
- Audit log in same DB / storage as the data (insider can tamper)
- No retention policy for audit logs (HIPAA requires 6 years)
- No audit log review mechanism (log exists but nobody reads them)
- Patient can't request an audit of who accessed their records (HIPAA right of accounting)

## Clinical correctness

### Units and ranges

Clinical data is unit-sensitive; unit bugs can kill patients.

Flag:
- Dosing calculations without explicit units
- Weight in lbs vs kg confusion (3x difference!)
- Temperature in F vs C confusion
- Lab values displayed in units different from the standard for the test
- No range validation on vital signs (heart rate of 3000 displayed without warning)
- No plausibility checks (12-year-old with systolic BP of 280 — probably a typo, should warn)

### Dosing calculations

If the app calculates medication doses:

**Flag aggressively.** Every dosing calculation should:
- Use integer or decimal math, never float
- Round to the minimum increment for the dosage form (can't give 1.333 tablets; round down to 1 or discuss halving)
- Warn on doses outside typical ranges
- Require explicit acknowledgment for pediatric doses (weight-based)

### Clinical decision support

If the app surfaces alerts / reminders / warnings:

Flag:
- Alerts that can be silently dismissed without attestation
- Alerts that aren't logged when dismissed (liability)
- Drug-drug interaction checks based on outdated data
- Allergy checks that match only by exact name (penicillin ≠ amoxicillin in a naive string match)
- Pregnancy-contraindicated drug checks that don't respect patient sex / pregnancy status

## Consent and authorization forms

Flag:
- Consent stored as boolean without version / timestamp / scope
- Consent revocable but revocation doesn't propagate (third parties already have data)
- Opt-in treated as opt-out or vice versa
- HIPAA authorization forms (for specific disclosures) not modeled separately from general consent

## Patient rights

Flag:
- No process for patient to request their records
- No process for patient to request amendment
- No process for patient to request restriction on disclosures
- No process for accounting of disclosures
- No designated Privacy Officer contact in the application

## Third-party integrations

Every third party handling PHI needs a BAA (Business Associate Agreement).

Flag:
- Analytics tools (Google Analytics, Segment, Mixpanel) collecting data from logged-in patient sessions (PHI risk if page URLs or event properties identify patient + context)
- Session recording tools (Hotjar, FullStory, LogRocket) running on PHI-containing pages
- Email services (SendGrid, Postmark) receiving PHI in email bodies
- SMS services (Twilio) receiving PHI in message bodies
- CDN / performance tools with access to PHI-bearing requests

Any of these needs a BAA. Flag their presence and note the need for verification.

## Specific clinical software patterns

### FHIR / HL7 integration

If integrating with EHRs via FHIR or HL7:

Flag:
- Trust of EHR-provided data without validation (EHR data is messy; validate shapes)
- Assumption of specific FHIR version (R4 vs STU3 have differences)
- No rate limiting on EHR API calls (most EHRs have strict limits)
- No retry / backoff for EHR integration errors
- Sync logic that doesn't handle "patient moved to another provider" gracefully

### DICOM / imaging

If handling DICOM:

Flag:
- DICOM tags containing PHI (patient name, birth date) not scrubbed for de-identified exports
- DICOM viewers with XSS via metadata fields
- No limits on DICOM file size (multi-GB studies can DoS ingest)

### Prescription / e-prescribing

Flag:
- EPCS (Electronic Prescribing of Controlled Substances) without proper identity proofing
- Prescription signing without two-factor authentication (DEA requires this)
- No DEA number validation for prescribers
- No check against PDMP (Prescription Drug Monitoring Program) data where required

## De-identification

For research / analytics uses of patient data:

Flag:
- "De-identified" data that's actually trivially re-identifiable (birthdate + zip + sex identifies most people)
- Safe Harbor de-identification not applied (removal of all 18 identifiers)
- Expert Determination claim without documentation
- De-identified data combined with re-identifying sources (common research bug)

## Breach notification

Flag:
- No mechanism to detect breaches (no anomaly detection on access patterns)
- No documented incident response plan
- No automated alerts on mass data access (one user viewing 1000 records)

## Process

For a healthcare codebase:

1. Map the PHI flow: where does PHI enter, where is it stored, who can see it, where does it leave?
2. Walk every endpoint that returns patient data, checking role and audit
3. Walk every clinical calculation (dosing, risk scores, etc.) for unit / range / boundary bugs
4. Check third-party integrations for BAA requirements
5. Check the audit log: what's logged, retention, access to logs
6. Write findings.

## Severity calibration

- **Critical**: PHI in plaintext at rest / in logs; unauthenticated access to any PHI; clinical calculation bug that could affect treatment
- **High**: PHI to third parties without BAA; missing audit log; weak authentication on patient portal; dosing calc without range validation
- **Medium**: missing minimum-necessary enforcement; consent model gaps; session timeouts too long
- **Low**: nits around patient-facing copy; minor audit fields missing

Always treat healthcare findings conservatively — err toward reporting. The cost of a missed critical finding here is measured in people, not dollars.
