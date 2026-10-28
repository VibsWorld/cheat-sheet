## Disable Connected User Experiences and Telemetry
Settings App (Indirect Control)
While you can’t directly stop the service from Settings, you can reduce or disable telemetry:

Open Settings → Privacy & Security → Diagnostics & feedback.

Set Diagnostic data to Basic (or the lowest available).

Turn off Tailored experiences and Feedback frequency.
This doesn’t fully stop the service, but it minimizes its activity.

Group Policy Editor (for Pro/Enterprise editions)
Run gpedit.msc.

Navigate to:
Computer Configuration → Administrative Templates → Windows Components → Data Collection and Preview Builds.

Double‑click Allow Telemetry → set it to Disabled.

## Disable WIndows Delivery Optimization 

