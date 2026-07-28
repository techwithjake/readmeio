---
title: Case study
deprecated: false
hidden: false
metadata:
  robots: index
---
# Tracking Down Intermittent Connectivity Drops Across a 180-Device Fleet

## The Problem

A subset of our field-deployed Raspberry Pi units were dropping off Tailscale intermittently — no consistent pattern, no obvious correlation to location, hardware batch, or time of day. Customers noticed before we did, which is the outcome you always want to avoid.

## Investigation

Pulling raw connectivity logs per-device wasn't going to scale past a handful of units, so I pulled logs into a structured table and ran SQL correlation across timestamps, device firmware versions, and network handoff events:

\`\`\`sql<br />SELECT device_id, firmware_version, COUNT(\*) AS drop_events<br />FROM connectivity_log<br />WHERE event_type = 'tailscale_disconnect'<br />AND ts > NOW() - INTERVAL '7 days'<br />GROUP BY device_id, firmware_version<br />ORDER BY drop_events DESC;<br />\`\`\`

<Callout icon="📘" theme="info">
  ### **Note:** The pattern that mattered wasn't the raw drop count — it was that drops clustered on one firmware version, not one location. That reframed the whole investigation.
</Callout>

## Root Cause

Isolating by firmware version pointed to a specific update that changed how the device handled DNS re-resolution after a network handoff — cellular to Starlink, for instance. Devices on that version were failing to re-resolve and silently sitting disconnected instead of retrying.

## Resolution

Rolled back the affected devices to the prior firmware via our in-house update mechanism, pushed a fix for the re-resolution bug, and added the drop-event query above as a standing check in our monitoring so this pattern gets caught in hours next time, not days.

## Takeaway

The fix was small. Finding it meant not trusting "random" as an explanation and going looking for the variable that actually correlated.
