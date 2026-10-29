# macOS 27 external touchscreens — the `displayUUID` wall

**TL;DR** — macOS 27 Golden Gate contains real **native external-touch support** behind the private `SupportExternalTouchscreen` feature flag. With the flag on, macOS recognizes a generic USB HID digitizer and `NSScreen.touchCapabilities` flips to `MultiTouch`. But WindowServer refuses to associate that digitizer with any display, because the HID service has no `displayUUID` property. That property is supplied by the HID **service provider/driver at creation time**. User-space attempts to set it — including from inside WindowServer under a SIP-disabled debugger — do not persist.

No exploit or zero-day is involved. This is private-API / driver-model research on a personal machine.

---

## Why this matters

- Apple announced the first **touchscreen MacBook Pro** in October 2026 (shipping November 2026), running macOS 27. Native touch on the Mac is now real.
- The OS also contains support for **external** HID touchscreens. It is undocumented and gated by a private feature flag plus a private HID service property.
- Open-source Android-tablet-as-second-screen projects (AnyDisplay, OpenDisplay, SideScreen, MacTouch, HIDTouch) currently work at the **CGEvent injection** level. They do not get real `NSTouch`/direct-touch delivery, because they cannot provide the `displayUUID` association.
- If external native touch is meant to be available to third-party displays, there needs to be a documented API or a requestable DriverKit HID entitlement. This repo is a public data point and a place to coordinate.

## What was observed

### 1. The hidden feature flag works

The `backboardd` feature flag `SupportExternalTouchscreen` is off by default. Enabling it (via the private FeatureFlags settings plist and a reboot) makes macOS accept a generic USB HID touchscreen.

A custom USB HID gadget presenting:

- Usage Page `0x0D` (Digitizer)
- Usage `0x04` (Touch Screen)
- TipSwitch / ContactID / X / Y / ContactCount

is enumerated as `AppleUserUSBHostHIDDevice` + `AppleUserHIDEventDriver`, with `PrimaryUsagePage=0xD, PrimaryUsage=0x4`. The kernel logs:

```
IOUserHIDEventDriver.cpp: parseElements: keyboard: 0 pointer: 0 0 scroll: 0 led: 0 digitizer: 1 ...
```

And `NSScreen.touchCapabilities` for the built-in screen changes from `0x0` (`NSScreenTouchCapabilitiesNone`) to `0x1` (`NSScreenTouchCapabilitiesMultiTouch`), i.e. macOS believes a touch digitizer exists.

### 2. WindowServer rejects the digitizer

WindowServer's SkyLight / BackBoard path receives the digitizer, then logs:

```
handle_digitizer_added: digitizer service has no usable displayUUID property; treating as no digitizer added
digitizer attached (sender ...) service:<BKIOHIDService: ...; primaryUsagePage: 0xD; primaryUsage: 0x4; name: AOSP on ARM64; transport: USB; serviceStatus: matched>
```

So the device is accepted as a HID digitizer, but not associated with a display. Without the association there is no native touch delivery.

### 3. The property is driver-owned, not user-settable

Attempts from user space:

- `IOHIDDeviceSetProperty(device, "displayUUID", CFUUIDRef)` succeeds on the device object and can be read back from that object; the event service does **not** inherit it.
- `IOHIDServiceClientSetProperty(service, "displayUUID", CFUUIDRef)` returns `true`, but a subsequent `IOHIDServiceClientCopyProperty` returns `NULL`. The write is accepted syntactically but not stored/visible.
- The same no-op behavior was observed when the call was made **from inside WindowServer** during a SIP-off debugger session. We are not publishing debugger steps; the result is that even WindowServer's own user-space API view cannot change this property.
- Apple's built-in `HSTouchHIDService.plugin` (the only HID plugin containing `displayUUID` strings) matches `AppleMultitouchDevice` with property `HSTouchHIDService=true` — i.e. Apple's own built-in multitouch hardware. It resolves the display via internal display-service / built-in USB port mapping; a generic external USB HID device does not enter that path.

### 4. What this means

`displayUUID` is created/provided by the **HID service provider (driver)** when the service is created. WindowServer reads it during `handle_digitizer_added` and does not retry. To make external native touch work, someone must supply a HID service that has this association from the beginning.

## What would solve it

1. **Apple public API / entitlement**: document a supported way for an external touch display to be associated (or add external touchscreen support to the HIDDriverKit / CoreHID documentation). A requestable entitlement specifically for external touchscreen association would be ideal.
2. **DriverKit HID service/dext**: a HIDDriverKit driver that matches the USB digitizer and provides `displayUUID` at service creation. Applicable entitlements appear to be in the HID DriverKit group:
   - `com.apple.developer.driverkit`
   - `com.apple.developer.driverkit.transport.hid`
   - `com.apple.developer.driverkit.family.hid.eventservice`
3. **Vendor-signed driver** (Touch-Base/UPDD, Alogic, display vendors): if any vendor already provides native external touch on macOS 27, their model would be very useful.
4. **Virtual HID device** (`com.apple.developer.hid.virtual.device`): the Topti project has a pending request for a virtual stylus/digitizer. That path may also be relevant for a software-only tablet bridge.

If you have one of these entitlements or have seen a third-party HID service provide `displayUUID`, please open an issue or comment — that is the missing link.

## Reproduce / evidence

See [EVIDENCE.md](EVIDENCE.md) for selected logs, HID descriptor summary, and the exact observations. The feature-flag path is private/unsupported; enabling it is at your own risk.

This repository intentionally does **not** include:

- debugger/SIP-bypass instructions,
- code that patches WindowServer,
- exploit chains of any kind.

If you are an Apple engineer, a DriverKit developer, or a vendor engineer and want the detailed debugger-side evidence, open a discussion and we can share privately.

## Discussion

Use **Discussions** in this repository for:

- reports of third-party HID services that do/don't get native touch on macOS 27,
- DriverKit `IOUserHIDEventService` property behavior,
- vendor/driver contact,
- feature requests to Apple.

Please keep it constructive: no "just disable SIP" instructions, no patching other people's systems.

## 中文摘要

macOS 27 里其实有**外接原生触控**的隐藏支持：打开 `SupportExternalTouchscreen` 后，系统会把通用 USB HID digitizer 认出来，`NSScreen.touchCapabilities` 也会从 0 变成 1。但 WindowServer 在 `handle_digitizer_added` 里要求 HID service 自带 `displayUUID` 属性，否则不把 digitizer 绑到任何显示器；这个属性由 HID service 的**驱动/提供者**在创建时给出，用户态（甚至 WindowServer 内部）都写不进去。因此：

- 开源安卓平板副屏项目（AnyDisplay / OpenDisplay / SideScreen / MacTouch / HIDTouch）目前只能做 CGEvent 注入；
- 真正原生触控需要 Apple 公开 API/entitlement，或带 HIDDriverKit 权限的签名驱动；
- 如果你有相关 entitlement、做过 DriverKit HID、或见过第三方触屏驱动在 macOS 27 上工作，欢迎开 issue/discussion。

## Credits / related projects

- [AnyDisplay](https://github.com/efekurucay/anydisplay) — Android tablet as Mac second screen with touch/pen, MIT.
- [Topti](https://github.com/Yukaii/topti) — virtual stylus/digitizer via CoreHID, HID Virtual Device entitlement pending.
- [HIDTouch](https://github.com/koshi545/HIDTouch) — userspace USB HID touch driver; documents the virtual-digitizer entitlement block.
- [MacTouch](https://github.com/kashmeister/mactouch) — external touchscreen driver with CGEvent gestures.
- [OpenDisplay](https://github.com/peetzweg/opendisplay) — Mac/iPhone/iPad second display.
- [SideScreen](https://github.com/tranvuongquocdat/SideScreen) — Android tablet as Mac second screen.
- Hardware context: [Huawei MatePad 11 DBY-W09 boot research](https://github.com/glasses666/dby-w09-boot-research).

## License

MIT for the text and any future code in this repository. Logs/quotes belong to their respective owners.
