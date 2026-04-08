# 🔒 Security Scan Report

**Generated:** 2026-04-02 15:50 UTC  
**Scan Root:** `/home/ubuntu/AI-SAST-Engine/media/uploads/4a699ed3-2e80-4ec2-9797-a5a8620f0044/codebase/Disease-Prediction-using-Fuzzy-Logic-master`  
**Frameworks:** hipaa

## Summary

| Metric | Count |
|--------|-------|
| Raw Findings | 18 |
| Compliance Violations | 18 |
| Attack Paths | 0 |
| Systemic Patterns | 1 |

### Severity Breakdown

- 🔴 **CRITICAL**: 8
- 🟠 **HIGH**: 7
- 🟡 **MEDIUM**: 3

## 🚨 Critical & High Violations

### [CRITICAL] Access Control — Illegal Access Prevention
- **Framework:** hipaa
- **Rule:** HIPAA-AC-002 — Access Control
- **File:** `html/patientlogin.php`
- **Description:** Authentication forms perform state-changing actions without CSRF protection, allowing attackers to force unauthorized authentication requests and potentially gain or manipulate access to ePHI.
- **Remediation:** Implement CSRF tokens for all authentication and state-changing forms. Validate tokens server-side and ensure they are unique per user session.

### [HIGH] Integrity — Input Validation
- **Framework:** hipaa
- **Rule:** HIPAA-INT-001 — Integrity
- **File:** `/home/ubuntu/AI-SAST-Engine/media/uploads/4a699ed3-2e80-4ec2-9797-a5a8620f0044/codebase/Disease-Prediction-using-Fuzzy-Logic-master/php/patientlogin.php`
- **Description:** Direct concatenation of user input into SQL queries enables SQL injection, compromising the integrity and confidentiality of systems handling ePHI.
- **Remediation:** Use parameterized queries or prepared statements (e.g., PDO or MySQLi with bound parameters) and validate all user inputs.

### [CRITICAL] Integrity — Plaintext Credentials
- **Framework:** hipaa
- **Rule:** HIPAA-ENC-009 — Encryption and Decryption
- **File:** `/home/ubuntu/AI-SAST-Engine/media/uploads/4a699ed3-2e80-4ec2-9797-a5a8620f0044/codebase/Disease-Prediction-using-Fuzzy-Logic-master/php/patientlogin.php`
- **Description:** Passwords are compared and stored in plaintext, exposing credentials and ePHI if the database is compromised.
- **Remediation:** Hash passwords using a strong adaptive hashing algorithm such as bcrypt or Argon2 and never store or compare plaintext passwords.

### [HIGH] Access Control — Authorization Enforcement
- **Framework:** hipaa
- **Rule:** HIPAA-AC-003 — Access Control
- **File:** `/home/ubuntu/AI-SAST-Engine/media/uploads/4a699ed3-2e80-4ec2-9797-a5a8620f0044/codebase/Disease-Prediction-using-Fuzzy-Logic-master/php/patientlogin.php`
- **Description:** Failure to regenerate session IDs after authentication can allow session fixation, weakening enforcement of authorized user access to ePHI.
- **Remediation:** Regenerate session identifiers upon successful login using session_regenerate_id(true) and enforce secure session handling practices.

### [HIGH] Integrity — Input Validation
- **Framework:** hipaa
- **Rule:** HIPAA-INT-001 — Integrity
- **File:** `/home/ubuntu/AI-SAST-Engine/media/uploads/4a699ed3-2e80-4ec2-9797-a5a8620f0044/codebase/Disease-Prediction-using-Fuzzy-Logic-master/php/patientsignup.php`
- **Description:** Unsanitized user input in SQL INSERT statements allows SQL injection, risking unauthorized modification or disclosure of ePHI.
- **Remediation:** Adopt prepared statements with bound parameters and apply strict server-side input validation.

### [CRITICAL] Integrity — Plaintext Credentials
- **Framework:** hipaa
- **Rule:** HIPAA-ENC-009 — Encryption and Decryption
- **File:** `/home/ubuntu/AI-SAST-Engine/media/uploads/4a699ed3-2e80-4ec2-9797-a5a8620f0044/codebase/Disease-Prediction-using-Fuzzy-Logic-master/php/patientsignup.php`
- **Description:** User passwords are stored without hashing or encryption, violating HIPAA requirements to protect authentication data tied to ePHI.
- **Remediation:** Hash all passwords with a strong one-way hashing function (bcrypt/Argon2) before storage and enforce password security policies.

### [HIGH] Access Control — Authorization Enforcement
- **Framework:** hipaa
- **Rule:** HIPAA-AC-003 — Access Control
- **File:** `/home/ubuntu/AI-SAST-Engine/media/uploads/4a699ed3-2e80-4ec2-9797-a5a8620f0044/codebase/Disease-Prediction-using-Fuzzy-Logic-master/php/patientsignup.php`
- **Description:** Session fixation risk due to missing session ID regeneration weakens control over authenticated access to ePHI.
- **Remediation:** Regenerate session IDs immediately after authentication and configure secure session cookie attributes.

### [CRITICAL] Access Control — Illegal Access Prevention
- **Framework:** hipaa
- **Rule:** HIPAA-AC-002 — Access Control
- **File:** `/home/ubuntu/AI-SAST-Engine/media/uploads/4a699ed3-2e80-4ec2-9797-a5a8620f0044/codebase/Disease-Prediction-using-Fuzzy-Logic-master/html/doctorlogin.php`
- **Description:** Lack of CSRF protection on doctor login forms enables unauthorized request submission that could compromise access to ePHI.
- **Remediation:** Add CSRF tokens to all authentication forms and verify them on submission.

### [CRITICAL] Access Control — Illegal Access Prevention
- **Framework:** hipaa
- **Rule:** HIPAA-AC-002 — Access Control
- **File:** `/home/ubuntu/AI-SAST-Engine/media/uploads/4a699ed3-2e80-4ec2-9797-a5a8620f0044/codebase/Disease-Prediction-using-Fuzzy-Logic-master/html/doctorlogin.php`
- **Description:** Signup forms collecting credentials and PII lack CSRF protection, allowing unauthorized account creation or manipulation affecting ePHI.
- **Remediation:** Implement anti-CSRF mechanisms for all signup and data submission forms.

### [HIGH] Integrity — Input Validation
- **Framework:** hipaa
- **Rule:** HIPAA-INT-001 — Integrity
- **File:** `/home/ubuntu/AI-SAST-Engine/media/uploads/4a699ed3-2e80-4ec2-9797-a5a8620f0044/codebase/Disease-Prediction-using-Fuzzy-Logic-master/php/doctorlogin.php`
- **Description:** SQL injection in doctor authentication queries allows bypass of authentication and exposure of ePHI.
- **Remediation:** Use prepared statements with parameter binding and validate all authentication inputs.

### [CRITICAL] Encryption and Decryption — Plaintext Credentials
- **Framework:** hipaa
- **Rule:** HIPAA-ENC-009 — §164.312(a)(2)(iv)
- **File:** `/home/ubuntu/AI-SAST-Engine/media/uploads/4a699ed3-2e80-4ec2-9797-a5a8620f0044/codebase/Disease-Prediction-using-Fuzzy-Logic-master/php/doctorlogin.php`
- **Description:** Passwords are handled and compared in plaintext, indicating credentials are likely stored unhashed. This violates HIPAA requirements to protect authentication credentials and ePHI from unauthorized disclosure.
- **Remediation:** Store passwords using a strong one-way hashing algorithm such as bcrypt or Argon2. Replace plaintext comparisons with password_verify() or equivalent secure verification mechanisms.

### [HIGH] Automatic Logoff — Session Timeout
- **Framework:** hipaa
- **Rule:** HIPAA-SESS-001 — §164.312(a)(2)(iii)
- **File:** `/home/ubuntu/AI-SAST-Engine/media/uploads/4a699ed3-2e80-4ec2-9797-a5a8620f0044/codebase/Disease-Prediction-using-Fuzzy-Logic-master/php/doctorlogin.php`
- **Description:** Failure to regenerate the session ID after authentication enables session fixation, undermining secure session management and increasing the risk of unauthorized access to ePHI.
- **Remediation:** Regenerate the session ID immediately after successful authentication using session_regenerate_id(true) and enforce secure session handling practices.

### [HIGH] Integrity — Input Validation
- **Framework:** hipaa
- **Rule:** HIPAA-INT-001 — §164.312(c)(1)
- **File:** `/home/ubuntu/AI-SAST-Engine/media/uploads/4a699ed3-2e80-4ec2-9797-a5a8620f0044/codebase/Disease-Prediction-using-Fuzzy-Logic-master/php/login.php`
- **Description:** Unsanitized user input is directly concatenated into SQL queries, allowing SQL injection that can compromise the integrity and confidentiality of ePHI.
- **Remediation:** Use prepared statements with parameterized queries (e.g., mysqli_prepare or PDO) and validate all user inputs before processing.

### [CRITICAL] Encryption and Decryption — Plaintext Credentials
- **Framework:** hipaa
- **Rule:** HIPAA-ENC-009 — §164.312(a)(2)(iv)
- **File:** `/home/ubuntu/AI-SAST-Engine/media/uploads/4a699ed3-2e80-4ec2-9797-a5a8620f0044/codebase/Disease-Prediction-using-Fuzzy-Logic-master/php/login.php`
- **Description:** Passwords are processed and compared in plaintext, exposing credentials and violating HIPAA safeguards for protecting authentication data.
- **Remediation:** Implement secure password storage using password_hash() and password_verify(). Migrate existing plaintext passwords by forcing password resets.

### [CRITICAL] Encryption and Decryption — Plaintext Credentials
- **Framework:** hipaa
- **Rule:** HIPAA-ENC-009 — §164.312(a)(2)(iv)
- **File:** `/home/ubuntu/AI-SAST-Engine/media/uploads/4a699ed3-2e80-4ec2-9797-a5a8620f0044/codebase/Disease-Prediction-using-Fuzzy-Logic-master/php/login.php`
- **Description:** The application stores a user's plaintext password in the session, increasing the risk of credential exposure if the session is compromised.
- **Remediation:** Never store passwords in sessions. Store only a user identifier and retrieve credentials securely from the database when needed.

## 🔄 Systemic Patterns

### SP-001
- **Description:** All authentication modules store or compare passwords in plaintext, violating HIPAA encryption requirements and exposing credentials to compromise.
- **Affected Files:** 4
- **Severity:** medium

## 📊 Token Usage

- **Total:** 18,270 / 2,000,000
- **Utilization:** 0.91%
- **Files Analyzed:** 6
