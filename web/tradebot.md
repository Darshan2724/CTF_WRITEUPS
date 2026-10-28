# KPMG CTF 2026: TradeBot — API Race Condition Writeup

## Challenge Information

| Field | Details |
| :--- | :--- |
| **Challenge Name** | TradeBot |
| **Category** | Web / Financial API / Concurrency Race Condition (TOCTOU) |
| **Points** | 20 |
| **Protocol** | JSON-RPC over HTTP |

---

## 1. Challenge Summary

TradeBot is an automated stock trading assistant backend allowing users to view quotes, browse public instruments, monitor portfolios, and place buy/sell orders. The challenge description indicates that a special confidential financial instrument exists outside the public order book. This instrument is accessible only to the internal settlement engine, which decides whether it can settle an order up front before routing, but never re-evaluates capacity or state during the settlement process.

---

## 2. API Analysis & Architectural Flaw

1. **Protocol Discovery:** Probing the health-check and root endpoints revealed the backend uses a structured JSON-RPC tool-calling protocol requiring an initial session handshake.
2. **Operations Enumeration:** Available calls include: `get_quote`, `get_stock_info`, `list_instruments`, `get_portfolio`, `risk_report`, and `place_order`.
3. **The `risk_report` Operation:** Calling `risk_report` exposed internal operational metrics:
   - Concurrency capacity cap per account.
   - Current active orders in-flight.
   - Latency metrics: routing latency and filling latency (multi-second processing window).
   - Integrity violation counter (initialized to 0).

### The Vulnerability: Non-Atomic TOCTOU
The engine checks account settlement capacity *once*, up front, and then spends several seconds executing the settlement and fill operations without maintaining an exclusive lock or atomic reservation.

---

## 3. Exploitation via Race Condition Burst

Because settlement takes several seconds and the admission gate checks concurrency once at arrival:
1. By launching a burst of concurrent order requests comfortably exceeding the account's concurrency cap within milliseconds of each other, multiple orders slip past the admission check simultaneously.
2. The simultaneous orders overlap inside what the settlement engine assumes is an exclusive execution window.
3. Upon detecting "settlement oversubscription," the engine's internal integrity handler attempts to reconcile the reserve state.
4. Instead of failing closed, the reconciliation logic dumps the internal reserve vault block—including the confidential sealed record containing the objective value—into the JSON-RPC response.

### Execution Script (Bash/Curl)

```bash
#!/bin/bash
SESSION="<ESTABLISHED_SESSION_ID>"
URL="https://target/api/rpc"

# Fire concurrent order placement burst
for i in {1..30}; do
  curl -s -X POST "$URL" \
    -H "X-Session-ID: $SESSION" \
    -H "Content-Type: application/json" \
    -d '{"jsonrpc":"2.0","method":"place_order","params":{"symbol":"ACME","qty":10},"id":'$i'}' &
done
wait
```

### Result
Two out of the thirty concurrent responses contained the engine's reconciliation payload, exposing the sealed reserve record and successfully solving the challenge.

---

## 4. Remediation & Best Practices
- **Atomic Concurrency Reservations:** Use atomic compare-and-swap (CAS) or database-level row locks across the entire transaction lifecycle rather than verifying capacity up front and acting later.
- **Fail-Closed Error Handling:** Handlers that catch invariant violations must log and abort rather than serializing internal state into user-facing responses.
- **Request Queuing & Rate Limiting:** Enforce per-session rate limits to mitigate concurrent request bursts at the API gateway layer.
