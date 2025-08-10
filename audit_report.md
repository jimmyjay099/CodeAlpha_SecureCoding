# Secure Coding Review — Vulnerability Scanner (CodeAlpha)

**Author:** Zakriya (intern)  
**Project:** vulnerability-scanner-main  
**Date:** 2025-08-10

---

## Summary
We audited `scanner.py` (Python) using automated tools and manual review. Tools used:
- `bandit` (static security analysis)
- `pip-audit` (dependency vulnerability scanner)

### Key Findings
- **High (Bandit):** `requests.get(..., verify=False)` disables SSL verification (Line 25).
- **Medium (Bandit):** `requests.get()` without a `timeout` may hang and cause resource exhaustion.
- **Dependencies (pip-audit):** 27 known vulnerabilities across installed packages (environment); notable fixes available for `requests`, `urllib3`, `flask-cors`, `cryptography`, etc.

---

## Detailed Findings and Recommendations

### Code Issues (from Bandit)
1. **Disable SSL verification (High)**  
   - **Location:** `scanner.py:25`  
   - **Description:** `verify=False` was used, allowing MITM attacks.  
   - **Recommendation:** Remove `verify=False`. If testing against self-signed certs, provide an explicit `--allow-insecure` flag (with warnings) or configure local CA bundle.

2. **No timeout on HTTP requests (Medium)**  
   - **Location:** `scanner.py:25`  
   - **Description:** Network calls without timeout may hang.  
   - **Recommendation:** Add `timeout=<reasonable_seconds>` and handle `Timeout` exceptions.

### Manual Review
- **Input validation:** The original script relied on simple `startswith` checks for URL detection; use robust parsing (`urlparse`) and simple IPv4 validation to avoid misclassification.
- **Exception handling:** Port scans did not catch socket errors; add try/except to avoid crashes.
- **Rate limiting:** Scans should include a small delay to reduce IDS/DoS flags and be polite to targets.
- **Logging:** Use `logging` instead of plain `print()` for better control of output and levels.

### Dependency Recommendations
Run `pip-audit` in a clean virtual environment and update pinned packages where fixes exist. Example safe pins for this project:
- `requests==2.32.4`
- `urllib3==2.5.0`

---

## Fixed Code
A secure, improved version `scanner_secure.py` was created with:
- Argparse CLI
- Input validation
- Request timeouts and optional `--allow-insecure`
- Improved error handling and logging
- Configurable scan parameters

(See `scanner_secure.py` file in the repo)

---

## Appendix — Scan outputs

### Bandit Output (excerpt)
```
>> Issue: [B501:request_with_no_cert_validation] Call to requests with verify=False disabling SSL certificate checks, security issue.
   Severity: High   Confidence: High
   Location: .\scanner.py:25:19

>> Issue: [B113:request_without_timeout] Call to requests without timeout
   Severity: Medium   Confidence: Low
   Location: .\scanner.py:25:19
```

### pip-audit Output (excerpt)
```
Found 27 known vulnerabilities in 16 packages
Name         Version     ID                  Fix Versions
requests     2.32.3      GHSA-9hjg-9r4m-mvj7 2.32.4
urllib3      2.2.2       GHSA-48p4-8xcf-vxj5 2.5.0
...
```

---

## How to reproduce scans (for submission)
1. Create & activate virtual environment:
   ```bash
   python -m venv venv
   venv\Scripts\activate
   pip install -r requirements.txt
   pip install bandit pip-audit
   ```
2. Run Bandit:
   ```bash
   python -m bandit -r scanner.py
   ```
3. Run pip-audit:
   ```bash
   python -m pip_audit
   ```

---

## Notes & Next Steps
- After updating dependencies, re-run `pip-audit` to verify vulnerabilities are resolved.
- Consider adding unit tests and CI pipeline (GitHub Actions) to run `bandit` and `pip-audit` automatically.
