# Evidence

Selected, redacted observations from a macOS 27.0.1 (build 26A434) Apple Silicon Mac + Huawei MatePad 11 (DBY-W09, Android 12L, APatch root) acting as a USB HID gadget.

No debugger steps, SIP-bypass instructions, or exploit code are included.

## HID descriptor (70 bytes)

- Usage Page: `0x0D` Digitizer
- Usage: `0x04` Touch Screen
- Input report: TipSwitch, ContactID, X (16-bit), Y (16-bit), ContactCount
- Report length: 7 bytes
- Report example written to `/dev/hidg0`: `01 01 00 40 00 40 01`

macOS sees:

```
AppleUserUSBHostHIDDevice
PrimaryUsagePage = 0xD
PrimaryUsage = 0x4
AppleUserHIDEventDriver
DisplayIntegrated = Yes
```

```
IOUserHIDEventDriver.cpp:994 parseElements: keyboard: 0 pointer: 0 0 scroll: 0 led: 0 digitizer: 1 proximity: 0 gameController: 0
```

## Feature flag

`backboardd` / `SupportExternalTouchscreen` is off by default in macOS 27.0.1. When enabled via the private FeatureFlags settings plist and rebooted, the probe reports `1` and the HID digitizer is enumerated as above.

## touchCapabilities

- Before any digitizer / after detach: `NSScreen.touchCapabilities = 0x0`
- With the HID digitizer enumerated and feature flag on: `0x1` (`NSScreenTouchCapabilitiesMultiTouch`)

Note: this only reports that macOS believes a touch-capable digitizer exists; it does not mean WindowServer has associated it with a display.

## WindowServer logs

```
handle_digitizer_added: digitizer service has no usable displayUUID property; treating as no digitizer added
digitizer attached (sender 100002894) service:<BKIOHIDService: ...; senderID: ...; eventSource: externalAccessory; primaryUsagePage: 0xD; primaryUsage: 0x4; name: AOSP on ARM64; transport: USB; serviceStatus: matched>
```

The order can vary by milliseconds; the digitizer is added as a matched HID service but rejected for display association.

## Property behavior (user space)

1. `IOHIDDeviceSetProperty(device, "displayUUID", uuid)`
   - returns success
   - `IOHIDDeviceGetProperty` on the same device returns the value
   - the corresponding `IOHIDServiceClient` does not expose it; WindowServer still reports missing displayUUID.

2. `IOHIDServiceClientSetProperty(service, "displayUUID", uuid)`
   - returns `true`
   - subsequent `IOHIDServiceClientCopyProperty(service, "displayUUID")` returns `NULL`
   - retrying before/after WindowServer's handler does not change the outcome.

3. Inside a SIP-off debugger session, the same `IOHIDServiceClientSetProperty` call made from WindowServer's own process also returned success but did not become visible; forcing a CFUUID into the handler's post-`CopyProperty` return value did not pass WindowServer's private type/ref checks. We are not publishing the debugger procedure; it did not produce native touch and is not a solution.

## Apple's HSTouchHIDService plugin

`/System/Library/HIDPlugins/ServicePlugins/HSTouchHIDService.plugin`:

- contains the only system HID plugin strings for `displayUUID`
- matches `IOProviderClass=AppleMultitouchDevice` and `IOPropertyMatch={HSTouchHIDService=true}`
- uses internal display-service / built-in USB port mapping
- not reachable for a generic external USB HID touchscreen

## Summary

The OS recognizes the external HID digitizer; the display association is the missing piece, and it is owned by the HID service provider/driver at creation time.
