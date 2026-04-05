Task 1 — Classify and Handle PII Fields

| Field               | Type           | Action              | Justification                                                                 |
|---------------------|----------------|---------------------|-------------------------------------------------------------------------------|
| **full_name**       | Direct PII     | ❌ Drop              | Directly identifies an individual, so it must be removed                      |
| **email**           | Direct PII     | ❌ Drop              | A unique identifier with high risk of misuse                                  |
| **date_of_birth**   | Indirect PII   | 🔒 Mask (year only) | Full DOB can lead to identification, so only partial data should be kept      |
| **zip_code**        | Indirect PII   | 🔒 Mask (partial)   | Reveals location, so only limited information should be retained              |
| **job_title**       | Indirect PII   | 🔄 Generalize       | Rare job titles may identify individuals, so use broader categories           |
| **diagnosis_notes** | Sensitive Data | 🔄 Pseudonymize     | Necessary for research, but remove names/IDs to protect identity              |


Task 2 — Ethical Issues

### Issue 1: API Key Exposure
```python
import os
API_KEY = os.getenv("API_KEY")
```

### Issue 2: Excess Data Collection
```python
for page in range(1, 11):
    pass
```
Task 2 — Audit the API Script for Ethical Compliance
Violation 1 — Bulk Scraping Beyond Reasonable Use (TOS Violation)
Problem: The script blindly loops through 100 pages with no rate limiting, no check on whether all that data is actually needed, and no respect for the API's usage limits. Free-tier keys almost universally prohibit bulk data harvesting. This violates the API's Terms of Service and the data minimization principle (collect only what you need).
Corrected Code:
import requests
import time

API_URL = "https://healthstats-api.example.com/records"
API_KEY = "free_tier_key_abc123"

MAX_PAGES = 5        # Only collect what is genuinely needed
RATE_LIMIT_DELAY = 1 # Seconds between requests — respect the API

records = []
for page in range(1, MAX_PAGES + 1):
    response = requests.get(API_URL, params={"page": page, "key": API_KEY})

    if response.status_code == 429:  # Too Many Requests
        print("Rate limit hit. Stopping collection.")
        break

    data = response.json()
    records.extend(data["results"])
    time.sleep(RATE_LIMIT_DELAY)  # Polite delay between requests

    Violation 2 — Storing Raw PII Permanently Without Anonymization (Ethical Violation)
Problem: save_to_database(records) stores the raw, unprocessed records permanently, including all Direct and Indirect PII fields. This violates:

Data minimization (GDPR Article 5, HIPAA) — don't store more than needed
Purpose limitation — data collected for analysis shouldn't be stored indefinitely in identifiable form
Storage limitation — personal data shouldn't be kept longer than necessary

Corrected Code:
from datetime import datetime
import hashlib

def anonymize(record):
    """Strip or pseudonymize PII before storage."""
    dob = datetime.strptime(record["date_of_birth"], "%Y-%m-%d")
    age = (datetime.today() - dob).days // 365  # Derive age, drop exact DOB

    return {
        # Pseudonymize: replace name+email with an anonymous ID
        "patient_id": hashlib.sha256(record["email"].encode()).hexdigest()[:12],
        "age": age,                                      # Replace DOB with age
        "zip_code": record["zip_code"][:3] + "**",       # Mask last 2 digits
        "job_category": generalize_job(record["job_title"]),  # Generalize job
        "diagnosis_notes": redact_names(record["diagnosis_notes"])  # Scrub free text
        # full_name and email are intentionally excluded
    }

anonymized_records = [anonymize(r) for r in records]
save_to_database(anonymized_records)  # Only store anonymized data
