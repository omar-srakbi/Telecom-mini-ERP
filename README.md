# Telecom-mini-ERP
# UPCOMING

## Introduction

A Hybrid Distributed Mini-ERP system for Financial & Telecom Service Centers operating across multiple branches. The system follows an Offline-First architecture with asynchronous background synchronization to a central SQL Server via Radmin VPN. It manages multi-safe treasury operations, dual-currency financials, telecom services (Syriatel/MTN), internet ISP invoicing, HR/expenses, and a comprehensive RBAC security model. The UI targets low-resource hardware (2GB RAM) and supports Arabic (RTL) and English.

## Glossary

- **ERP**: Enterprise Resource Planning system
- **Branch**: A physical service center location running a local SQLite instance
- **Central_Server**: The central SQL Server instance that aggregates data from all branches
- **Sync_Engine**: The background worker responsible for replicating local SQLite changes to Central_Server
- **Safe**: An isolated treasury account with its own balance and permission set
- **Exchange_Rate**: A Buy/Sell rate pair for a given currency relative to USD
- **SYP**: Syrian Pound, the local currency
- **USD**: US Dollar, the base currency
- **RBAC**: Role-Based Access Control
- **EOD**: End of Day — the daily cash reconciliation process
- **ISP**: Internet Service Provider (24 providers managed in the Invoices Safe)
- **Audit_Log**: An immutable record of every system action
- **Transfer_Handshake**: A two-step confirmation protocol for inter-safe or inter-branch transfers
- **Blind_EOD**: An EOD process where the employee enters physical cash counts without seeing the system balance
- **VPN**: Virtual Private Network (Radmin VPN used for branch connectivity)
- **PBT**: Property-Based Testing
- **Sync_Queue**: A local queue of pending changes awaiting replication to Central_Server

---

## Requirements

### Requirement 1: Dual Currency & Exchange Rate Management

**User Story:** As a branch cashier, I want to process transactions in both USD and SYP using dynamic exchange rates, so that customers can pay in either currency at the current rate.

#### Acceptance Criteria

1. THE Exchange_Rate_Manager SHALL maintain a Buy Rate and a Sell Rate for SYP relative to USD.
2. WHEN a transaction amount is entered in SYP, THE Exchange_Rate_Manager SHALL convert it to USD using the current active Sell Rate.
3. WHEN a transaction amount is entered in USD, THE Exchange_Rate_Manager SHALL convert it to SYP using the current active Buy Rate.
4. WHEN an Admin updates an exchange rate with the Global Broadcast flag enabled, THE Sync_Engine SHALL propagate the new rate to all branch nodes within the next sync cycle.
5. WHEN an Admin updates an exchange rate with the Global Broadcast flag disabled, THE Exchange_Rate_Manager SHALL apply the new rate only to the local branch.
6. IF a transaction is initiated and no active exchange rate exists, THEN THE Exchange_Rate_Manager SHALL reject the transaction and return an error indicating no active rate is configured.
7. THE Exchange_Rate_Manager SHALL record a timestamped history entry for every rate change, including the previous rate, new rate, operator identity, and broadcast flag value.
8. WHILE a Global Broadcast is in progress, THE Sync_Engine SHALL lock rate updates on receiving branches until the broadcast is fully applied.

---

### Requirement 2: Multi-Safe Treasury Management

**User Story:** As a treasury manager, I want to manage six isolated safes with independent balances and transaction histories, so that funds across different business lines remain segregated and auditable.

#### Acceptance Criteria

1. THE Treasury_Manager SHALL maintain exactly six safes: Wholesale, Syriatel, MTN, Lines_SimCard, Sham_Cash, and Invoices.
2. WHEN a deposit is made to a safe, THE Treasury_Manager SHALL increase that safe's balance by the deposited amount and record the transaction.
3. WHEN a withdrawal is made from a safe, THE Treasury_Manager SHALL decrease that safe's balance by the withdrawn amount and record the transaction.
4. IF a withdrawal is requested from a safe and the safe's balance is insufficient, THEN THE Treasury_Manager SHALL reject the withdrawal and return an insufficient-funds error.
5. WHEN a transfer is initiated between two safes, THE Treasury_Manager SHALL require a Transfer_Handshake before committing the transfer.
6. THE Lines_SimCard_Safe SHALL track physical SIM card inventory (MTN/Syriatel new, replacement, and Golden lines) alongside digital cash balances.
7. THE Invoices_Safe SHALL apply a 10% margin to all 24 ISP transactions, except for the "Lima" ISP which SHALL apply a 20% margin (10% supplier, 10% house).
8. THE Sham_Cash_Safe SHALL support currency exchange operations and supplier settlement for accessories.
9. WHEN a Syriatel Cash or MTN Cash digital payment is processed, THE Treasury_Manager SHALL record it against the corresponding safe (Syriatel or MTN respectively).
10. THE Treasury_Manager SHALL track external bank payments via Bimo Bank for University fees, Passport fees, and Education fees as distinct transaction sub-types.
11. WHEN a safe balance is queried, THE Treasury_Manager SHALL return the balance in both USD and SYP using the current active exchange rate.

---

### Requirement 3: Transfer Handshake Protocol

**User Story:** As a branch manager, I want all inter-safe and inter-branch transfers to require two-step confirmation, so that no funds move without explicit acknowledgment from both parties.

#### Acceptance Criteria

1. WHEN a sender initiates a transfer, THE Transfer_Handshake_Service SHALL create a pending transfer record with status "Pending" and notify the designated receiver.
2. WHEN a receiver confirms a pending transfer, THE Transfer_Handshake_Service SHALL atomically debit the sender's safe and credit the receiver's safe, then update the transfer status to "Completed".
3. IF a receiver rejects a pending transfer, THEN THE Transfer_Handshake_Service SHALL update the transfer status to "Rejected" and restore any reserved funds to the sender's safe.
4. WHEN a pending transfer has not been confirmed or rejected within 24 hours, THE Transfer_Handshake_Service SHALL automatically expire it and set the status to "Expired".
5. THE Transfer_Handshake_Service SHALL prevent a sender from initiating a new transfer from the same safe while a pending transfer from that safe is outstanding.
6. WHILE a transfer is in "Pending" status, THE Treasury_Manager SHALL reserve the transfer amount from the sender's available balance.

---

### Requirement 4: Role-Based Access Control (RBAC)

**User Story:** As a system administrator, I want to assign granular permissions per user per safe, so that employees can only access the safes and operations they are authorized for.

#### Acceptance Criteria

1. THE RBAC_Manager SHALL support the following permission types per safe: View, Receive, Pay, Transfer, Exchange, and View_Balance.
2. WHEN a user attempts an operation on a safe, THE RBAC_Manager SHALL verify the user holds the required permission for that safe before allowing the operation.
3. IF a user attempts an operation they are not authorized for, THEN THE RBAC_Manager SHALL deny the operation, log an unauthorized-access attempt in the Audit_Log, and return an access-denied error.
4. THE RBAC_Manager SHALL support restricting individual users to SYP-only transactions, preventing them from initiating or viewing USD amounts.
5. WHERE a user is restricted to SYP-only, THE RBAC_Manager SHALL hide USD fields and reject any USD transaction attempts from that user.
6. THE RBAC_Manager SHALL support an Admin role with unrestricted access to all safes and all operations.
7. WHEN an Admin modifies a user's permissions, THE RBAC_Manager SHALL apply the new permissions immediately for all subsequent operations by that user.
8. THE RBAC_Manager SHALL prevent any user, including Admins, from deleting Audit_Log entries.

---

### Requirement 5: Blind End-of-Day (EOD) Reconciliation

**User Story:** As a branch manager, I want employees to perform end-of-day cash counts without seeing the system balance, so that discrepancies are detected honestly and reported to management.

#### Acceptance Criteria

1. WHEN an employee initiates an EOD session, THE EOD_Manager SHALL present a cash-entry form that does not display the system's expected balance for any safe.
2. WHEN an employee submits their physical cash count, THE EOD_Manager SHALL compare the submitted count against the system's calculated balance and compute the discrepancy.
3. IF a discrepancy exists between the submitted count and the system balance, THEN THE EOD_Manager SHALL record the discrepancy and generate an alert visible only to Admin-role users.
4. WHEN an EOD session is completed, THE EOD_Manager SHALL lock the session and prevent further modification of the submitted counts.
5. THE EOD_Manager SHALL record the employee identity, timestamp, per-safe submitted amounts, system balances, and discrepancy values for each EOD session.
6. WHEN an Admin reviews an EOD session, THE EOD_Manager SHALL display both the employee-submitted counts and the system-calculated balances side by side.

---

### Requirement 6: Omniscient Audit Log

**User Story:** As a compliance officer, I want every system action recorded in an immutable audit log, so that all activity can be reviewed and unauthorized actions are traceable.

#### Acceptance Criteria

1. THE Audit_Logger SHALL record an entry for every: Login attempt (success or failure), Screen view, Button click, Data edit, Data delete, and Unauthorized access attempt.
2. WHEN an audit entry is created, THE Audit_Logger SHALL capture: timestamp, user identity, action type, affected entity type, affected entity ID, old value (if applicable), new value (if applicable), and branch ID.
3. THE Audit_Logger SHALL store audit entries in the local SQLite database for a minimum of 60 days.
4. WHEN an audit entry is 60 days old, THE Audit_Logger SHALL mark it eligible for local purge only after confirming it has been successfully synced to Central_Server.
5. THE Sync_Engine SHALL replicate all audit entries to Central_Server for long-term archiving regardless of their age.
6. THE Audit_Logger SHALL be append-only; no user or process SHALL be permitted to modify or delete audit entries.
7. WHEN a user views the audit log, THE RBAC_Manager SHALL restrict the visible entries to those within the user's authorized scope unless the user holds Admin role.

---

### Requirement 7: Offline-First Sync Engine

**User Story:** As a branch operator, I want all transactions to be saved locally first and synced to the central server in the background, so that operations continue uninterrupted during network outages.

#### Acceptance Criteria

1. WHEN a transaction is committed, THE Sync_Engine SHALL first persist it to the local SQLite database before returning a success response to the UI.
2. WHEN network connectivity to Central_Server is available, THE Sync_Engine SHALL process the Sync_Queue and replicate pending records to Central_Server in the background without blocking the UI.
3. WHEN network connectivity is unavailable, THE Sync_Engine SHALL continue queuing local transactions and retry synchronization when connectivity is restored.
4. THE Sync_Engine SHALL detect and resolve conflicts using a last-write-wins strategy based on the transaction timestamp, with Admin notification for any conflict resolved.
5. THE Sync_Engine SHALL operate within a bandwidth budget of 2 Mbps and compress payloads before transmission.
6. WHEN a sync batch is successfully committed to Central_Server, THE Sync_Engine SHALL mark the corresponding Sync_Queue entries as "Synced" and record the sync timestamp.
7. IF a sync batch fails to commit to Central_Server, THEN THE Sync_Engine SHALL retain the batch in the Sync_Queue and retry with exponential backoff (initial delay 30 seconds, max delay 30 minutes).
8. THE Sync_Engine SHALL provide a status indicator visible in the UI showing: Connected, Syncing, Pending (with count), or Disconnected.
9. WHEN a Global Broadcast exchange rate update is received from Central_Server, THE Sync_Engine SHALL apply it to the local Exchange_Rate_Manager before processing any subsequent transactions.

---

### Requirement 8: ISP Invoice & Margin Management

**User Story:** As an invoicing clerk, I want to manage internet service provider invoices with automatic margin calculation, so that profitability is tracked accurately per ISP.

#### Acceptance Criteria

1. THE Invoice_Manager SHALL maintain a registry of exactly 24 ISP providers.
2. WHEN an ISP invoice is processed, THE Invoice_Manager SHALL apply a 10% margin to the invoice cost for all ISPs except "Lima".
3. WHEN a "Lima" ISP invoice is processed, THE Invoice_Manager SHALL apply a 20% total margin, split as 10% supplier margin and 10% house margin.
4. THE Invoice_Manager SHALL calculate the customer-facing price as: cost + applicable margin.
5. WHEN an invoice is recorded, THE Invoice_Manager SHALL associate it with the Invoices_Safe and update the safe balance accordingly.
6. THE Invoice_Manager SHALL generate a per-ISP profitability report showing total invoiced amount, total cost, and total margin earned.
7. IF an ISP is not found in the registry, THEN THE Invoice_Manager SHALL reject the invoice and return an unknown-ISP error.

---

### Requirement 9: SIM Card & Physical Inventory Management

**User Story:** As a lines manager, I want to track physical SIM card inventory for MTN and Syriatel, so that stock levels and sales are accurately recorded.

#### Acceptance Criteria

1. THE Inventory_Manager SHALL track the following SIM card categories: MTN New, MTN Replacement, MTN Golden, Syriatel New, Syriatel Replacement, and Syriatel Golden.
2. WHEN a SIM card is received into stock, THE Inventory_Manager SHALL increase the count for the corresponding category.
3. WHEN a SIM card is sold or issued, THE Inventory_Manager SHALL decrease the count for the corresponding category and record the transaction against the Lines_SimCard_Safe.
4. IF a SIM card sale is requested for a category with zero stock, THEN THE Inventory_Manager SHALL reject the sale and return an out-of-stock error.
5. THE Inventory_Manager SHALL provide a current stock report showing counts per category.
6. WHEN stock for any SIM card category falls below a configurable minimum threshold, THE Inventory_Manager SHALL generate a low-stock alert.

---

### Requirement 10: HR & Expenses Module

**User Story:** As an HR administrator, I want to track employee financial transactions including salaries, advances, and deductions, so that payroll and expense management are accurate.

#### Acceptance Criteria

1. THE HR_Manager SHALL record employee withdrawals, electricity bills, salary payments, salary advances, and vacation records.
2. WHEN a salary advance is recorded for an employee, THE HR_Manager SHALL deduct the advance amount from the employee's next monthly salary calculation.
3. WHEN a monthly salary is processed, THE HR_Manager SHALL compute the net salary as: gross salary minus all outstanding advances for that month.
4. THE HR_Manager SHALL maintain a per-employee ledger showing all financial transactions chronologically.
5. WHEN a vacation is recorded, THE HR_Manager SHALL store the start date, end date, employee identity, and vacation type.
6. THE HR_Manager SHALL generate a monthly expense summary report covering total salaries paid, total advances outstanding, total electricity costs, and total other withdrawals.

---

### Requirement 11: Multi-Language & RTL UI

**User Story:** As an Arabic-speaking operator, I want the UI to fully support Arabic with right-to-left layout, so that I can use the system comfortably in my native language.

#### Acceptance Criteria

1. THE UI_Framework SHALL support English and Arabic as selectable display languages.
2. WHEN Arabic is selected as the display language, THE UI_Framework SHALL render all text, labels, and menus in Arabic and apply RTL layout direction.
3. WHEN English is selected as the display language, THE UI_Framework SHALL render all text, labels, and menus in English and apply LTR layout direction.
4. THE UI_Framework SHALL persist the selected language preference per user account.
5. THE UI_Framework SHALL render correctly on hardware with 2GB RAM without exceeding 300MB of application memory under normal operation.
6. WHEN the display language is changed, THE UI_Framework SHALL apply the new language and layout direction without requiring an application restart.

---

### Requirement 12: Low-Bandwidth & Performance Optimization

**User Story:** As a branch operator on a slow connection, I want the sync and UI to remain responsive under 2 Mbps bandwidth, so that daily operations are not disrupted by network limitations.

#### Acceptance Criteria

1. THE Sync_Engine SHALL compress all sync payloads using a lossless compression algorithm before transmission.
2. THE Sync_Engine SHALL transmit sync batches in chunks not exceeding 256KB per request to avoid saturating a 2 Mbps link.
3. THE UI_Framework SHALL remain responsive (input latency under 200ms) while the Sync_Engine is actively syncing in the background.
4. WHEN the Sync_Engine detects available bandwidth below 512 Kbps, THE Sync_Engine SHALL reduce sync frequency to avoid degrading user experience.
5. THE Sync_Engine SHALL prioritize syncing financial transactions and exchange rate updates over audit log entries when bandwidth is constrained.
