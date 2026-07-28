---
title: Case study
deprecated: false
hidden: false
metadata:
  robots: index
---
# Tracking Down a Silent Service Failure Across a Raspberry Pi Fleet

## The Problem

A background sync service on a subset of our field-deployed Raspberry Pi units started crash-looping — restarting, failing within a couple seconds, restarting again, over and over. Systemd's restart counter climbed steadily on affected devices while the rest of the fleet ran fine. There was no obvious pattern by location, hardware batch, or deployment date. It looked random.

## Investigation

With 180+ devices in the field, pulling logs one SSH session at a time wasn't going to scale. We already had a fleet audit script for this. It reads device inventory, connects to each unit over a secure mesh network, and pulls back systemd service status. The first move was widening its use. We had it also grab the failing service's recent journal output, across every device reporting the bad status.

That surfaced the actual failure. Here's the relevant slice of the journal log from one affected device:

```text
s3_sync_daemon.sh: line 436: _site: unbound variable
[error] ERR at s3_sync_daemon.sh:721: cmd='_ext=$(_infer_app_config_ext audio)' exit=1
[info] Daemon exiting
```

<Callout icon="📘" theme="info">
  ### Note

  The pattern that mattered wasn't which devices were failing. It was that every failing device hit the exact same line in the exact same script. That reframed the investigation. It went from "what's different about these devices" to "what's different about this one code path."
</Callout>

## Root Cause

The script ran with `set -u`, which makes bash treat any reference to an unset variable as a fatal error. A conditional block that read a per-device config file was the only place that assigned two variables, `_site` and `_facility`. Here's that block:

```bash
if [[ -f "$_dev_cfg" ]]; then
    _site=$(grep -m1 '^SITE_NAME=' "$_dev_cfg" | cut -d= -f2-)
    _facility=$(grep -m1 '^FACILITY_NAME=' "$_dev_cfg" | cut -d= -f2-)
fi
```

On devices where that config file happened to be missing, the script never assigned `_site` or `_facility` at all. The very next validation check referenced `_site` to decide whether to log a warning. Under `set -u`, referencing an unset variable there killed the daemon immediately. It died before it could even log the warning it was trying to check for. That made the failure self-hiding: the exact code path meant to handle "config file missing gracefully" crashed the service instead.

It wasn't fleet-wide because it wasn't about the devices — it was about which devices were missing that one local config file, for unrelated provisioning reasons.

<Callout icon="🚧" theme="warn">
  ### Warning

  This failure mode hid itself: the code path meant to log a warning about a missing config file crashed the service instead. Any `set -u`/`set -e` script risks this same trap if it references a variable that only a conditional file check assigns.
</Callout>

## Resolution

The fix was small. Initialize both variables to empty strings before the conditional, so `set -u` has nothing to complain about even when the config file is absent. Add `|| true` to the `grep` calls too, so a non-match doesn't separately trip `set -e`. Here's the patched block:

```bash
local _site="" _facility=""
if [[ -f "$_dev_cfg" ]]; then
    _site=$(grep -m1 '^SITE_NAME='    "$_dev_cfg" | cut -d= -f2- || true)
    _facility=$(grep -m1 '^FACILITY_NAME=' "$_dev_cfg" | cut -d= -f2- || true)
fi
```

We rolled the patched script out fleet-wide using our existing deployment tooling. It backs up the current script on each device before replacing it, then restarts the service. It waits, then captures `systemctl status` per device into a structured result set. That way, a failed rollout on any single unit is immediately visible rather than silent.

The audit script's status check now runs as a standing monitor, so a config-file gap on a newly provisioned device gets caught by the same signal within hours instead of surfacing as a customer-reported outage days later.

<Callout icon="👍" theme="okay">
  ### Tip

  When rolling out a fix like this across a large fleet, back up the existing script on each device before overwriting it. Capture `systemctl status` per device into a structured result set immediately after restart. That turns "did the rollout work?" into a single glance, instead of a per-device guessing game.
</Callout>

## Takeaway

A failure that looks randomly distributed across a fleet is often a single deterministic bug. Only some devices happen to trigger it, because of one specific state on those devices. It's not about anything special about the devices themselves. The fix wasn't in "fixing the random ones." It was in finding the one line every failing device had in common.
