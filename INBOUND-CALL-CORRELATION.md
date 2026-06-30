# Correlating Inbound Calls — LiveKit ↔ Vobiz

A step-by-step guide to match a **LiveKit** inbound call session with its **Vobiz**
call record (CDR), so you can join the two logs together.

---

## The key

An inbound call has two SIP legs: caller → Vobiz, and Vobiz → your LiveKit agent. Each
leg has its own SIP Call-ID, so they are stored in different CDR fields. The one that
equals what LiveKit sees is **`bridge_uuid`**:

| LiveKit attribute  | Vobiz CDR field   | What it is                                   |
| ------------------ | ----------------- | -------------------------------------------- |
| `sip.callIDFull`   | **`bridge_uuid`** | The leg between Vobiz and LiveKit — your key  |
| —                  | `sip_call_id`     | The **caller** leg — will **not** match       |
| —                  | `uuid`            | The CDR record's own id (shown as "Call ID")  |

> ⚠️ Match on **`bridge_uuid`**. Do **not** use `sip_call_id` (that is the inbound
> caller leg) or `sip.callID` (a LiveKit-internal value).

---

## Step 1 — Capture the Call-ID in your agent

Inbound callers are already in the room when your agent connects, so read the
participant attributes right after the session starts:

```python
await session.start(room=ctx.room, agent=YourAgent(), room_input_options=...)

for p in ctx.room.remote_participants.values():
    attrs = p.attributes or {}
    sip_call_id = attrs.get("sip.callIDFull")        # ← join key (== Vobiz bridge_uuid)
    caller      = attrs.get("sip.phoneNumber")       # caller number (From)
    did         = attrs.get("sip.trunkPhoneNumber")  # number they dialed (To)

    logger.info("inbound call: id=%s from=%s to=%s", sip_call_id, caller, did)
```

Store `sip_call_id` with your session so you can look it up later.

---

## Step 2 — Fetch the call record from Vobiz

Call the Vobiz CDR API for the relevant time window and DID:

```bash
curl -G "https://api.vobiz.ai/api/v1/Account/{auth_id}/cdr" \
  --data-urlencode "call_direction=inbound" \
  --data-urlencode "to_number=+91XXXXXXXXXX" \
  --data-urlencode "start_date=2026-06-30" \
  --data-urlencode "end_date=2026-06-30" \
  --data-urlencode "per_page=50" \
  -H "X-Auth-ID: {auth_id}" \
  -H "X-Auth-Token: {auth_token}"
```

Each record in the response `data[]` array includes `bridge_uuid` along with call
details (`caller_id_number`, `destination_number`, `duration`, `billsec`, `mos`,
`jitter`, `cost`, `hangup_cause`, `uuid`, and more).

---

## Step 3 — Match the two records

Find the CDR whose `bridge_uuid` equals the `sip.callIDFull` you captured:

```python
match = next(
    (r for r in cdr_response["data"] if r.get("bridge_uuid") == sip_call_id),
    None,
)
if match:
    vobiz_call_id = match["uuid"]          # the id shown in the Vobiz console
    # match["duration"], match["mos"], match["cost"], match["hangup_cause"], ...
    ...
```

That single record gives you everything you need to merge your LiveKit session with the
Vobiz call log.

---

## Fallback (coarse match)

If a record has no `bridge_uuid` (e.g. a call that was not bridged), approximate using
`caller_id_number` (From) + `destination_number` (To) + a `start_time` window. The
`bridge_uuid` match is exact and preferred.
