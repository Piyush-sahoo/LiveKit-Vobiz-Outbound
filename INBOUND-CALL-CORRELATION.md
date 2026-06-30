# Correlating Inbound Calls — LiveKit ↔ Vobiz

A step-by-step guide to match a **LiveKit** inbound call session with its **Vobiz**
call record (CDR), so you can join the two logs together.

---

## Overview

On an inbound call, Vobiz routes the caller to your LiveKit agent. The agent receives
the call as a SIP participant, and LiveKit exposes details about that call as
**participant attributes**.

The key field is the **SIP Call-ID**, which both sides share:

| LiveKit                 | Vobiz CDR      |
| ----------------------- | -------------- |
| `sip.callIDFull`        | `sip_call_id`  |

Vobiz preserves the SIP Call-ID end-to-end, so these two values are **equal** — that's
your join key. No special configuration is required.

---

## Step 1 — Capture the Call-ID in your agent

Inbound callers are already in the room when your agent connects, so read the
participant attributes right after the session starts:

```python
await session.start(room=ctx.room, agent=YourAgent(), room_input_options=...)

for p in ctx.room.remote_participants.values():
    attrs = p.attributes or {}
    sip_call_id = attrs.get("sip.callIDFull")        # ← join key (the SIP Call-ID)
    caller      = attrs.get("sip.phoneNumber")       # caller number (From)
    did         = attrs.get("sip.trunkPhoneNumber")  # number they dialed (To)

    logger.info("inbound call: id=%s from=%s to=%s", sip_call_id, caller, did)
```

> Use `sip.callIDFull` (the real SIP Call-ID). Do **not** use `sip.callID` — that is a
> LiveKit-internal value and will not match Vobiz.

Store `sip_call_id` with your session so you can look it up later.

---

## Step 2 — Fetch the call record from Vobiz

Call the Vobiz CDR API for the relevant time window and DID:

```bash
curl -G "https://api.vobiz.ai/api/v1/Account/{auth_id}/cdr" \
  --data-urlencode "call_direction=inbound" \
  --data-urlencode "to_number=+9180XXXXXXXX" \
  --data-urlencode "start_date=2026-06-30" \
  --data-urlencode "end_date=2026-06-30" \
  --data-urlencode "per_page=50" \
  -H "X-Auth-ID: {auth_id}" \
  -H "X-Auth-Token: {auth_token}"
```

Each record in the response `data[]` array includes `sip_call_id` along with call
details (`caller_id_number`, `destination_number`, `duration`, `billsec`, `mos`,
`jitter`, `cost`, `hangup_cause`, and more).

---

## Step 3 — Match the two records

Find the CDR whose `sip_call_id` equals the `sip.callIDFull` you captured:

```python
match = next(
    (r for r in cdr_response["data"] if r["sip_call_id"] == sip_call_id),
    None,
)
if match:
    # match["duration"], match["mos"], match["cost"], match["bridge_uuid"], ...
    ...
```

That single record gives you everything you need to merge your LiveKit session with the
Vobiz call log.

---

## Fallback (coarse match)

If you don't have the Call-ID for a given record, you can still approximate a match using
`caller_id_number` (From) + `destination_number` (To) + a `start_time` window. The
Call-ID match is exact and preferred.
