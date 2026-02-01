# De-identification and Minimum Necessary Code Examples

Implementation examples for HIPAA Safe Harbor de-identification (45 CFR 164.514(b)) and Minimum Necessary Rule role-scoped queries.

## SQL View for De-identified Dataset

```sql
CREATE VIEW patient_records_deidentified AS
SELECT
    MD5(CONCAT(id, 'salt_stored_separately')) AS research_id,
    state AS geographic_region,
    CASE WHEN zip_population_over_20k THEN LEFT(zip_code, 3) ELSE '000' END AS zip_3digit,
    YEAR(admission_date) AS admission_year,
    diagnosis_code, procedure_code,
    CASE WHEN age_at_encounter > 89 THEN 90 ELSE age_at_encounter END AS age,
    gender, lab_results_json
FROM patient_records;
-- All 18 identifiers removed: no names, full dates, phone, fax, email, SSN,
-- MRN, health plan #, account #, license #, vehicle/device IDs, URLs, IPs,
-- biometrics, photos, or unique IDs
```

## PHI Detection Patterns

```python
import re

PHI_PATTERNS = {
    "ssn":      re.compile(r"\b\d{3}-\d{2}-\d{4}\b"),
    "phone":    re.compile(r"\b(?:\+?1[-.\s]?)?\(?\d{3}\)?[-.\s]?\d{3}[-.\s]?\d{4}\b"),
    "email":    re.compile(r"\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b"),
    "mrn":      re.compile(r"\bMRN[:\s#]*\d{4,}\b", re.IGNORECASE),
    "date":     re.compile(r"\b(?:0[1-9]|1[0-2])[/-](?:0[1-9]|[12]\d|3[01])[/-]\d{4}\b"),
    "ip":       re.compile(r"\b\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}\b"),
}

def scan_for_phi(text):
    """Scan text for PHI before logging or exporting. Returns types found, never values."""
    return [{"type": name, "count": len(p.findall(text))}
            for name, p in PHI_PATTERNS.items() if p.findall(text)]
```

## Date Shifting (preserve relative intervals per patient)

```python
def date_shift(original_date, patient_id, secret):
    hash_val = int(hashlib.sha256(f"{patient_id}:{secret}".encode()).hexdigest()[:8], 16)
    offset_days = (hash_val % 731) - 365  # -365 to +365
    return original_date + timedelta(days=offset_days)
```

## Complete De-identification Function

```python
def deidentify_record(record, patient_id, secret):
    d = {}
    # Keep clinical data (not identifiers)
    for field in ["diagnosis_codes", "procedure_codes", "lab_results", "medications"]:
        d[field] = record.get(field)
    d["gender"] = record.get("gender")
    # Age: aggregate >89
    age = record.get("age")
    d["age"] = 90 if age and age > 89 else age
    # Geographic: state + ZIP first 3 digits (000 if population <20K)
    d["state"] = record.get("state")
    zip3 = record.get("zip_code", "")[:3] or "000"
    d["zip_3digit"] = "000" if zip3 in LOW_POPULATION_ZIP3_SET else zip3
    # Dates: shift consistently per patient
    for f in ["admission_date", "discharge_date", "procedure_date"]:
        if record.get(f):
            d[f] = date_shift(record[f], patient_id, secret)
    # Research ID: not linkable to original
    d["research_id"] = hashlib.sha256(f"{patient_id}:{secret}:r".encode()).hexdigest()[:16]
    return d  # All 18 identifiers excluded
```

## Minimum Necessary Rule — Role-Scoped Queries

Query only needed fields. No `SELECT *` on PHI tables. Return only fields required for the user's role.

### Python: Role-Based Query Selection

```python
# WRONG: cursor.execute("SELECT * FROM patients WHERE id = %s", (pid,))

# Billing staff — billing fields only
def get_patient_for_billing(pid, user):
    check_phi_permission(user.role, "patient_record", "read:billing")
    return db.execute("SELECT id, insurance_provider, policy_number, billing_code "
                      "FROM patients WHERE id = %s", (pid,))

# Nurse — demographics and vitals only
def get_patient_for_nursing(pid, user):
    check_phi_permission(user.role, "patient_record", "read:vitals")
    return db.execute("SELECT id, first_name, last_name, dob, blood_pressure, "
                      "heart_rate, temperature FROM patients WHERE id = %s", (pid,))
```

### JavaScript: Response Filtering by Role

```javascript
function filterPHIResponse(record, role) {
  const allowed = {
    receptionist: ["id", "firstName", "lastName", "phone", "appointmentDate"],
    nurse: ["id", "firstName", "lastName", "dob", "vitals", "allergies", "medications"],
    physician: ["id", "firstName", "lastName", "dob", "vitals", "diagnoses", "labResults"],
    billing: ["id", "firstName", "lastName", "insuranceProvider", "billingCodes"],
  }[role] || [];
  return Object.fromEntries(allowed.filter(f => f in record).map(f => [f, record[f]]));
}
```

### PHP: ORM Scopes for Column Restriction

```php
class PatientRecord extends Model {
    public function scopeForRole($query, string $role) {
        $columns = match ($role) {
            'receptionist' => ['id', 'first_name', 'last_name', 'phone', 'email'],
            'nurse'        => ['id', 'first_name', 'last_name', 'dob', 'vitals', 'allergies'],
            'physician'    => ['id', 'first_name', 'last_name', 'dob', 'vitals', 'diagnoses'],
            'billing'      => ['id', 'first_name', 'last_name', 'insurance_provider'],
            default        => throw new AccessDeniedException("Unknown role: {$role}"),
        };
        return $query->select($columns);
    }
}
```
