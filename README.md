# Intuit Stablecoin Wallet & Crypto Investing — System Design

**Author:** Mahendra Maraka | Principal Engineer  
**Case Study:** Intuit's Stablecoin Wallet and Crypto Investing  
**Scope:** Global stablecoin wallet + crypto investing as a shared platform across QuickBooks, TurboTax, Credit Karma, and MailChimp

---

## Table of Contents

1. [What We Are Building](#1-what-we-are-building)
2. [Who We Are Building For](#2-who-we-are-building-for)
3. [System Overview](#3-system-overview)
4. [Component: Wallet Service](#4-component-wallet-service)
5. [Component: Transaction Service](#5-component-transaction-service)
6. [Component: Ledger Service](#6-component-ledger-service)
7. [Component: Stablecoin & Blockchain Gateway](#7-component-stablecoin--blockchain-gateway)
8. [Component: QR Code Payments (X9A Standard)](#8-component-qr-code-payments-x9a-standard)
9. [Component: Crypto Investing Service](#9-component-crypto-investing-service)
10. [Component: Price Feed Service](#10-component-price-feed-service)
11. [Component: Tax Engine & TurboTax Sync](#11-component-tax-engine--turbotax-sync)
12. [Component: Fraud Detection](#12-component-fraud-detection)
13. [Component: Compliance Service](#13-component-compliance-service)
14. [Component: Reconciliation Service](#14-component-reconciliation-service)
15. [Data Model — Every Table](#15-data-model--every-table)
16. [How Everything Connects — End-to-End Flows](#16-how-everything-connects--end-to-end-flows)
17. [Scale & Reliability](#17-scale--reliability)
18. [Security Architecture](#18-security-architecture)
19. [Telemetry & Monitoring](#19-telemetry--monitoring)
20. [Disaster Recovery](#20-disaster-recovery)
21. [Engineering Practices](#21-engineering-practices)
22. [Key Decisions & Trade-offs](#22-key-decisions--trade-offs)

---

## 1. What We Are Building

### Product 1: Stablecoin Wallet

A stablecoin is a digital currency worth exactly $1.00 — always. Unlike Bitcoin, the price never changes. Think of it as a digital dollar that moves like a text message.

The stablecoin wallet lets any Intuit user:
- Hold a balance in USDC, USDT, or PYUSD (three major stablecoins)
- Send money to any other Intuit user instantly
- Receive payments from anyone inside the Intuit ecosystem
- Pay businesses by scanning or showing a QR code (X9A standard)
- Have payments automatically recorded in QuickBooks or TurboTax

**Why stablecoin instead of a regular bank transfer?**

| What | Bank Wire | Stablecoin |
|---|---|---|
| Time to arrive | 1–3 business days | ~90 seconds |
| Cost | $15–45 flat fee | ~$0.01–2.00 |
| Available hours | Business hours only | 24/7/365 |
| International FX fee | 2–3% | None (USD-pegged) |
| Auto-record in QuickBooks | Manual entry | Automatic |

### Product 2: Crypto Investing

Crypto investing lets users buy, hold, and sell Bitcoin and Ethereum directly inside the Intuit wallet — without needing a Coinbase account. The crypto is held safely on their behalf by an institutional custodian (BitGo).

**Intuit's unique advantage:** TurboTax already has 50 million users who file crypto taxes manually today. This product makes that automatic — every trade is tracked, every gain is calculated, and the IRS form (1099-DA) is pre-filled in TurboTax. No competitor can offer this.

---

## 2. Who We Are Building For

| Person | Product | Problem Today | Wallet Solves |
|---|---|---|---|
| Maria (small business owner) | QuickBooks | Paying a supplier in Mexico costs $345 in fees and takes 3 days | 90-second USDC transfer, near-zero cost, auto-recorded in QB |
| James (individual filer) | TurboTax | Tax refund takes 3 weeks; crypto taxes require hours of manual entry | Refund in seconds; crypto taxes auto-calculated and pre-filled |
| Priya (individual consumer) | Credit Karma | Venmo charges 1.75% for instant transfers | Near-zero fee P2P; buy crypto in 3 taps |
| Tom (merchant) | MailChimp | Card processing takes 2–3% per transaction + next-day settlement | QR code scan → payment in 90 seconds, zero fee, auto-sync to QB |

---

## 3. System Overview

### 3.1 High-Level Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                      FOUR INTUIT PRODUCTS                           │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌───────────┐ │
│  │  QuickBooks  │ │  TurboTax    │ │ Credit Karma │ │MailChimp  │ │
│  │  (Wallet SDK)│ │  (Wallet SDK)│ │  (Wallet SDK)│ │(Wallet SDK│ │
│  └──────┬───────┘ └──────┬───────┘ └──────┬───────┘ └─────┬─────┘ │
└─────────┼───────────────┼────────────────┼────────────────┼────────┘
          │  HTTPS                          │                │
          ▼                                                   
┌──────────────────────────────────────────────────────────────────────┐
│                SECURITY GATEWAY (API Gateway)                        │
│  Verifies login · Enforces speed limits · Routes requests           │
└─────────────────────────────┬────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                      │
        ▼                     ▼                      ▼
┌──────────────┐  ┌───────────────────┐  ┌────────────────────┐
│WALLET SERVICE│  │TRANSACTION SERVICE│  │ COMPLIANCE SERVICE │
│Holds balances│  │Processes payments │  │KYC · Fraud · OFAC  │
└──────┬───────┘  └─────────┬─────────┘  └────────────────────┘
       │                    │
       ▼                    ▼
┌──────────────────────────────────────────────────────┐
│                LEDGER SERVICE                        │
│  Records every dollar in/out using accounting rules  │
└──────────────────────┬───────────────────────────────┘
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
┌──────────────┐ ┌──────────────┐ ┌───────────────────────┐
│  BLOCKCHAIN  │ │CRYPTO INVEST │ │   RECONCILIATION       │
│  GATEWAY     │ │  SERVICE     │ │   Nightly balance      │
│(Stablecoin)  │ │Buy/Hold/Sell │ │   matching with        │
└──────────────┘ └──────┬───────┘ │   payment partners     │
                         │         └───────────────────────┘
                         ▼
                ┌──────────────────────┐
                │  PRICE FEED SERVICE  │
                │  TAX ENGINE          │
                │  TURBOTAX SYNC       │
                └──────────────────────┘

SHARED INFRASTRUCTURE:
  Message Queue (Kafka) · Database (PostgreSQL) · Cache (Redis) · S3 (files)
```

### 3.2 Deployment

The system runs in two AWS regions simultaneously:
- **us-east-1** handles 80% of traffic normally
- **us-west-2** handles 20% normally, takes 100% automatically if east fails
- **Route 53 health checks** flip traffic within 60 seconds of a region failure
- **Database writes** are synchronously replicated — no data is ever lost on failover

### 3.3 How Services Talk to Each Other

| Connection | Protocol | Why |
|---|---|---|
| Product apps → Security Gateway | HTTPS REST | Every app understands REST; simple and universal |
| Gateway → Wallet Service | HTTPS REST | External-facing; REST stays consistent |
| Wallet Service → Transaction Service | gRPC | Internal, high-frequency, faster than REST |
| Transaction Service → Fraud Detection | gRPC | 50ms time limit — binary protocol saves 5–10ms |
| Transaction Service → Ledger Service | gRPC | Internal, high-frequency |
| Price Feed → Crypto Service | gRPC streaming | Prices push every 500ms — streaming is the right model |
| Any service → Socure / Sardine | HTTPS REST | Vendor provides REST API; we adapt |

**What is gRPC?** It is a way for services to call each other that is faster than standard web requests (REST). It uses a compact binary format instead of text, which matters when a service needs to make thousands of calls per second.

---

## 4. Component: Wallet Service

### What it does
Stores and manages user wallet accounts and balances. This is the single source of truth for how much money each user has.

### Key design decisions

**One balance row per currency** — instead of one balance column on the wallet record:

```sql
-- WRONG: adding a new stablecoin requires a schema change
wallets (wallet_id, usdc_balance, usdt_balance, pyusd_balance, btc_balance)

-- RIGHT: one row per currency, new currency = just a new row
wallet_balances (wallet_id, currency, available_balance, pending_balance)
-- 'pending_balance' = money reserved for a transfer that hasn't confirmed yet
-- 'available_balance' = what the user can actually use right now
```

**Why we separate 'available' from 'pending':**  
When Alice sends 50 USDC to Bob, the blockchain takes 90 seconds to confirm. During that 90 seconds, we must make sure Alice cannot spend that same 50 USDC again. We move it to `pending_balance` immediately, before the blockchain confirms. When confirmed, it's removed from pending. If the transaction fails, it moves back to available.

### Reading a balance — three options

| Approach | Speed | Accuracy | Used for |
|---|---|---|---|
| Read from cache (Redis) | ~2ms | Up to 5 seconds stale | Displaying balance to user |
| Read from database directly | ~8ms | Exact | Before initiating any transfer |
| Read from cache with warning | ~1ms | May be stale | Notification display only |

**Preventing the stampede problem (Thundering Herd):**  
If 100,000 users all check their balance at the same moment (e.g., April 15 midnight), and all their cache entries expired at the same time, all 100,000 requests hit the database at once. This causes a crash.

Solution: Add random expiry time variation. Instead of every cache entry expiring at exactly 5 seconds, each entry expires at a random time between 4 and 6 seconds. The load spreads out naturally.

```java
// Cache TTL with jitter — prevents synchronized cache expiry
int baseTtlSeconds = 5;
int jitter = (int)(Math.random() * 2) - 1;  // -1 to +1 second randomly
redis.setex(cacheKey, baseTtlSeconds + jitter, balanceValue);
```

**Preventing the cache refresh rush (Probabilistic Early Expiry):**  
An additional technique: we start refreshing the cache slightly before it expires, while still serving the old value to everyone else. Only one thread does the refresh; everyone else gets the still-valid old value.

---

## 5. Component: Transaction Service

### What it does
Processes all payment transactions — stablecoin transfers, QR payments, and crypto orders. It coordinates all the steps required to safely move money.

### The transfer process, step by step

```
User taps "Send $50 USDC"
  │
  ▼
1. Check for duplicate (has this exact request been seen before?)
   → Redis stores each unique request ID for 24 hours
   → If duplicate: return the original response, do nothing new
  │
  ▼
2. Ask Fraud Detection to review (gRPC call, 45ms timeout)
   → If fraud service takes too long: APPROVE anyway, don't block the user
   → If fraud service says DECLINE: reject with explanation
  │
  ▼
3. Lock both wallets in the database (prevents race conditions)
   → Always lock wallets in alphabetical order by ID
   → CRITICAL: this prevents deadlock (explained below)
  │
  ▼
4. Check available balance >= transfer amount
   → If not: return "Insufficient funds" error
  │
  ▼
5. In ONE database transaction (all succeed or all fail together):
   a. Deduct from sender's available_balance
   b. Add to receiver's available_balance
   c. Write two ledger entries (debit + credit)
   d. Write an "outbox event" — a message to send to the queue
  │
  ▼
6. Return transaction ID to user
   (blockchain confirmation happens in the background)
```

### Preventing deadlock

**The problem:** If Alice is sending to Bob while Bob is sending to Alice at the same time:
- Thread 1 locks Alice's wallet, waits for Bob's
- Thread 2 locks Bob's wallet, waits for Alice's
- Both are stuck forever (deadlock)

**The solution:** Always lock wallets in alphabetical order by wallet ID.  
Both threads will always try to lock the lower-ID wallet first. One succeeds; the other waits. No deadlock possible.

```java
// Deadlock prevention — always lock in consistent order
String firstId  = walletA.compareTo(walletB) < 0 ? walletA : walletB;
String secondId = walletA.compareTo(walletB) < 0 ? walletB : walletA;

// Both threads agree on this order, so only one gets both locks
lockWallet(firstId);
lockWallet(secondId);
// ... execute transfer
```

### Preventing duplicate payments (Idempotency)

If a user's app crashes after sending but before receiving confirmation, the app will retry. Without protection, the payment would be processed twice.

Solution: Every request carries a unique ID chosen by the client. The server stores this ID in Redis for 24 hours. If the same ID arrives again, return the original response — do not process again.

---

## 6. Component: Ledger Service

### What it does
Records every financial event permanently and accurately. Uses the same accounting principle as bookkeeping: double-entry accounting.

### Double-entry accounting explained

Every payment creates exactly two records:
- One record showing money leaving (DEBIT)
- One record showing money arriving (CREDIT)

The total of all DEBITs must equal the total of all CREDITs — always. If they don't match, something is wrong.

```sql
-- When Alice sends Bob 50 USDC:

INSERT INTO ledger_entries VALUES
  (txn_id, alice_wallet, 'DEBIT',  50.00, 'USDC', alice_balance_after),
  (txn_id, bob_wallet,   'CREDIT', 50.00, 'USDC', bob_balance_after);

-- Every night, we verify this invariant:
-- SUM of all DEBITS = SUM of all CREDITS
-- Any discrepancy triggers an immediate alert
SELECT entry_type, SUM(amount)
FROM ledger_entries
GROUP BY entry_type;
-- Must show: DEBIT total = CREDIT total
```

### Rules for the ledger

1. **Never update** a ledger entry after it is written
2. **Never delete** a ledger entry — ever
3. If a payment is reversed, write a new entry with the opposite sign — do not erase the original
4. Every entry includes a snapshot of the balance after the entry — this creates a verifiable audit trail

### The Outbox pattern — why it exists

**The problem we're solving:**  
When a payment is processed, we need to do two things:
1. Write to the database (the ledger)
2. Send a message to the queue (so the blockchain gateway knows to proceed)

If we write to the database, then try to send to the queue, and the queue send fails — the database has the record but the blockchain never gets told. The payment is stuck.

**The solution:**  
Write both things to the database in the same single operation. A background process reads the outbox and sends to the queue. Since both the ledger entry and the outbox entry are in the same database transaction, either both succeed or both fail — never one without the other.

```
Database Transaction:
  ├── Write ledger_entry (debit)
  ├── Write ledger_entry (credit)
  └── Write outbox_event (to be sent to queue)
  COMMIT — all three happen, or none happen

Background poller (every 100ms):
  Reads unsent outbox_events
  Sends each to Kafka queue
  Marks as sent in database
```

---

## 7. Component: Stablecoin & Blockchain Gateway

### What it does
Submits stablecoin transfers to the Ethereum or Solana blockchain, monitors for confirmation, and handles failures.

### How stablecoin transfers work on a blockchain

USDC (the stablecoin we use) is a "smart contract" on the Ethereum blockchain. A smart contract is a small program that lives on the blockchain and runs automatically when triggered.

When we send USDC:
1. We build an instruction: "move 50 USDC from address A to address B"
2. We digitally sign the instruction using a cryptographic key (stored in a hardware device — never in software)
3. We broadcast the signed instruction to the Ethereum network
4. Ethereum miners include the instruction in the next block (~12 seconds)
5. After 12 blocks (~2.5 minutes): the transfer is permanent and cannot be reversed

### Gas fees — the cost of blockchain transactions

Every Ethereum transaction pays a "gas fee" to the network. This fee fluctuates based on network demand — like surge pricing.

| Network | Typical fee | Confirmation time | When to use |
|---|---|---|---|
| Ethereum | $0.50–$5.00 | ~90 seconds | Most transfers (most trusted) |
| Solana | ~$0.001 | ~0.4 seconds | High-frequency, small amounts |
| Polygon | ~$0.01 | ~8 minutes | Large amounts, cost-sensitive |

**Handling gas spikes:**
```
Before submitting each transaction:
  1. Estimate the current gas price
  2. If gas price > user's configured maximum:
     → Queue the transaction
     → Notify user: "Transfer queued — network is busy. Will process when fees normalize."
     → Do NOT lose the money or fail silently
  3. If within range:
     → Submit with 20% buffer above estimate (prevents failure from price increase mid-submission)
```

### Circuit breaker — when the blockchain is slow

If the blockchain service is failing or slow, we should not keep sending requests. We use a circuit breaker:
- **Closed (normal):** requests pass through
- **Open (tripped):** after 50% of last 10 requests failed, stop sending for 30 seconds; return "pending" to users
- **Half-open (recovering):** try one request; if it succeeds, return to normal

This prevents a blockchain outage from cascading into a complete wallet outage.

### Failed transaction — what happens

```
Scenario: Transaction submitted to blockchain, but never confirmed (e.g., gas too low)

After 10 minutes without confirmation:
  1. Mark transaction as FAILED
  2. Reverse the balance change:
     - Move amount back from pending to available for the sender
     - Write a correcting ledger entry
  3. Notify the user: "Transfer could not complete. $50 USDC returned to your wallet. Reference: abc123"
  4. Log the incident for investigation

Result: User never loses money. Worst case is a failed transfer with a clear explanation.
```

---

## 8. Component: QR Code Payments (X9A Standard)

### What X9A is
X9A is an industry standard for encoding payment information in a QR code. It defines a consistent format so that any Intuit app can scan any Intuit QR code and know exactly what to do.

### How the QR code works

```
Tom (the merchant) opens MailChimp and generates a QR code:

QR Code contains:
{
  "merchant_wallet": "tom-wallet-id",
  "amount": "49.99",           ← locked amount, cannot be changed
  "currency": "USDC",
  "expires_at": "14:30:30",   ← QR code expires in 30 seconds
  "one_time_code": "uuid",     ← used once only, then invalid
  "signature": "..."            ← digital signature proving this is real
}

Priya (the customer) scans the QR code with her Credit Karma app:
  1. App reads and decodes the QR code
  2. Verifies the digital signature — confirms Tom's wallet really generated this
  3. Checks expiry — if more than 30 seconds old, rejects it
  4. Checks one-time code — if already used, rejects it (prevents copying the QR and reusing it)
  5. Shows Priya: "Pay $49.99 USDC to Tom's store?"
  6. Priya confirms → payment processes normally
```

### Why 30-second expiry and one-time codes?

**Without expiry:** Someone photographs Tom's QR code and uses it again tomorrow to steal from Tom.  
**Without one-time codes:** Someone photographs the QR code and pays multiple times with the same code.  
**With both:** A copied QR code is useless after 30 seconds and can only work once.

### The digital signature

The QR code is signed using ECDSA (a cryptographic signing method). Tom's app has a private key (stored securely). The signature is created from all the QR data using this key.

When Priya's app verifies the signature using Tom's public key:
- If any field was changed (amount, expiry, merchant ID) → signature is invalid → rejected
- If the signature is valid → the QR code is exactly what Tom intended → safe to pay

---

## 9. Component: Crypto Investing Service

### What it does
Allows users to buy, hold, and sell Bitcoin and Ethereum from inside their Intuit wallet. Manages orders through a custodian, tracks holdings, and prepares tax data.

### The custodian model — how it actually works

Intuit does not hold Bitcoin directly. Instead, Intuit partners with BitGo — an institutional custodian used by major financial institutions. BitGo holds the actual crypto in offline, insured storage.

```
User's perspective: "I have 0.002 BTC in my wallet"

Reality:
  ┌─────────────────────────────────────────┐
  │  Intuit's ledger                        │
  │  user_id: priya                         │
  │  asset: BTC                             │
  │  quantity: 0.002                        │  ← this is what Priya sees
  └──────────────────────────────────────┬──┘
                                         │ mirrors
  ┌──────────────────────────────────────▼──┐
  │  BitGo's vault                          │
  │  intuit_master_account                  │
  │  BTC balance includes Priya's 0.002     │  ← this is the actual crypto
  └─────────────────────────────────────────┘
```

**Why this model?**
- Users never need a private key or seed phrase — it just works like a bank account
- Crypto is insured (BitGo's coverage: $100M+)
- Regulatory compliance is far simpler — this is the same model banks use
- Users cannot lose their crypto by forgetting a password

### The buy flow

```
Priya taps "Buy $100 of Bitcoin"

Step 1: Price check
  → Look up current BTC price from cache (must be fresh — less than 5 seconds old)
  → Show Priya: "You will receive approximately 0.00231 BTC at $43,285"

Step 2: Verification
  → Is Priya's identity verified? (KYC)
  → Has she hit her daily crypto limit?
  → Fraud check

Step 3: Reserve funds
  → Move $100 from available_balance to pending_balance (cannot be double-spent)

Step 4: Place order
  → Send order to BitGo: "Buy $100 of BTC"
  → BitGo executes this on a real exchange (Coinbase Pro, Kraken)
  → Fill confirmed: 0.002312 BTC at $43,285.52, fee = $0.50

Step 5: Record everything
  → Deduct $100.50 from Priya's USDC balance (amount + fee)
  → Add 0.002312 to Priya's BTC holding
  → Create a tax lot record (see §11)

Step 6: Notify
  → Push notification: "You bought 0.002312 BTC at $43,285.52"
  → TurboTax receives the trade data
```

### Why we chose a custodian over self-custody

| | Custodian (BitGo) | Self-Custody |
|---|---|---|
| User experience | No private key, no seed phrase | User manages their own keys |
| If user forgets password | Account recovery via identity check | Crypto is lost forever |
| Insurance | $100M+ | None |
| Regulatory path | Works with existing money licenses | Requires additional exchange license (12–18 months) |
| **Decision** | **✅ Phase 1** | **Consider for advanced users in Phase 3** |

---

## 10. Component: Price Feed Service

### What it does
Provides accurate, real-time cryptocurrency prices to the wallet. Detects bad data automatically.

### Why multiple sources matter

If we used a single price source and it went down or gave a wrong price, users could:
- Buy crypto at the wrong price (costing them money)
- See wildly incorrect portfolio values
- Have orders rejected because the price looked invalid

**Solution:** Use three independent sources simultaneously. Display the middle (median) value. If any two disagree by more than 0.5%, halt new crypto orders and alert the team.

```
Every 500 milliseconds:

  Source 1 (Chainlink):   BTC = $43,282.10
  Source 2 (Coinbase):    BTC = $43,285.00
  Source 3 (Kraken):      BTC = $43,288.50

  Median = $43,285.00  ← this is what we use
  Max difference = $288.50 - $282.10 = $6.40
  $6.40 / $43,285 = 0.015% difference ← within 0.5% threshold, safe to use

  Store in cache with 500ms expiry.
```

### What happens when the price feed is stale

| Staleness | Action |
|---|---|
| 0–500ms | Normal — serve price as-is |
| 500ms–5s | Show price with "may be slightly delayed" warning |
| Over 5s | Stop accepting new crypto buy/sell orders. Show "Market data temporarily unavailable." Existing holdings are unaffected. |
| Over 60s | Escalate alert to on-call engineer |

---

## 11. Component: Tax Engine & TurboTax Sync

### What it does
Tracks the cost of every crypto purchase (for tax purposes), calculates gains and losses on sales, generates the IRS 1099-DA form, and pre-populates TurboTax returns.

### Why this matters

In the US, cryptocurrency is taxed as property. Every time you sell, trade, or spend crypto, you owe taxes on the profit. To calculate the profit, you need to know:
- What you paid for the crypto (cost basis)
- When you bought it (to determine short-term vs long-term rate)

This information is tracked per "tax lot" — each purchase creates a separate lot.

### Tax lots explained

```
Priya buys Bitcoin in three separate purchases:

Lot 1: January 15 — 0.001 BTC at $43,000 = $43.00
Lot 2: March 20 — 0.001 BTC at $50,000 = $50.00
Lot 3: June 10 — 0.001 BTC at $60,000 = $60.00

She sells 0.002 BTC in December at $70,000/BTC:
  Proceeds = 0.002 × $70,000 = $140.00

Using FIFO (First In, First Out — the IRS default method):
  Lot 1 (0.001 BTC, cost $43): Profit = $70 - $43 = $27.00 (held > 1 year = long-term rate)
  Lot 2 (0.001 BTC, cost $50): Profit = $70 - $50 = $20.00 (held < 1 year = short-term rate)

Tax calculation happens automatically. Priya never touches a spreadsheet.
```

### The 1099-DA — the new IRS form

Starting tax year 2025, the IRS requires every crypto custodian to issue Form 1099-DA — similar to the 1099-B form used for stock sales. It reports:
- Gross proceeds from all sales
- Cost basis for each sale
- Short-term and long-term classification
- Net gain or loss

Our system generates this automatically for every user. It then pushes the data to TurboTax through an internal API, so when the user opens TurboTax, their crypto taxes are already filled in.

**This is Intuit's moat:** No bank, exchange, or fintech can pre-populate a TurboTax return. Only Intuit can — because they own TurboTax.

---

## 12. Component: Fraud Detection

### What it does
Reviews every transaction before it processes, in under 50 milliseconds, and decides whether to approve, flag for review, or decline.

### How it works

The system uses a machine learning model that was trained on historical transaction patterns. For every new transaction, it receives:

| Feature | What it tells us |
|---|---|
| Transaction amount | Is this unusually large for this user? |
| Speed | Has this user made 10 transfers in the last hour? |
| Recipient | Is this someone the user has paid before? |
| Device | Is this a new device the user has never used? |
| Location | Is this transaction coming from a new country? |
| Time of day | Is this unusual for this user's typical behavior? |
| Crypto-specific | First crypto purchase? Unusually large crypto order? |

The model returns a score from 0 (definitely normal) to 1 (definitely fraud). Based on the score:
- Below 0.6: Approve automatically
- 0.6–0.8: Approve but flag for monitoring
- Above 0.8: Require additional verification (biometric or SMS code)
- Above 0.95: Decline

### Why we use a machine learning model instead of simple rules

A rule like "decline if amount > $5,000" would block legitimate large business payments. A rule like "decline if new device" would block every user who gets a new phone.

A machine learning model can consider all signals together. A $5,000 transaction from a business user who regularly sends large amounts from their known device: approve. A $50 transaction from a new device at 3am to an unknown recipient in a country the user has never visited: flag.

### The false positive problem

At eBay, we achieved 35% less fraud while blocking less than 1% of real customers. This balance matters: if the model is too aggressive, real customers get blocked and call support. If too lenient, fraud losses increase.

We review this balance weekly — if either number moves in the wrong direction, we investigate before it affects users significantly.

### What if fraud detection is down?

If the fraud service is unavailable or takes too long (more than 45ms), we **approve the transaction** and flag it for retroactive review. We never let fraud detection be the reason a legitimate payment fails.

---

## 13. Component: Compliance Service

### What it does
Enforces legal requirements around identity verification, sanctions screening, and transaction monitoring.

### Identity verification (KYC — Know Your Customer)

Before any user can send or receive money, they must complete identity verification. This is a legal requirement in the US — it prevents money laundering and financial crime.

**Process:**
1. User submits government ID (driver's license or passport)
2. User takes a live selfie
3. Socure (our identity provider) matches the selfie to the ID
4. Socure checks the person against government databases (fraud history, synthetic identity)
5. Result: Verified, Rejected, or Manual Review required

**For crypto:** Stricter rules apply. The IRS requires us to collect Social Security Numbers for crypto accounts (same as a brokerage account). This is built into the onboarding flow before any crypto trading is enabled.

### Sanctions screening (OFAC)

Before every transaction, we check both the sender and the recipient against the US Office of Foreign Assets Control (OFAC) list — a database of sanctioned individuals and organizations.

**This is non-negotiable.** Sending money to a sanctioned person is a federal crime.

- This check uses rule-based matching — not AI. It must be 100% deterministic and explainable.
- A match automatically blocks the transaction. No exceptions.
- The response time must be under 100ms so it doesn't slow down payments.

### Transaction monitoring (AML — Anti-Money Laundering)

Even after a transaction completes, we continuously analyze patterns for money laundering indicators:
- Small amounts sent to many different people (structuring)
- Large amounts moved quickly between accounts
- Unusual activity inconsistent with the account's history

This monitoring uses AI to detect patterns that rules alone would miss. Suspicious patterns create alerts in a queue that human compliance specialists review.

---

## 14. Component: Reconciliation Service

### What it does
Every night, compares our records against the records of every external payment partner to make sure every dollar is accounted for.

### Why this is necessary

We move money through external networks (Ethereum blockchain, custodian BitGo, etc.). Those networks have their own records. If our records say $50 was sent but the blockchain says $50.01 arrived, something needs investigation.

### The nightly process

```
11:00 PM every night:

1. Download settlement files from:
   - Ethereum blockchain (transaction history)
   - BitGo (crypto holdings and movements)
   - Any other payment partners

2. For each item in each file:
   Compare against our ledger_entries
   → MATCH: record as reconciled
   → IN OUR LEDGER ONLY: investigate (may be timing, may be an error)
   → IN PARTNER FILE ONLY: critical — we may have missed recording something

3. Auto-resolve tiny rounding differences (less than $0.01)

4. Flag all other gaps for human review, ranked by dollar amount

5. Verify the fundamental accounting rule:
   Total of all DEBIT entries = Total of all CREDIT entries
   If not equal: emergency alert immediately

6. Generate report by 6 AM for Finance and Compliance teams
```

**Lesson from X Money:** Automated reconciliation found two systematic errors in our ledger within the first month. These would have caused regulatory problems if discovered during a state examination. Running reconciliation nightly caught them in time.

---

## 15. Data Model — Every Table

### Core wallet tables

```sql
-- Users: linked to Intuit login system
CREATE TABLE users (
    user_id         UUID         PRIMARY KEY DEFAULT gen_random_uuid(),
    intuit_sso_id   VARCHAR(255) UNIQUE NOT NULL,  -- from Intuit's login system
    email           VARCHAR(255) UNIQUE NOT NULL,
    identity_status VARCHAR(50)  NOT NULL DEFAULT 'PENDING',
        -- PENDING | VERIFIED | REJECTED | EXPIRED
    account_type    VARCHAR(50)  NOT NULL DEFAULT 'CONSUMER',
        -- CONSUMER | SMALL_BUSINESS | ENTERPRISE
    daily_limit_usd NUMERIC(20,8) NOT NULL DEFAULT 10000.00,
    created_at      TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

-- Wallets: one per user in Phase 1
CREATE TABLE wallets (
    wallet_id   UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id     UUID        NOT NULL REFERENCES users(user_id),
    status      VARCHAR(50) NOT NULL DEFAULT 'ACTIVE',
        -- ACTIVE | SUSPENDED | CLOSED
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Balances: one row per wallet per currency
-- Separate table allows new currencies without changing wallet schema
CREATE TABLE wallet_balances (
    wallet_id           UUID         NOT NULL REFERENCES wallets(wallet_id),
    currency            VARCHAR(20)  NOT NULL,
        -- USDC | USDT | PYUSD | BTC | ETH
    available_balance   NUMERIC(30,8) NOT NULL DEFAULT 0,
        -- What the user can spend right now
    pending_balance     NUMERIC(30,8) NOT NULL DEFAULT 0,
        -- Money reserved for a transfer in progress
    version             BIGINT        NOT NULL DEFAULT 0,
        -- Optimistic lock: prevents two simultaneous updates from conflicting
    updated_at          TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    PRIMARY KEY (wallet_id, currency),
    CONSTRAINT balance_non_negative  CHECK (available_balance >= 0),
    CONSTRAINT pending_non_negative  CHECK (pending_balance   >= 0)
);

-- Transactions: the customer-facing payment record
CREATE TABLE transactions (
    txn_id              UUID         PRIMARY KEY DEFAULT gen_random_uuid(),
    from_wallet_id      UUID         REFERENCES wallets(wallet_id),
    to_wallet_id        UUID         REFERENCES wallets(wallet_id),
    external_address    VARCHAR(255),  -- for sends to external blockchain addresses
    type                VARCHAR(50)  NOT NULL,
        -- SEND | RECEIVE | QR_PAYMENT | CRYPTO_BUY | CRYPTO_SELL | DEPOSIT | WITHDRAWAL
    amount              NUMERIC(30,8) NOT NULL,
    currency            VARCHAR(20)  NOT NULL,
    network_fee         NUMERIC(30,8) NOT NULL DEFAULT 0,
    status              VARCHAR(50)  NOT NULL DEFAULT 'INITIATED',
        -- INITIATED | PENDING | COMPLETED | FAILED | REVERSED
    blockchain_tx_hash  VARCHAR(255),
    idempotency_key     VARCHAR(255) UNIQUE NOT NULL,  -- prevents duplicate processing
    created_at          TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    completed_at        TIMESTAMPTZ
);

-- Ledger entries: immutable accounting records
-- NEVER UPDATE. NEVER DELETE. Append-only forever.
CREATE TABLE ledger_entries (
    entry_id        UUID         PRIMARY KEY DEFAULT gen_random_uuid(),
    txn_id          UUID         NOT NULL REFERENCES transactions(txn_id),
    wallet_id       UUID         NOT NULL,
    direction       VARCHAR(10)  NOT NULL CHECK (direction IN ('DEBIT', 'CREDIT')),
    amount          NUMERIC(30,8) NOT NULL CHECK (amount > 0),
    currency        VARCHAR(20)  NOT NULL,
    balance_after   NUMERIC(30,8) NOT NULL,  -- snapshot: how much was in wallet after this entry
    recorded_at     TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

-- Outbox: ensures ledger writes and queue messages stay in sync
CREATE TABLE outbox_events (
    event_id        UUID         PRIMARY KEY DEFAULT gen_random_uuid(),
    event_type      VARCHAR(100) NOT NULL,
    reference_id    VARCHAR(255) NOT NULL,  -- txn_id or other primary reference
    payload         JSONB        NOT NULL,
    created_at      TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    sent_at         TIMESTAMPTZ  -- NULL means not yet sent to queue
);
-- Index to find unsent events quickly
CREATE INDEX idx_outbox_unsent ON outbox_events (created_at) WHERE sent_at IS NULL;
```

### Crypto investing tables

```sql
-- Crypto holdings: mirrors what BitGo holds on our behalf
CREATE TABLE crypto_holdings (
    holding_id      UUID         PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID         NOT NULL REFERENCES users(user_id),
    asset           VARCHAR(20)  NOT NULL,  -- BTC | ETH
    quantity        NUMERIC(30,8) NOT NULL DEFAULT 0,
    custodian_ref   VARCHAR(255),  -- BitGo's internal reference for reconciliation
    updated_at      TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    UNIQUE (user_id, asset)
);

-- Tax lots: every purchase creates one lot — enables cost basis calculation
CREATE TABLE tax_lots (
    lot_id              UUID         PRIMARY KEY DEFAULT gen_random_uuid(),
    holding_id          UUID         NOT NULL REFERENCES crypto_holdings(holding_id),
    purchase_date       DATE         NOT NULL,
    purchase_price_usd  NUMERIC(20,8) NOT NULL,  -- price per unit at time of purchase
    quantity_purchased  NUMERIC(30,8) NOT NULL,
    quantity_remaining  NUMERIC(30,8) NOT NULL,  -- decreases when sold
    cost_method         VARCHAR(30)  NOT NULL DEFAULT 'FIFO',
    fully_sold_at       TIMESTAMPTZ  -- NULL means some or all still held
);

-- Price records: stored after each aggregation cycle
CREATE TABLE asset_prices (
    asset           VARCHAR(20)   PRIMARY KEY,
    price_usd       NUMERIC(20,8) NOT NULL,
    recorded_at     TIMESTAMPTZ   NOT NULL,
    is_stale        BOOLEAN       NOT NULL DEFAULT false
);

-- Annual tax reports: pre-computed for 1099-DA generation
CREATE TABLE tax_reports (
    report_id           UUID         PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID         NOT NULL REFERENCES users(user_id),
    tax_year            SMALLINT     NOT NULL,
    status              VARCHAR(30)  NOT NULL DEFAULT 'PENDING',
    short_term_gain     NUMERIC(20,8),
    long_term_gain      NUMERIC(20,8),
    total_proceeds      NUMERIC(20,8),
    total_cost          NUMERIC(20,8),
    report_generated_at TIMESTAMPTZ,
    turbotax_synced_at  TIMESTAMPTZ,
    UNIQUE (user_id, tax_year)
);
```

### QR code table

```sql
-- QR codes: X9A standard, short-lived, digitally signed
CREATE TABLE qr_codes (
    qr_id               UUID         PRIMARY KEY DEFAULT gen_random_uuid(),
    merchant_wallet_id  UUID         NOT NULL REFERENCES wallets(wallet_id),
    amount              NUMERIC(30,8),  -- NULL = customer enters amount
    currency            VARCHAR(20)  NOT NULL DEFAULT 'USDC',
    one_time_code       UUID         NOT NULL UNIQUE,  -- prevents reuse
    digital_signature   TEXT         NOT NULL,
    expires_at          TIMESTAMPTZ  NOT NULL,  -- 30-second TTL
    used_at             TIMESTAMPTZ,
    created_at          TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);
```

---

## 16. How Everything Connects — End-to-End Flows

### Flow 1: Stablecoin transfer (Maria sends $500 to her supplier)

```
Step  Component              What happens                              Notes
────  ─────────────────────  ────────────────────────────────────────  ─────────────────────────
1     QuickBooks app         POST /wallets/transfer {amount:500,       Must include unique
                             currency:USDC, to:supplier-wallet}        request ID
2     Security Gateway       Verify Maria's login token, check         Reject if token
                             rate limit (max 100 req/min per user)     invalid or expired
3     Transaction Service    Check: has this request ID been seen      If duplicate: return
                             before? (Redis lookup, 24-hour memory)    original response
4     Fraud Detection        gRPC call → review transaction (45ms      If timeout: approve,
                             timeout)                                   don't block payment
5     Database               Lock both wallets in alphabetical order   Deadlock prevention
6     Database               Check Maria's available_balance >= 500    If not: return error
7     Database               BEGIN one combined operation:             All succeed or all fail:
                               Deduct 500 from Maria's balance         no partial states
                               Add 500 to supplier's balance
                               Write two ledger entries (DEBIT+CREDIT)
                               Write outbox message
                             COMMIT
8     Transaction Service    Return transaction ID to Maria's app      Blockchain is async —
                             with status: INITIATED                    Maria sees "Processing"
9     Outbox Poller          Every 100ms: reads unsent messages,       Safe because message
      (background)           sends to Kafka queue                      is already in database
10    Blockchain Gateway     Receives queue message, builds the        Signs in hardware
      (Kafka consumer)       Ethereum transfer instruction, signs it,  device — key never
                             submits to Ethereum                       in software
11    Confirmation Poller    Checks Ethereum every 3 seconds for       Timeout: 10 minutes
      (background)           confirmation
12    On confirmation        Updates transaction status to COMPLETED   Sends push notification
                             Publishes "transfer completed" event      to both Maria and supplier
13    QuickBooks sync        QuickBooks receives event, records        Maria sees it in her
                             the $500 payment automatically            books automatically
```

### Flow 2: Crypto buy (Priya buys $100 of Bitcoin)

```
Step  Component              What happens
────  ─────────────────────  ──────────────────────────────────────────────────────────────
1     Credit Karma app       Priya taps "Buy Bitcoin" → POST /crypto/orders
2     Price Feed Service     Check BTC price from cache (must be < 5 seconds old)
                             Show Priya: "$43,285/BTC → you receive 0.00231 BTC"
3     Priya confirms
4     Compliance Service     Identity verified? Daily crypto limit not exceeded? OFAC clear?
5     Fraud Detection        Review: first crypto purchase? Unusual amount? New device?
6     Transaction Service    Lock $100 in pending_balance (cannot be double-spent)
7     Outbox + Kafka         Write outbox message → send to queue
8     Custodian Service      Receives queue message → sends "buy $100 BTC" order to BitGo
9     Exchange execution     BitGo routes order to Coinbase Pro or Kraken → fills at market
                             Fill confirmed: 0.002312 BTC at $43,285.52, fee = $0.50
10    Tax Lot Engine         Creates new tax lot record: {date, price, quantity}
11    Transaction Service    Deduct $100.50 from USDC balance, add 0.002312 to BTC holding
12    Notification           "You bought 0.002312 BTC at $43,285.52" push notification
13    TurboTax sync          Trade data sent to TurboTax for future tax return pre-fill
```

### Flow 3: Nightly reconciliation

```
11:00 PM:
  Download records from: Ethereum blockchain, BitGo, any other partners
  For each item: match against our ledger_entries
    MATCH → reconciled
    OURS ONLY → investigate (timing issue or error)
    THEIRS ONLY → critical gap — escalate immediately
  Verify: total DEBITs = total CREDITs in our ledger
  Generate report → Finance and Compliance teams by 6 AM
```

---

## 17. Scale & Reliability

### How we handle 500 transactions per second

| Challenge | Approach |
|---|---|
| Services get overloaded | Kubernetes automatically adds more server instances when load increases, removes them when it decreases |
| Cache gets hot | Balance checks hit Redis (~2ms), not the database (~8ms). Database handles writes only. |
| Database gets overloaded | PgBouncer (connection pooler) sits in front of PostgreSQL, preventing too many simultaneous connections |
| One service is slow | Each service has an independent pool of resources — a slow crypto service cannot steal resources from the transaction service |
| TurboTax season (April 15) | We scale to 3× capacity before the peak, not after it starts |

### Preventing the common failure patterns

**Thundering herd:** When the cache expires, 1,000 simultaneous requests hit the database. Prevention: add random 0–2 second variation to all cache expiry times so they don't all expire at once.

**Retry storm:** When a service comes back from downtime, 10,000 backed-up retries hit it at once. Prevention: add random delays to retries (e.g., wait 1–4 seconds on first retry, 2–8 seconds on second, etc.)

**Cascade failure:** Service A is slow → Service B piles up waiting → Service B crashes → everything cascades. Prevention: Circuit breakers. If service A fails 50% of recent calls, Service B stops calling it for 30 seconds and returns a safe fallback response.

### The two-region setup

```
Normal operation (80% East, 20% West):
  us-east-1 ──── most traffic ────► users
  us-west-2 ──── some traffic ────► users

Database replication:
  us-east-1 primary ──sync──► us-west-2 replica
  (sync means: write only confirmed when BOTH regions have it — zero data loss)

Region failure:
  us-east-1 health check fails
  Route 53 automatically shifts 100% traffic to us-west-2 (< 60 seconds)
  us-west-2 scales from 20% to 100% capacity automatically
  Recovery time: under 5 minutes
```

---

## 18. Security Architecture

### Layers of protection

```
Layer 1 — Network
  All traffic between services uses mutual TLS (both sides verify each other)
  API Gateway is the only entry point from the internet
  All internal services are in private subnets — not reachable from internet

Layer 2 — Identity
  Every request carries a login token (JWT) — expires after 15 minutes
  For transactions over $100 or from a new device: require biometric or SMS confirmation
  Token revocation list in Redis — invalidated tokens rejected within seconds

Layer 3 — Keys (most critical for crypto)
  ALL cryptographic keys stored in AWS CloudHSM hardware devices
  Keys never exist in software memory — operations happen inside the hardware device
  Moving large crypto amounts requires 3 independent systems to agree (multi-signature)
  Key rotation: automated quarterly

Layer 4 — Data
  All personally identifiable information encrypted in the database
  All communication between services: TLS 1.3 (older versions rejected)
  Never log: private keys, social security numbers, full account numbers, passwords

Layer 5 — Compliance
  OFAC sanctions check: every transaction, every user, no exceptions
  Identity verification: no transactions until identity is verified
  Audit logs: every sensitive action recorded, retained for 7 years
```

### Threat responses

| Threat | How we handle it |
|---|---|
| Someone copies a QR code | 30-second expiry + one-time code makes the copy useless |
| Someone sends the same payment twice | Idempotency key returns original response, processes nothing new |
| Two transfers hit the same wallet at once | Database row locking ensures they process sequentially |
| Crypto private key theft attempt | Key never leaves hardware device — cannot be stolen via software |
| Fake price feed data | Three independent sources; if two disagree by >0.5%, halt trading |
| Blockchain front-running | Use private submission channels that don't expose transactions publicly |

---

## 19. Telemetry & Monitoring

### The four numbers every service tracks

Every service in the system tracks these four measurements, from day one — not after launch:

| Measurement | What it means | Alert threshold |
|---|---|---|
| **Latency** | How long each operation takes | If P99 exceeds 200ms for wallet ops, or 50ms for fraud review |
| **Traffic** | How many requests per second | Used to detect sudden drops (could mean upstream failure) |
| **Error rate** | Percentage of requests that fail | Alert if above 0.1% |
| **Saturation** | How full are our resources | Alert if database connections > 80%, or message queue > 1,000 backlogged |

### Business measurements we track

| Measurement | Alert | Why |
|---|---|---|
| Transaction success rate | Below 99.9%: immediate alert | This is what customers feel |
| Crypto price freshness | Stale > 5 seconds: pause crypto orders | Stale price = incorrect trades |
| AI fraud model accuracy | Weekly review; alert if changes > 5% | Catch drift before it affects users |
| Custodian response time | P99 > 5 seconds: investigate | Slow custodian = delayed crypto orders |
| Tax report completion | 100% must complete before January 31 | IRS deadline |
| Reconciliation gaps | Any material gap: immediate investigation | Financial integrity |

### Tracing through the system

Every request carries a unique trace ID from the moment it enters the Security Gateway. This ID is passed to every service the request touches. When something goes wrong, we can find every log entry and database operation for that exact transaction in seconds.

Every log entry includes: the trace ID, which service it came from, the wallet ID, the transaction amount and currency, and how long the operation took. We never log: passwords, private keys, or personal information.

---

## 20. Disaster Recovery

### Recovery targets

| Target | Goal | How we achieve it |
|---|---|---|
| Recovery Time (RTO) | Under 5 minutes | Automated failover, pre-scaled standby region |
| Data Loss (RPO) | Under 30 seconds | Synchronous database replication to second region |
| Mean Time to Repair | Under 15 minutes | Monthly practice drills, runbooks, on-call automation |

### Monthly failure drills (GameDay)

We do not assume recovery works. We test it:

1. **Kill the primary database** — verify automatic promotion of backup in under 30 seconds
2. **Introduce artificial slowness on the blockchain gateway** — verify circuit breaker opens and payments return "pending" (not "failed")
3. **Flood the fraud service** — verify transaction service falls back to approve (does not block all payments)
4. **Simulate the custodian (BitGo) being offline** — verify crypto orders queue safely and users see a clear message
5. **Kill 50% of queue messages** — verify the outbox pattern guarantees eventual delivery anyway

### When a region fails — 30-minute runbook

```
T+0:  Monitoring detects: health checks failing across us-east-1
T+2:  On-call engineer confirms: is this one availability zone or the whole region?
T+3:  Route 53 health check fails → DNS automatically routes 100% traffic to us-west-2
T+4:  us-west-2 automatically scales from 20% to 100% capacity (K8s HPA)
T+5:  Service restored. Verify: success rate > 99.9%, latency within targets
T+10: Audit in-flight transactions — verify no transactions were lost
T+30: Post-incident review started. Written summary within 48 hours.
```

### Crypto-specific recovery scenarios

| Scenario | Response |
|---|---|
| Gas fees spike 10× | Pause new stablecoin transfers, queue them, notify users with estimated wait time |
| Transaction stuck in blockchain for > 10 minutes | Cancel, reverse the balance, notify user with full explanation |
| BitGo (custodian) is unreachable | Pause new crypto orders. Existing holdings are safe — they are in our ledger. Show users "Market temporarily unavailable." |
| Price feed all three sources disagree | Halt new crypto orders. Display last known prices with "pricing currently updating" message. |

---

## 21. Engineering Practices

### API contract first

Before any code is written, the API specification is written, reviewed, and approved by all four product teams. If the API needs to change after shipping:
- Minor additions: fine, with no notice required
- Breaking changes: 90 days notice minimum, with a versioned migration path

### Testing pyramid

```
End-to-end tests (5%):
  Full happy path: user buys Bitcoin, price updates, TurboTax receives data
  Run before each production deployment

Integration tests (25%):
  Every error scenario: what if fraud service is down? what if blockchain times out?
  Every compensation path: what if a payment fails halfway through?

Unit tests (70%):
  Every business calculation: tax lot FIFO, fee calculation, balance update
  Code cannot ship without 80% line coverage
```

### Feature flags on everything

Every new capability launches behind a feature flag — an on/off switch controlled without a deployment:

- New stablecoin currency: flag off → test with 1% of users → expand → fully on
- New crypto asset: flag off → test → expand
- New AI model version: shadow mode (runs but doesn't decide) → canary (5% of users) → full rollout

If anything goes wrong: turn the flag off in 30 seconds. No rollback deployment required.

### Incident process

When something breaks:
1. Post in the incidents channel immediately — even before knowing what's wrong
2. Prioritize reducing impact over finding the cause (restart the pod first, investigate second)
3. Update the team every 5 minutes while the incident is active
4. Write a post-incident review within 48 hours: what happened, why, what we're changing
5. No blame — the goal is a better system, not a guilty person
6. Action items from the review go into the next sprint and are tracked to completion

---

## 22. Key Decisions & Trade-offs

| Decision | What we chose | What we didn't choose | Why | When to reconsider |
|---|---|---|---|---|
| **Stablecoin first** | USDC | USDT first | USDC has monthly published audits of its reserves; USDT has faced regulatory scrutiny | Add USDT in Phase 2 after USDC is stable |
| **Crypto custody** | Institutional custodian (BitGo) | Users hold their own keys | Regulatory compliance, user experience, insurance; self-custody requires 12–18 months of additional licensing | Offer advanced self-custody option in Phase 3 for power users |
| **Database** | PostgreSQL | CockroachDB, Google Spanner | PostgreSQL handles far more than our 500 TPS requirement; team knows it; 30 years of proven reliability | If we need > 20,000 TPS sustained, or need to write from 3+ geographic regions simultaneously |
| **Message coordination** | Kafka with choreography (services react to events) | Temporal workflow engine | Our payment flow has 6 steps — Temporal would add significant infrastructure complexity for marginal benefit | If we add human review steps (e.g., manual fraud approval that takes hours) or exceed 10 steps |
| **Price feeds** | Three independent sources, median | Single source | Single source failure = all crypto trading halts; three sources = one can fail without impact | If one source becomes the clear industry standard with strong SLA guarantees |
| **Tax lot method** | FIFO (First In, First Out) as default | HIFO, Specific Identification | FIFO is the IRS default and lowest compliance risk; simplest to explain to users | Allow user selection in account settings for tax optimization — Phase 2 |
| **Internal communication** | gRPC for service-to-service | REST everywhere | gRPC is faster (saves 5–10ms per call) and the strict schema prevents accidental contract changes between services | Never — keep gRPC internal, REST external |
| **Fraud model** | In-house machine learning model | Buy a vendor model | Intuit's 100M+ users provide training data advantages that no vendor model can match | Use vendor-labeled training data for new geographic markets where we have no historical data |
| **OFAC screening** | Rule-based matching | Machine learning | Federal law requires this to be deterministic and auditable; ML cannot be used here | Never — this is a regulatory requirement |

---

*This document covers the complete system design for the Intuit Stablecoin Wallet and Crypto Investing platform. Every component is described at the level of what it does, how it works, what decisions were made, and why.*

*Built on the same patterns as X Money — different asset class. Same architecture. I know what works and what I'd change.*
