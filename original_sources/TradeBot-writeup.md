# TradeBot — Writeup

> PRIVATE — the organizers' rules forbid publishing writeups, walkthroughs or solutions during or after the CTF (GitHub, Medium, blogs, etc.). Do not publish or share.
> This version intentionally omits the final answer. It never states the value, its format, or a placeholder for it. Say “the objective value” / “the final answer”.

| | |
|---|---|
| Event | KPMG CTF 2026 Campus Edition |
| Category | API |
| Points | 20 |
| Tools | curl, manual JSON-RPC over HTTP |

## Challenge summary
An “automated stock trading assistant” backend. It lets you look up stock prices, browse the market, check your portfolio, and place buy/sell orders. The description says there is a special instrument not sold on the public order book, readable only by the internal settlement engine — and hints that the engine decides whether it can settle an order before it finishes routing that order, and never re-checks that decision afterward.

## 1. First look
The web root returned nothing useful. A standard health-check path revealed the service was actually exposed as a tool-calling protocol endpoint rather than a plain REST API — i.e. a backend meant to be driven by structured “tool calls” (JSON-RPC over HTTP with a session handshake), not a page-based site.

## 2. Analysis and approach
1. Established the session handshake required by the protocol (an initial handshake call returns a session identifier that every following call must include as a header).
2. Enumerated the available operations. Among them: get a quote, get stock info, list the public instruments, view portfolio, view a “risk report,” look up an order by id, place an order, and a basic health check. The place-order operation’s own description stated that admission to settle is governed by a risk engine that caps how many orders for an account may be settling concurrently, and that orders arriving while at capacity are rejected.
3. Called the risk-report operation: it returned the concurrency cap, the number currently in-flight, and two separate latency figures (one for “routing,” one for “filling”) — plus a counter for integrity violations, sitting at zero. This matched the hint precisely: an admission decision made once, up front, trusted for the whole multi-second operation that follows, with no re-check at the end.
4. Tried guessing a name for the special instrument directly through the lookup/info/order operations with made-up symbols — consistently and cleanly rejected as unknown. Dead end: this wasn’t a naming puzzle.
5. Reasoned that if admission is “check capacity, then run a multi-second settlement without re-checking,” sending more concurrent order requests than the stated capacity should let several of them pass the check at nearly the same instant and overlap inside what the engine assumes is an exclusive settlement window — i.e. force the race the hint was describing.

## 3. Solution
Placed one ordinary order first to see the normal response shape and timing (confirmed the multi-second routing+fill delay matched the risk-report figures).

Then fired a burst of concurrent order-placement calls — using a background-job loop in a shell script so every request hit the server at essentially the same moment — using a count comfortably above the stated concurrency cap, all for an ordinary, publicly-tradable symbol, all over the same session.

Most responses came back as ordinary successful fills. A small number of them (two, out of thirty, whose orders had settled within milliseconds of each other) came back with extra fields that no other response contained: a marker from the engine itself reporting its own integrity had been compromised by “settlement oversubscription,” followed by a reconciliation block for an internal reserve vault — which, as part of “reconciling,” included the sealed record it was supposed to be protecting. Both occurrences of that record agreed with each other exactly, confirming it was a real, reproducible result rather than noise.

Scripts and full raw responses are kept locally under the challenge folder (not reproduced here).

## 4. Root cause
A textbook time-of-check / time-of-use (TOCTOU) flaw: the capacity check and the reservation of a settlement “slot” are not atomic with the settlement itself. The engine decides once, up front, whether it has room, then spends a multi-second window actually doing the work without holding that decision as an exclusive lock. Send enough concurrent requests and more than one settlement ends up inside what was meant to be a single-occupant window. The code path that detects this invariant violation, rather than failing safely, “reconciles” by emitting the protected internal record directly in its response — turning an availability bug (overbooking) into a confidentiality bug (secret disclosure).

## 5. Result
The objective was reached and verified end to end: the technique above reliably reproduced the same disclosed internal record across multiple independent concurrent bursts, confirming it was the intended solution path rather than a fluke.

## 6. Remediation
- Make the admission check and the settlement itself atomic — hold the capacity reservation (a lock/semaphore, or an atomic compare-and-reserve) for the full duration of the operation, not just at the moment of admission.
- Guard the sensitive internal resource itself with mutual exclusion (a proper lock, or a database-level row lock) so it can never be “read” by more than one in-flight operation at a time, and fail closed — reject/abort — rather than fail open when that invariant is violated.
- Never let a generic error-handling or reconciliation code path, reachable from normal user input, serialize sensitive internal state into its response. An integrity-violation handler should log and alert, not leak.
- Rate-limit or queue concurrent requests per client as defense in depth, so a logic bug in admission control isn’t trivially race-able from a single session.

## 7. Lessons
- Take hints written like a vulnerability description completely literally — they often name both the bug class (here, a non-atomic check-then-act / TOCTOU) and the shape of the exploit (race more concurrent requests than a stated limit).
- A basic health/status endpoint is worth checking early on an unfamiliar API-style target; it can reveal the real protocol in play when the obvious root path gives nothing.
- A self-reported “risk report” or capacity/metrics endpoint, if the target exposes one, is a gift — it can hand you the exact numbers (limit, timing) needed to size a race-condition attack without any guessing.
- Don’t waste time brute-forcing a secret resource’s name when the prompt explicitly says the only legitimate path to it is through a specific internal mechanism — that’s a strong signal to attack the mechanism’s logic, not the resource’s naming.
