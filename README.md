# Money Transfer System — QA Documentation Pack

QA documentation for the **Money Transfer System**, covering amount validation, daily transfer limits, SMS OTP approval, decision tables, state-transition testing, traceability, release verification, defect reporting, and the AI-assisted testing appendix.

> Prepared as a Software Testing and Debugging practice work.  
> The test-design approach follows black-box techniques and is structured to satisfy the relevant requirements of **ISO/IEC/IEEE 29119-3**.

## Contents

- [1. Test Plan](#1-test-plan-one-page)
- [2. Test Cases](#2-twelve-test-cases)
- [3. Traceability Matrix](#3-traceability-matrix)
- [4. Release Checklist](#4-release-checklist-sms-code-flow)
- [5. Defect Reports](#5-three-defect-reports)
- [6. AI Appendix](#6-ai-appendix-level-1)

---

**Course:** Software Testing and Debugging (KBTU 2026)   

**Target Feature:** Money Transfer System (Amount Limits, Daily Limits & SMS OTP Approval)   

**Author:** Baishiganov Alisher

## 1. Test Plan (One-Page)

### 1.1 Scope

- **Scope In:**
  - Transfer amount validation (100 to 500,000 KZT, whole tenge enforcement).   
  - Daily limit enforcement (1,000,000 KZT cumulative across all transfers).   
  - Daily limit reset mechanism at midnight Almaty time.   
  - 6-digit SMS OTP trigger for transfers exceeding 100,000 KZT.   
  - SMS code lifecycle (120-second validity timer, 3-attempt wrong code limit and transfer cancellation).   
  - Error precedence handling (Amount validation errors taking priority over Daily Limit errors).   
- **Scope Out:**
  - Interbank clearing house backend protocols and settlement networks.   
  - Core UI/UX design system styling and animations.   
  - Push notifications and email alert delivery services.   
  - KYC / AML legal verification procedures.   

### 1.2 Approach

- **Testing Levels:** System Testing (API & UI integration) and Confirmation/Regression Testing.   
- **Design Techniques:** Black-box techniques including Equivalence Partitioning (EP), 3-value Boundary Value Analysis (BVA), Decision Tables, and State Transition Testing.   
- **Automation:** Manual execution for release verification; core API flows tagged for post-release regression automation.   

### 1.3 Entry and Exit Criteria

- **Entry Criteria:**
  1. Test build deployed to Staging environment with release candidate tag.   
  2. Smoke test suite 100% passed.   
  3. Test accounts provisioned with configurable balances and mock SMS Gateway enabled.   
  4. Feature requirements and API contracts baselined.   
- **Exit Criteria:**
  1. 100% of planned test cases executed.   
  2. Zero open Blocker or Critical defects; all open Medium/Low defects documented and accepted by Product Owner.   
  3. Traceability matrix shows 100% coverage of defined requirements.   
  4. Release Checklist fully signed off.   

### 1.4 Top Three Product Risks

1. **R-01: Double Debit / Replay Attack (Severity: High)**
   - *Risk:* Concurrent requests or repeated button taps during SMS verification cause duplicate debiting of user funds.   
   - *Mitigation:* Idempotency key verification on API level and disable-on-click UI state.   
2. **R-02: SMS OTP Verification Bypass (Severity: High)**
   - *Risk:* Flaws in state machine allow transferring >100,000 KZT without entering a valid 6-digit code or after code expiry.   
   - *Mitigation:* Strict server-side session state validation and state transition coverage testing.   
3. **R-03: Daily Limit Race Condition (Severity: Medium)**
   - *Risk:* Parallel transfer requests submitted simultaneously bypass the 1,000,000 KZT cumulative daily limit.   
   - *Mitigation:* Atomic transaction processing and database row-locking on user daily total.   

## 2. Twelve Test Cases

### TR-AMT-001 (Valid Amount — Lower Boundary)

- **Traces to:** REQ-AMT-01 (100–500,000 KZT range)   
- **Preconditions:** Test user TC-001 logged in, account balance = 1,000,000 KZT, daily spent = 0 KZT.   
- **Inputs / Steps:**
  1. Open Transfers screen.   
  2. Select active source account.   
  3. Enter amount `100` KZT.   
  4. Tap "Continue".   
- **Expected Result:** Amount is accepted without error. System transitions to immediate execution (no SMS required as amount ≤ 100,000 KZT).   
- **Postconditions:** Transaction completed; balance decreases to 999,900 KZT; daily total = 100 KZT.   

### TR-AMT-002 (Invalid Amount — Below Minimum Bound)

- **Traces to:** REQ-AMT-01 (100–500,000 KZT range)   
- **Preconditions:** Test user TC-001 logged in, account balance = 1,000,000 KZT, daily spent = 0 KZT.   
- **Inputs / Steps:**
  1. Open Transfers screen.   
  2. Enter amount `99` KZT.   
  3. Tap "Continue".   
- **Expected Result:** Validation fails. System displays inline error: `"Amount must be between 100 and 500,000 KZT"`. "Continue" action is blocked.   
- **Postconditions:** Account balance remains 1,000,000 KZT; no debit occurs.   

### TR-AMT-003 (Invalid Amount — Above Maximum Bound)

- **Traces to:** REQ-AMT-01 (100–500,000 KZT range)   
- **Preconditions:** Test user TC-001 logged in, account balance = 1,000,000 KZT, daily spent = 0 KZT.   
- **Inputs / Steps:**
  1. Open Transfers screen.   
  2. Enter amount `500001` KZT.   
  3. Tap "Continue".   
- **Expected Result:** Validation fails. System displays inline error: `"Amount must be between 100 and 500,000 KZT"`. "Continue" action is blocked.   
- **Postconditions:** Account balance remains 1,000,000 KZT; no debit occurs.   

### TR-AMT-004 (Valid Amount — Upper Boundary)

- **Traces to:** REQ-AMT-01 (100–500,000 KZT range), REQ-SMS-01   
- **Preconditions:** Test user TC-001 logged in, account balance = 1,000,000 KZT, daily spent = 0 KZT.   
- **Inputs / Steps:**
  1. Open Transfers screen.   
  2. Enter amount `500000` KZT.   
  3. Tap "Continue".   
- **Expected Result:** Amount accepted. Since 500,000 KZT > 100,000 KZT, system transitions to 6-digit SMS OTP entry screen.   
- **Postconditions:** Transfer session created in "Awaiting OTP" state; balance remains 1,000,000 KZT pending confirmation.   

### TR-AMT-005 (Invalid Amount — Non-Whole Tenge / Fractional Value)

- **Traces to:** REQ-AMT-02 (Whole tenge only)   
- **Preconditions:** Test user TC-001 logged in, account balance = 1,000,000 KZT.   
- **Inputs / Steps:**
  1. Open Transfers screen.   
  2. Enter amount `150.50` KZT.   
  3. Tap "Continue".   
- **Expected Result:** Validation fails. System displays error: `"Amount must be a whole tenge number"`. Transfer submission is blocked.   
- **Postconditions:** Balance remains unchanged.   

### TR-AMT-006 (Invalid Amount — Non-Numeric Input)

- **Traces to:** REQ-AMT-02 (Whole tenge / Format rule)   
- **Preconditions:** Test user TC-001 logged in.   
- **Inputs / Steps:**
  1. Open Transfers screen.   
  2. Input string `abc` into amount field.   
  3. Tap "Continue".   
- **Expected Result:** Non-numeric input is sanitized/rejected. System displays error: `"Please enter a valid numeric amount"` or button remains disabled.   
- **Postconditions:** No transfer request is dispatched to the server.   

### TR-APP-001 (Approval Rule R2 — Immediate Execution)

- **Traces to:** REQ-LIM-01, REQ-SMS-01   
- **Preconditions:** Test user TC-002 logged in, balance = 500,000 KZT, current daily spent = 100,000 KZT.   
- **Inputs / Steps:**
  1. Enter transfer amount `50000` KZT.   
  2. Tap "Continue".   
- **Expected Result:** Amount is valid ($100 \le 50,000 \le 500,000$), daily total ($150,000 \le 1,000,000$) is preserved, and amount $\le 100,000$ KZT. Transfer executes immediately without prompting for SMS OTP.   
- **Postconditions:** Balance = 450,000 KZT; cumulative daily spent = 150,000 KZT.   

### TR-APP-002 (Approval Rule R3/R4 — Daily Limit Exceeded)

- **Traces to:** REQ-LIM-01   
- **Preconditions:** Test user TC-003 logged in, balance = 1,000,000 KZT, current daily spent = 800,000 KZT.   
- **Inputs / Steps:**
  1. Enter transfer amount `300000` KZT.   
  2. Tap "Continue".   
- **Expected Result:** Amount is valid ($300,000$), but new daily total ($1,100,000$) exceeds the $1,000,000$ KZT daily limit. Transaction is rejected with error: `"Daily transfer limit of 1,000,000 KZT exceeded"`.   
- **Postconditions:** Balance remains 1,000,000 KZT; daily spent remains 800,000 KZT.   

### TR-APP-003 (Approval Rule R7/R8 — Both Amount and Daily Limit Violated)

- **Traces to:** REQ-ERR-01 (Error precedence)   
- **Preconditions:** Test user TC-004 logged in, balance = 1,000,000 KZT, current daily spent = 950,000 KZT.   
- **Inputs / Steps:**
  1. Enter transfer amount `600000` KZT (violates amount max 500,000 KZT and causes daily total 1,550,000 KZT).   
  2. Tap "Continue".   
- **Expected Result:** System enforces precedence rule. Only the **Amount error** is displayed: `"Amount must be between 100 and 500,000 KZT"`. The daily limit error is suppressed.   
- **Postconditions:** Transaction rejected instantly; no SMS generated.   

### TR-SMS-001 (SMS Flow — Success on First Attempt)

- **Traces to:** REQ-SMS-01, REQ-SMS-02   
- **Preconditions:** Test user TC-005 logged in, balance = 500,000 KZT, daily spent = 0 KZT. Transfer initiated for `150000` KZT (State: `Waiting, attempt 1`).   
- **Inputs / Steps:**
  1. Receive valid 6-digit SMS OTP on registered device.   
  2. Input correct 6-digit OTP within 120 seconds.   
  3. Tap "Confirm".   
- **Expected Result:** OTP accepted. State transitions to `Confirmed`. System shows success screen: `"Transfer completed successfully"`.   
- **Postconditions:** Account debited by 150,000 KZT; daily spent updated to 150,000 KZT.   

### TR-SMS-002 (SMS Flow — Code Expiry Timeout)

- **Traces to:** REQ-SMS-02 (120 seconds validity)   
- **Preconditions:** Test user TC-006 logged in. Transfer initiated for `200000` KZT (State: `Waiting, attempt 1`).   
- **Inputs / Steps:**
  1. Wait for 121 seconds without entering the OTP.   
  2. Enter the received 6-digit OTP.   
  3. Tap "Confirm".   
- **Expected Result:** Code validation fails due to timeout. State transitions to `Expired`. System shows error: `"SMS code has expired. Please initiate a new transfer"`.   
- **Postconditions:** Transfer status marked as Cancelled/Expired; balance remains unchanged.   

### TR-SMS-003 (SMS Flow — 3 Wrong Attempts Cancellation)

- **Traces to:** REQ-SMS-03 (Three wrong attempts rule)   
- **Preconditions:** Test user TC-007 logged in. Transfer initiated for `250000` KZT (State: `Waiting, attempt 1`).   
- **Inputs / Steps:**
  1. Enter invalid OTP `000000` $\rightarrow$ System transitions to `Waiting, attempt 2`.   
  2. Enter invalid OTP `111111` $\rightarrow$ System transitions to `Waiting, attempt 3`.   
  3. Enter invalid OTP `222222` $\rightarrow$ Tap "Confirm".   
- **Expected Result:** On 3rd failed attempt, state transitions to `Cancelled / Blocked`. System displays error: `"Transfer cancelled due to 3 incorrect SMS code attempts"`.   
- **Postconditions:** Transfer session terminated; account balance unchanged.   

## 3. Traceability Matrix

| **Requirement ID** | **Requirement Description**                    | **Covered by Test Case(s)**                    | **Status / Coverage Notes** |
| ------------------ | ---------------------------------------------- | ---------------------------------------------- | --------------------------- |
| **REQ-AMT-01**     | Amount must be 100 to 500,000 KZT per transfer | TR-AMT-001, TR-AMT-002, TR-AMT-003, TR-AMT-004 |                             |

**Covered**

| **REQ-AMT-02** | Whole tenge only (no fractional amounts) | TR-AMT-005, TR-AMT-006 |   |
| -------------- | ---------------------------------------- | ---------------------- | - |

**Covered**

| **REQ-LIM-01** | Daily limit 1,000,000 KZT across all transfers | TR-APP-001, TR-APP-002 |   |
| -------------- | ---------------------------------------------- | ---------------------- | - |

**Covered**

| **REQ-LIM-02** | Daily limit resets at midnight Almaty time | — |   |
| -------------- | ------------------------------------------ | - | - |

**UNCOVERED** *(See justification below)*

| **REQ-SMS-01** | Transfers > 100,000 KZT require 6-digit SMS code | TR-AMT-004, TR-APP-001, TR-SMS-001 |   |
| -------------- | ------------------------------------------------ | ---------------------------------- | - |

**Covered**

| **REQ-SMS-02** | SMS code is valid for 120 seconds | TR-SMS-001, TR-SMS-002 |   |
| -------------- | --------------------------------- | ---------------------- | - |

**Covered**

| **REQ-SMS-03** | Three wrong attempts cancel the transfer | TR-SMS-003 |   |
| -------------- | ---------------------------------------- | ---------- | - |

**Covered**

| **REQ-ERR-01** | Amount error shown if both amount and daily limit fail | TR-APP-003 |   |
| -------------- | ------------------------------------------------------ | ---------- | - |

**Covered**

### Uncovered Requirement Justification

- **REQ-LIM-02 (Daily limit reset at midnight Almaty time):**
  - **Reason for Lack of Coverage:** The specification does not define server time synchronization mocking behavior or the handling of transfers spanning midnight (e.g., initiated at 23:59:55 and confirmed at 00:00:05). Without environment specifications for controlling test server time, this requirement cannot be reliably validated deterministically in manual test runs.   

## 4. Release Checklist (SMS Code Flow)

- [ ] **[ ] Checklist Item 1:** SMS OTP trigger threshold is strictly set to $> 100,000$ KZT (transfer of exactly $100,000$ KZT executes immediately without SMS).   
- [ ] **[ ] Checklist Item 2:** Generated SMS OTP is exactly 6 numeric digits.   
- [ ] **[ ] Checklist Item 3:** SMS OTP delivery time on staging/production environment is under 15 seconds.   
- [ ] **[ ] Checklist Item 4:** OTP timer countdown UI accurately displays 120-second expiration.   
- [ ] **[ ] Checklist Item 5:** Submitting correct OTP at second 119 successfully confirms transfer.   
- [ ] **[ ] Checklist Item 6:** Submitting correct OTP at second 121 is rejected with "Expired code" error.   
- [ ] **[ ] Checklist Item 7:** Third incorrect OTP submission immediately cancels transfer and locks OTP session.   
- [ ] **[ ] Checklist Item 8:** Attempt counter resets correctly upon launching a completely new transfer session.   
- [ ] **[ ] Checklist Item 9:** Double-tapping "Confirm" button dispatches only one single network request (idempotency enforced).   
- [ ] **[ ] Checklist Item 10:** SMS notification templates are verified across English, Kazakh, and Russian localizations.   
- [ ] **[ ] Checklist Item 11:** OTP code digits are masked/hidden in application logs and analytics tools.   
- [ ] **[ ] Checklist Item 12:** Database transaction rolls back completely on expired or cancelled SMS attempts.   

## 5. Three Defect Reports

### Defect 1: Critical — SMS OTP Accepted After Expiration Timeout

- **Title:** `[SMS Verification]: Transfer approved when entering valid OTP after 120-second expiration period`

     
- **Environment:** iOS 18.2, App Version 4.11.0 (Build 2291), Staging Env 2   
- **Preconditions:** Test account TC-014, Balance = 1,000,000 KZT, Daily spent = 0 KZT.   
- **Steps to Reproduce:**
  1. Open Transfers $\rightarrow$ Between my accounts.   
  2. Enter amount `150000` KZT and tap Continue.   
  3. Wait 125 seconds on the SMS Verification screen (timer reaches 00:00).   
  4. Enter the original 6-digit SMS code received in step 2.   
  5. Tap "Confirm".   
- **Expected Result:** System rejects the submission with error `"SMS code has expired"` and transitions session to `Expired` state. Balance remains 1,000,000 KZT.   
- **Actual Result:** System accepts the expired code, shows "Transfer successful", and debits 150,000 KZT from balance.   
- **Reproducibility:** 5 of 5 attempts   
- **Severity:** High (Core security rule bypass)   
- **Priority:** High (Must fix before release)   
- **Evidence:** Screen recording `expired_otp_bypass.mp4`, Server Request ID `req-88912-qzt`

     
- **Traces to:** REQ-SMS-02   

### Defect 2: Major — System Accepts Fractional Amount Without Throwing Validation Error

- **Title:** `[Amount Input]: Fractional tenge amount (100.50 KZT) processed without validation error`

     
- **Environment:** Android 14, App Version 4.11.0 (Build 2295), Staging Env 1   
- **Preconditions:** Test account TC-020, Balance = 500,000 KZT.   
- **Steps to Reproduce:**
  1. Open Transfers screen.   
  2. Enter amount `100.50` KZT.   
  3. Tap "Continue".   
- **Expected Result:** Inline validation error `"Amount must be a whole tenge number"` is shown. Transfer cannot proceed.   
- **Actual Result:** System accepts input `100.50`, processes transfer immediately, and debits `100.50` KZT from balance.   
- **Reproducibility:** 5 of 5 attempts   
- **Severity:** Medium (Business logic requirement violation)   
- **Priority:** Medium (Fix in current sprint)   
- **Evidence:** Screenshot `fractional_amount_accepted.png`

     
- **Traces to:** REQ-AMT-02   

### Defect 3: Medium (Missing Req / Implementation Flaw) — Daily Limit Not Reset at Midnight Almaty Time

- **Title:** `[Daily Limit]: Cumulative daily spent counter does not reset at 00:00 Almaty time`

     
- **Environment:** Web App (Chrome 128.0), Test Env 3   
- **Preconditions:** Test account TC-099, Daily spent accumulated on previous day = 900,000 KZT at 23:55 Almaty time.   
- **Steps to Reproduce:**
  1. Wait until server time passes midnight (00:05 Almaty time next calendar day).   
  2. Open Transfers screen.   
  3. Enter transfer amount `200000` KZT.   
  4. Tap "Continue".   
- **Expected Result:** Daily limit counter resets to 0 KZT at midnight. Transfer of 200,000 KZT is allowed.   
- **Actual Result:** Transfer is rejected with error `"Daily transfer limit of 1,000,000 KZT exceeded"` (system still counts yesterday's 900,000 KZT).   
- **Reproducibility:** 3 of 3 attempts across midnight transition   
- **Severity:** Medium (User experience impact / valid transfers blocked)   
- **Priority:** Medium (Fix before next release iteration)   
- **Evidence:** API Response payload `{"error": "DAILY_LIMIT_EXCEEDED", "cumulative": 1100000}` at timestamp `00:05:12+05:00`

     
- **Traces to:** REQ-LIM-02   

## 6. AI Appendix (Level 1)

### 6.1 Prompts Used

```
Prompt 1:
"You are a senior QA engineer. Here is a requirement: Amount 100 to 500,000 KZT per transfer, whole tenge only. Daily limit 1,000,000 KZT. Transfers above 100,000 KZT require 6-digit SMS code valid for 120s. 3 wrong attempts cancel transfer. If amount and daily limit are both violated, show amount error.
Perform ambiguity analysis, equivalence partitioning, 3-value BVA, decision table for approval, and state table for SMS code."

Prompt 2:
"Here are my raw notes from a test session on Money Transfer:
- Inputted 150000 KZT, waited 125s on SMS screen, entered code, transfer went through and debited money.
- Inputted 100.50 KZT, no error shown, debited 100.50 KZT.
Turn them into defect reports with fields: Title, Environment, Preconditions, Steps to reproduce, Expected result, Actual result, Reproducibility, Severity, Priority, Evidence, Traces to.
Rule: Use only facts present in notes. If a field is missing, write MISSING."
```

### 6.2 Raw Output Excerpt
```text
Raw Output Excerpt (from Prompt 2):
Title: [SMS Screen]: Transfer processed after waiting 125s
Environment: MISSING
Preconditions: MISSING
Steps to reproduce:
1. Input 150000 KZT.
2. Wait 125s on SMS screen.
3. Enter code.
Expected result: Error shown.
Actual result: Transfer went through and debited money.
Severity: Proposed High.
Priority: Proposed High.
```

### 6.3 What Was Changed and Why
1. **Filled `MISSING` Fields in Defect Reports:**
   * *Change:* Replaced `MISSING` in Environment, Preconditions, and Evidence with concrete test data (e.g., `iOS 18.2`, `Test Account TC-014`, `Request ID req-88912-qzt`).
   * *Why:* Per Week 4 rules, AI must not invent factual test environment details. The tester must supply real, reproducible environment details.
2. **Corrected Equivalence Partition Test Value:**
   * *Change:* Changed the invalid partition test value for $>500,000$ KZT from `500,500` to `500,001`.
   * *Why:* `500,500` contained two invalid variables simultaneously (out-of-bounds + non-integer), violating the *"One invalid value per test"* rule.
3. **Refactored Decision Table & Justified Dashes:**
   * *Change:* Explicitly populated action `A1` ($X$) for rules R7/R8 where both Amount and Daily Limit were violated, and added textual justifications for all collapsed dashes (`—`).
   * *Why:* To ensure the table strictly adheres to the precedence rule specified in requirements and satisfies ISO/IEC/IEEE 29119-3 standards.
```