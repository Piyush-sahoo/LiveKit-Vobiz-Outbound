# Inbound Call Correlation — LiveKit ↔ Vobiz

Tie a **LiveKit** inbound session back to the matching **Vobiz** call record (CDR).

---

## What's happening

```
PSTN caller ──▶ Vobiz (Kamailio) ──INVITE──▶ LiveKit inbound trunk
                                                   │  dispatch rule
                                                   ▼
                                          Room + SIP participant ──▶ Agent
```

- On **outbound** you originate the leg, so you inject a custom SIP header and Vobiz
  echoes it — easy correlation.
- On **inbound** you don't touch the INVITE. The agent job runs with **empty metadata**
  (`ctx.job.metadata == ''`). Everything you can correlate on lives in the **SIP
  participant attributes** LiveKit derives from the INVITE Vobiz sends.

---

## What Vobiz sends

On an inbound call, LiveKit surfaces the SIP participant with attributes like:

```
PARTICIPANT JOINED: identity=sip_+91XXXXXXXXXX  name="Phone +91XXXXXXXXXX"  kind=SIP
  sip.callID            = SCL_xxxxxxxxxxxx                       # LiveKit-internal id — do NOT use
  sip.callIDFull        = xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx   # real SIP Call-ID on the wire  ⭐
  sip.callStatus        = ringing
  sip.trunkID           = ST_xxxxxxxxxxxx                        # LiveKit inbound trunk
  sip.ruleID            = SDR_xxxxxxxxxxxx                       # LiveKit dispatch rule
  sip.phoneNumber       = +91XXXXXXXXXX                         # From (caller)
  sip.trunkPhoneNumber  = +9180XXXXXXXX                         # To (DID dialed)
  (no sip.h.* headers — Vobiz forwards no custom headers by default)
```

`sip.callIDFull` is the SIP `Call-ID` from the INVITE — **this is the join key.**
(`sip.callID`, the `SCL_…` value, is LiveKit-internal and means nothing to Vobiz.)

---

## How to capture

Inbound SIP participants are in the room *before* the agent connects, so
`participant_connected` does **not** fire for them. Read them after `session.start()`:

```python
await session.start(room=ctx.room, agent=OutboundAssistant(), room_input_options=...)

for p in ctx.room.remote_participants.values():
    attrs = p.attributes or {}
    sip_call_id = attrs.get("sip.callIDFull")        # ↔ Vobiz CDR sip_call_id
    caller      = attrs.get("sip.phoneNumber")       # From
    did         = attrs.get("sip.trunkPhoneNumber")  # To
```

---

## How to relate (Vobiz CDR API)

The Vobiz CDR record exposes **`sip_call_id`** — match it against LiveKit's
`sip.callIDFull`.

> ✅ **Confirmed:** Vobiz preserves the SIP `Call-ID` end-to-end, so
> `sip_call_id` **equals** `sip.callIDFull`. The direct match works — no
> Kamailio or trunk changes needed.

List CDRs:

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

In the response `data[]`, find the record where:

```
record["sip_call_id"] == livekit_attrs["sip.callIDFull"]
```

That record also gives you `bridge_uuid` (Vobiz call-session UUID), `caller_id_number`,
`destination_number`, `duration`, `billsec`, `mos`, `jitter`, `cost`, `hangup_cause`,
etc. — everything you need to merge the two call logs.

**Fallback join** when you only need coarse matching: `caller_id_number` +
`destination_number` + a `start_time` window.

---

## Optional: a custom business id

The `sip_call_id` ↔ `sip.callIDFull` match is confirmed working, so this is **not
required**. Use it only if you want a stable id you control (e.g. your own
campaign/lead id) carried in-session, independent of the SIP Call-ID:

1. Have Kamailio stamp a header on the INVITE to LiveKit, e.g. `X-Vobiz-Call-UUID: <CallUUID>`.
2. Map it on the **LiveKit inbound trunk** so it lands as an attribute:

   ```python
   api.SIPInboundTrunkInfo(
       ...,
       include_headers=api.SIPHeaderOptions.SIP_X_HEADERS,        # all X-* → sip.h.*, OR:
       headers_to_attributes={"X-Vobiz-Call-UUID": "vobiz.callUUID"},
   )
   ```

3. Read `attrs["sip.h.x-vobiz-call-uuid"]` (or `attrs["vobiz.callUUID"]`) in the agent.

> `include_headers` on the **outbound** `CreateSIPParticipantRequest` (in `agent.py`)
> does not affect inbound — it must be set on the **inbound trunk**.
