# Inbound Call Correlation — LiveKit ↔ Vobiz

How to tie a **LiveKit** call/session back to the matching **Vobiz** call record (trunk
webhook / CDR) for **inbound** calls — and why the trick you use for outbound doesn't
carry over unchanged.

---

## The problem

For **outbound** calls it's easy. You originate the leg, so you inject your own
correlation id as a custom SIP header and Vobiz echoes it back in the trunk webhook:

```python
# agent.py — outbound
await ctx.api.sip.create_sip_participant(
    api.CreateSIPParticipantRequest(
        ...,
        headers=sip_headers or {},                  # e.g. {"X-My-Call-Id": "abc123"}
        include_headers=sip_protocol.SIP_X_HEADERS, # echo X-* headers back
    )
)
```

→ Vobiz trunk webhook (`CallInitiated` / `Hangup`) carries that header value, so the
LiveKit room and the Vobiz CDR share a key.

For **inbound**, the originator is the **caller / carrier → Vobiz**, not you. You never
touch the INVITE, so there's no place to inject a custom header from the agent side.
The correlation handle has to come from **whatever Vobiz puts on the INVITE it sends to
LiveKit**.

---

## Inbound call flow

```
PSTN caller ──▶ Vobiz (Kamailio) ──INVITE──▶ LiveKit inbound trunk
                                                   │
                                          dispatch rule (SDR_…)
                                                   │
                                                   ▼
                                          Room + SIP participant
                                                   │
                                                   ▼
                                              Agent (this repo)
```

The agent job runs with **empty metadata** on inbound (`ctx.job.metadata == ''`) —
the metadata channel is outbound-only. Everything you can correlate on lives in the
**SIP participant attributes**.

---

## What Vobiz sends

On an inbound call, LiveKit surfaces the SIP participant with attributes like:

```
PARTICIPANT JOINED: identity=sip_+91XXXXXXXXXX  name="Phone +91XXXXXXXXXX"  kind=SIP
  sip.callID            = SCL_xxxxxxxxxxxx                       # LiveKit-internal id — NOT useful for Vobiz
  sip.callIDFull        = xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx   # real SIP Call-ID on the wire  ⭐
  sip.callStatus        = ringing
  sip.trunkID           = ST_xxxxxxxxxxxx                        # LiveKit inbound trunk
  sip.ruleID            = SDR_xxxxxxxxxxxx                       # LiveKit dispatch rule
  sip.phoneNumber       = +91XXXXXXXXXX                         # From (caller)
  sip.trunkPhoneNumber  = +9180XXXXXXXX                         # To (DID dialed)
  (no sip.h.* headers — Vobiz forwarded no custom headers)
```

**Key takeaways**

1. **Inbound dispatch already works** — rule `SDR_xxxxxxxxxxxx` routes the DID to this
   agent. No dispatch changes needed.
2. **Vobiz forwards no custom header by default** (`no sip.h.*`). So there is no
   ready-made business id to read; the outbound echo trick has no automatic inbound twin.
3. **`sip.callIDFull` is the natural join key** — the actual SIP `Call-ID` from the
   INVITE. (`sip.callID`, the `SCL_…` value, is LiveKit's own handle and means nothing
   to Vobiz — do not use it for correlation.)

> ⚠️ Attribute-name gotcha: correlate on **`sip.callIDFull`**, not `sip.callID`.

---

## Reading the attributes in the agent

Inbound SIP participants are present in the room *before* the agent connects, so
`participant_connected` does **not** fire for them. Enumerate after `session.start()`
(or use `ctx.wait_for_participant()`):

```python
await session.start(room=ctx.room, agent=OutboundAssistant(), room_input_options=...)

# Inbound caller is already in the room at connect time:
for p in ctx.room.remote_participants.values():
    attrs = p.attributes or {}
    sip_call_id = attrs.get("sip.callIDFull")    # ↔ Vobiz SIPCallID
    caller      = attrs.get("sip.phoneNumber")   # From
    did         = attrs.get("sip.trunkPhoneNumber")  # To
    header_uuid = attrs.get("sip.h.x-vobiz-call-uuid")  # only if Vobiz injects it (see Option B)
```

---

## Two correlation strategies

### Option A — Passive: `sip.callIDFull` ↔ Vobiz `SIPCallID`  (zero config)

Match the LiveKit `sip.callIDFull` against the `SIPCallID` field in the Vobiz inbound
webhook / call log.

- ✅ No Kamailio or trunk changes.
- ⚠️ **Depends on Vobiz preserving the Call-ID** on the leg toward LiveKit. Vobiz runs
  Kamailio (a B2BUA); if it re-generates the `Call-ID` for the LiveKit leg, the
  webhook's `SIPCallID` won't equal LiveKit's `sip.callIDFull`.

**Verify once:** open the Vobiz call log for the inbound call (To `+9180XXXXXXXX`) and
check whether its `SIPCallID` contains `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`.
- Matches → use Option A, done.
- Differs → use Option B.

### Option B — Active: inject a header in Kamailio  (deterministic)

Have Vobiz/Kamailio stamp a custom header carrying its own `CallUUID` onto the INVITE it
sends to LiveKit on the inbound origination route:

```
X-Vobiz-Call-UUID: <CallUUID>
```

Then map it on the **LiveKit inbound trunk** (`SIPInboundTrunkInfo`) so it lands as a
participant attribute:

```python
api.SIPInboundTrunkInfo(
    ...,
    include_headers=api.SIPHeaderOptions.SIP_X_HEADERS,        # map all X-* headers, OR:
    headers_to_attributes={"X-Vobiz-Call-UUID": "vobiz.callUUID"},
)
```

Read it: `participant.attributes["sip.h.x-vobiz-call-uuid"]` (with `include_headers`)
or `participant.attributes["vobiz.callUUID"]` (with `headers_to_attributes`).

- ✅ Deterministic; Vobiz's own `CallUUID` is in-session — the true inbound equivalent
  of the outbound echo.
- ⚠️ Requires a Kamailio header rule on the inbound route + a one-time inbound-trunk
  update. `include_headers` on the **outbound** `CreateSIPParticipantRequest` (as in
  `agent.py`) does **not** affect inbound — it must be set on the inbound trunk.

---

## Recommendation

1. Check the Call-ID against the Vobiz log (one lookup). If it matches → **Option A**,
   ship it, no infra change.
2. If it doesn't match, or you want a stable business id independent of SIP retries/
   re-INVITEs → **Option B**.

Either way, `sip.phoneNumber` (From) + `sip.trunkPhoneNumber` (To) + a timestamp window
is a decent fallback join when you only need coarse matching.

---

## Reference — fields

| LiveKit attribute       | Meaning                          | Vobiz webhook field        |
| ----------------------- | -------------------------------- | -------------------------- |
| `sip.callIDFull`        | SIP `Call-ID` on the INVITE      | `SIPCallID` (if preserved) |
| `sip.callID`            | LiveKit-internal id (`SCL_…`)    | — (do not use)             |
| `sip.phoneNumber`       | Caller number                    | `From`                     |
| `sip.trunkPhoneNumber`  | DID dialed                       | `To`                       |
| `sip.trunkID`           | LiveKit inbound trunk id         | —                          |
| `sip.ruleID`            | LiveKit dispatch rule id         | —                          |
| `sip.h.<header>`        | Forwarded SIP header (Option B)  | header value you set       |
