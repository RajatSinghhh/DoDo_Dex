# DoDo Dex Security Audit – Findings Overview

This repository aggregates the **individual vulnerability reports** uncovered during the audit of the protocol’s smart‑contract suite. Each standalone report (located in the same directory) provides an in‑depth description, reproduction steps, recommended fixes, and status tracking for the issue.

---

## 📑 Table of Contents

| ID | Finding Title                                                                          | Severity<sup>†</sup> | Link                                                      |
| -- | -------------------------------------------------------------------------------------- | -------------------- | --------------------------------------------------------- |
| 1  | Anyone Can Trigger Withdrawals from the Gateway on Behalf of Any User                  | High                 | [Report 1](./finding-1-anyone-can-trigger-withdrawals.md) |
| 2  | ETH Transfer via `_handleETHDeposit()` Fails Due to Payability Mismatch                | Medium               | [Report 2](./finding-2-handle-eth-deposit-payability.md)  |
| 3  | `onCall()` Not Marked **payable** Causes `msg.value` to be Zero in Internal Swap Logic | Medium               | [Report 3](./finding-3-oncall-not-payable.md)             |
| 4  | Incorrect Fee Handling on DoDo Router Swaps in `onCall` Function                       | Low                  | [Report 4](./finding-4-dodo-router-fees.md)               |
| 5  | Owner Loses Access to Deposited Tokens (No Retrieval Mechanism)                        | High                 | [Report 5](./finding-5-owner-no-retrieval.md)             |
| 6  | Missing Setter for `gasLimit` in `GatewayCrossChain.sol` Causes Rigid Configuration    | Low                  | [Report 6](./finding-6-gaslimit-immutable.md)             |
| 7  | Incorrect ETH Accounting Due to Unchecked `msg.value` in `withdrawToNativeChain()`     | Medium               | [Report 7](./finding-7-withdraw-eth-accounting.md)        |

> <sup>†</sup> *The severity levels are provisional; consult each report for the final classification.*

---

## 1 Anyone Can Trigger Withdrawals from the Gateway on Behalf of Any User

* **Risk:** Unauthorized asset withdrawals leading to potential loss of funds.
* **Impacted Contract(s):** `Gateway.sol`
* **Fix Status:** *See detailed report*
* 👉 [Full report](./finding-1-anyone-can-trigger-withdrawals.md)

## 2 ETH Transfer via `_handleETHDeposit()` Fails Due to Payability Mismatch

* **Risk:** ETH deposits revert, blocking user onboarding.
* **Impacted Contract(s):** `GatewaySend.sol`
* 👉 [Full report](./finding-2-handle-eth-deposit-payability.md)

## 3 `onCall()` Not Marked *payable* Causes `msg.value` = 0

* **Risk:** Internal swap logic receives zero ETH, causing faulty execution paths.
* **Impacted Contract(s):** `SwapRouter.sol`
* 👉 [Full report](./finding-3-oncall-not-payable.md)

## 4 Incorrect Fee Handling on DoDo Router Swaps in `onCall`

* **Risk:** Fee mis‑calculation results in value loss for users or protocol.
* **Impacted Contract(s):** `SwapRouter.sol`, external DoDo integration
* 👉 [Full report](./finding-4-dodo-router-fees.md)

## 5 Owner Cannot Retrieve Deposited Tokens

* **Risk:** Owner‑controlled treasury assets become permanently locked.
* **Impacted Contract(s):** `Treasury.sol`
* 👉 [Full report](./finding-5-owner-no-retrieval.md)

## 6 No Setter for `gasLimit` in `GatewayCrossChain.sol`

* **Risk:** Inflexible gas configuration can break cross‑chain messaging as network conditions evolve.
* 👉 [Full report](./finding-6-gaslimit-immutable.md)

## 7 Unchecked `msg.value` in `withdrawToNativeChain()` Leads to Incorrect Accounting

* **Risk:** ETH balance discrepancies; potential fund misappropriation.
* 👉 [Full report](./finding-7-withdraw-eth-accounting.md)

---

### 📂 Directory Structure

```
.
├── finding-1-anyone-can-trigger-withdrawals.md
├── finding-2-handle-eth-deposit-payability.md
├── finding-3-oncall-not-payable.md
├── finding-4-dodo-router-fees.md
├── finding-5-owner-no-retrieval.md
├── finding-6-gaslimit-immutable.md
├── finding-7-withdraw-eth-accounting.md
└── README.md   ← (you are here)
```

### ✍️ Contributing

If you identify additional vulnerabilities or improvements, please open an issue or submit a pull request. Security disclosures can be sent directly to *[security@auditdrop.com](mailto:security@auditdrop.com)*.

---

© <?= date('Y') ?> <?= $ownerName ?> – All rights reserved.
