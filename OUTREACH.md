# Outreach / influence plan

Canonical repo: https://github.com/glasses666/macos27-external-touch
Canonical discussion: see the Discussions tab.

## Goals

1. Get the finding in front of Apple, DriverKit/HID developers, and touch-driver vendors.
2. Find anyone who already has a third-party HID service with `displayUUID` on macOS 27.
3. Turn that into either a public API/entitlement path or a vendor/DriverKit collaboration.
4. Keep the OSS tablet-as-second-screen ecosystem (AnyDisplay/OpenDisplay/SideScreen/MacTouch/HIDTouch) aware of the native path, so they can add it when it becomes possible.

## Posting order

### Phase 1 — canonical post (today)
- Create this repo + README + EVIDENCE.
- Enable Discussions and post the first thread:
  - title: "macOS 27 external touchscreens: native support exists behind a private feature flag; displayUUID is the wall"
  - body: TL;DR + link to README/EVIDENCE + specific asks.

### Phase 2 — targeted technical issues (today/tomorrow, max 3)
Post only where the maintainer has shown interest in native touch / HID:

1. **AnyDisplay** (`efekurucay/anydisplay`)
   - angle: "Your app already does touch/pen via CGEvent; macOS 27 has a native path behind displayUUID. Here is the evidence. Have you considered/seen a DriverKit route? Would you accept a native-touch design discussion?"
   - link canonical repo + discussion.

2. **Topti** (`Yukaii/topti`)
   - angle: "You requested `com.apple.developer.hid.virtual.device` for a virtual stylus. Data point: a generic USB HID digitizer is recognized by macOS 27 but rejected at WindowServer unless the HID service provides displayUUID. If Apple responds, ask whether the entitlement covers display association too."
   - link canonical.

3. **HIDTouch** (`koshi545/HIDTouch`) or **MacTouch** (`kashmeister/mactouch`)
   - angle: "Your README documents the virtual-digitizer entitlement block. We found the external-HID path and the displayUUID gate; here is the data. Have you seen a signed HID service provide it?"
   - pick one; do not duplicate spam.

### Phase 3 — broader reach (after initial reactions)
- Apple Developer Forums thread (HIDDriverKit / macOS) pointing to the repo; ask for entitlement/API clarification.
- Reddit: r/macapps, r/MacOS, r/AppleSilicon (one post, not three).
- Hacker News "Show HN"/technical submission if the write-up is solid.
- X/Twitter thread with screenshots/logs (tags: macOS 27, HIDDriverKit, touchscreen).
- Chinese communities: V2EX, 酷安, Bilibili (user account), 小红书 — adapted Chinese summary.
- Vendor outreach: Touch-Base (UPDD), Alogic support — ask whether macOS 27 native external touch is supported and how they associate displays.

## Message discipline

- Lead with the evidence and the ask, not with "Apple broke it".
- No exploit/debugger instructions; no "disable SIP" advice.
- One issue per repo; reference the canonical discussion; no mass tagging.
- Correct false claims quickly; keep the repo as the source of truth.
- Track responses in this file (date, link, outcome).

## Tracking

| Date | Channel | Link | Response |
|---|---|---|---|
| 2026-10-29 | GitHub repo + discussion | (fill after creation) | — |
| 2026-10-29 | AnyDisplay issue | (fill) | — |
| 2026-10-29 | Topti issue | (fill) | — |
| 2026-10-29 | HIDTouch issue | (fill) | — |
