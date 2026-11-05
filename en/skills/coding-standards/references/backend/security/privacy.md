# Privacy and personal information

This document lists engineering implementation points only and is not legal advice. Article numbers refer to the Personal Information Protection Law of the People's Republic of China (PIPL) and the EU GDPR.

**Applies to:** high risk (all rules in this document)

## Data classification

### Maintain a personal data inventory [Recommended]

Record which personal data the system processes, where it is stored, who receives it, and how long it is kept. Update the inventory along with schema and API changes.

**Why:** If you do not know where data is, you cannot respond to deletion and access requests or assess the impact of a leak.

**Source:** GDPR, PIPL (China Personal Information Protection Law), OWASP ASVS

### Define protection requirements by level [Recommended]

Assign each field to a level, and state the encryption, access, logging, and retention requirements for each level. The table below is a starting point. Adjust it to the business.

| Level | Examples | Requirements |
|---|---|---|
| Public | Public product information | No special requirements |
| Internal | Tickets, operations data | Login required for access |
| Personal information | Name, phone number, email, address | Least privilege, masked display, no plaintext in logs |
| Sensitive personal information | ID numbers, biometrics, medical and health data, financial accounts, location tracking, information of minors under 14 | Field encryption, access auditing, separate consent |

**Source:** OWASP ASVS, PIPL (China Personal Information Protection Law)

### Identify sensitive data by its legal definition [Recommended]

The categories listed in PIPL Article 28 include biometrics, religious beliefs, specific identities, medical and health data, financial accounts, location tracking, and information of minors under 14. The special categories in GDPR Article 9(1) also include race, political opinions, trade union membership, genetic data, and sexual orientation, among others. The two scopes differ. Systems serving both regions take the union.

**Source:** PIPL (China Personal Information Protection Law), GDPR

## Minimization

### Limit collection to the processing purpose [Required]

Every field must have a stated purpose. Do not collect what you do not use. Default configuration processes only the data necessary for the purpose. PIPL Article 6 requires the minimum scope needed to achieve the purpose, and GDPR Articles 5(1)(c) and 25 have similar requirements.

**Example:** When you only need to verify adulthood, store "is an adult", not the date of birth.

**Source:** PIPL (China Personal Information Protection Law), GDPR

### Return only necessary fields from APIs [Required]

Define response fields separately for lists and details. Return a full value only when it is truly needed. Mask it otherwise.

Avoid:

```json
{ "name": "Jane Doe", "idNumber": "<full ID number>", "phone": "<full phone number>" }
```

Prefer:

```json
{ "name": "J*** Doe", "phone": "138****1234" }
```

**Source:** OWASP ASVS

### Control the spread of personal data [Required]

Keep personal data out of URLs and query parameters, do not write it to browser local storage, and use `Cache-Control: no-store` on responses containing personal data. Do not send direct identifiers such as name, phone number, or email to instrumentation or third-party analytics. Send pseudonymous IDs only to entrusted processors listed in the data inventory, and send only the necessary fields. For logging rules, see [Audit and abuse prevention](audit-and-abuse.md). Do not use unprocessed production data in test environments.

**Source:** OWASP ASVS

### Distinguish de-identification from anonymization [Recommended]

De-identified data can still identify a person with additional information, so it is still personal information. Only anonymized data, which cannot identify a person and cannot be restored, stops being personal information (PIPL Articles 4 and 73). An unkeyed hash of a phone number is not anonymization, because it can be enumerated and reversed.

**Source:** PIPL (China Personal Information Protection Law)

## Consent and notice

### Record consent per purpose [Required]

Record who consented, when, to which version of the terms, and for what purpose, so you can prove it when needed (GDPR Article 7(1)). Obtain consent again when the processing purpose, method, or type of information changes (PIPL Article 14).

**Source:** GDPR, PIPL (China Personal Information Protection Law)

### Make withdrawing consent as easy as giving it [Required]

Provide a convenient way to withdraw. After withdrawal, downstream processing stops accordingly (PIPL Article 15, GDPR Article 7(3)). Do not refuse service because someone does not consent, unless the processing is necessary to provide the service (PIPL Article 16).

**Source:** PIPL (China Personal Information Protection Law), GDPR

### Use a separate toggle where separate consent is required [Required]

Sensitive personal information (Article 29), providing data to other processors (Article 23), making data public (Article 25), and providing data outside the country (Article 39) require separate consent, and cannot be bundled into one checkbox with other terms. Processing the information of minors under 14 requires guardian consent (Article 31).

**Source:** PIPL (China Personal Information Protection Law)

## Retention and deletion

### Give every data type a retention period [Required]

The retention period is the shortest time necessary to achieve the purpose, unless the law provides otherwise (PIPL Article 19, GDPR Article 5(1)(e)). Write periods into configuration and enforce them with scheduled jobs, not by hand.

| Data | Period source | Action on expiry |
|---|---|---|
| Account profile | Agreed period after account closure | Delete |
| Transaction records | Statutory retention period | Delete on expiry, restrict use until then |
| Access logs | Security and legal requirements | Expire automatically |

**Tooling:** Object storage lifecycle rules, database partition expiry.

**Source:** PIPL (China Personal Information Protection Law), GDPR, OWASP ASVS

### Delete all copies [Required]

Deletion covers the primary database, caches, search indexes, the data warehouse, file storage, dead-letter and failed-task inputs, and third-party recipients. Backups expire naturally on their retention cycle. After restoring a backup, reapply deletions that already happened. Audit records store only internal identifiers, are treated as "processing restricted" during the retention period, and are deleted on expiry.

**Why:** If you delete only from the primary database, the data can still be found in indexes and reports.

### Turn deletion triggers into a process [Required]

PIPL Article 47 requires proactive deletion when the purpose has been achieved, the service stops, the retention period expires, consent is withdrawn, or processing is unlawful. When the statutory retention period has not ended or deletion is technically difficult, stop all processing other than storage and security protection. GDPR Article 17(3) also lists exceptions. The system must support a "processing restricted" state.

**Source:** PIPL (China Personal Information Protection Law), GDPR

## Individual rights

### Provide access, correction, deletion, and export [Required]

These capabilities need an API or an admin workflow, not manual database queries (PIPL Articles 45 to 47 and 50, GDPR Articles 12, 15, 17, and 20).

| Requirement | Engineering meaning |
|---|---|
| Export | A structured, commonly used, machine-readable format, such as JSON or CSV |
| Identity verification | Confirm the requester is the individual before executing, so this does not become a leak path |
| Deadline | GDPR is generally within one month, PIPL requires prompt handling, record times in the ticket |
| Refusal | State the reason and keep a record |
| Other people's data | Exports do not include other people's information |

**Source:** PIPL (China Personal Information Protection Law), GDPR

## Incidents

### Locate and report leaks quickly [Recommended]

Logs and the data inventory must be able to answer what data leaked and who is affected. PIPL Article 57 requires immediate remediation and notification. GDPR Article 33 requires notifying the supervisory authority within 72 hours of becoming aware, where feasible.

**Source:** PIPL (China Personal Information Protection Law), GDPR
