# Samsung SCX-4828 scanning on modern Windows: investigation

Status: research only. No driver code yet. See "Next steps".

## The problem
Printing works through the Samsung/HP Universal Print Driver. Scanning does not: Windows
rejects the old Samsung SCX-4x28 scan driver (TWAIN + WIA, 2009–2011 era, mostly 32-bit TWAIN).
Samsung's printer business moved to HP in 2017. HP hosts the old files but has not released a
Windows 11 scan driver for this model.

## Options considered

| # | Option | Gives scanning *inside apps*? | Effort | Verdict |
|---|--------|------------------------------|--------|---------|
| 1 | Write a new Windows driver | Yes (WIA/TWAIN) | Weeks of work, plus code signing | Possible, not recommended |
| 2 | Pull the scan driver out of the old Samsung package | Yes, if Windows accepts it | Low | Try first; depends on the exact error |
| 3 | Bridge through Linux SANE: Raspberry Pi + AirSane → Windows' built-in eSCL client | Yes, for WIA apps and Windows Scan | Low–medium, about $50 of hardware | **Recommended fallback** |
| 4 | VueScan (commercial; has its own SCX-4x28 support) | No, standalone app only | Very low | Quick stopgap |

### 1. New driver
* The scan protocol is already public. SANE's `xerox_mfp` backend (GPL) drives this model:
  USB ID `04e8:342d`, status "good".
* On Windows, a WIA minidriver runs in user mode on top of Microsoft's in-box `usbscan.sys`,
  so no kernel code would be needed.
* The blocker is distribution. 64-bit Windows 10/11 only installs signed driver packages. That
  means either test-signing mode (a watermark, and Secure Boot must be off) or Microsoft
  attestation signing, which needs an EV certificate (about $300 a year).
* Building and debugging it needs the printer attached to a Windows machine.

### 2. Reuse the old driver
The old package already *is* the scanner software, so "extracting" it just means installing its
`.inf` by hand (Device Manager → Update driver → Have Disk). Whether that works depends on why
Windows rejects it:
* **Memory Integrity (Core Isolation) block**: check Windows Security → Device security.
* **Old or invalid signature**: newer Windows builds refuse it.
* **32-bit-only TWAIN**: 64-bit apps can't see the scanner even when the driver is installed.
  WIA apps (Windows Scan, Fax and Scan, NAPS2 in WIA mode) can.
* **Windows 11 24H2 USB-scanner regressions** have also been reported.

### 3. SANE bridge (recommended)
```
laptop (Windows Scan / any WIA app) ──eSCL over Wi-Fi──► Raspberry Pi: AirSane + SANE xerox_mfp ──USB──► SCX-4828
```
* AirSane exposes SANE scanners over eSCL/AirScan.
* Windows 11, and Windows 10 with KB5014666 or later, add eSCL scanners natively:
  Settings → Printers & scanners → Add device. No vendor driver is involved, so nothing gets
  rejected.
* Printing keeps working over the printer's Ethernet port (on the -FN model), or through CUPS
  on the Pi.
* Still to verify: that the Windows eSCL device shows up for every app you use. TWAIN-only apps
  may need NAPS2 or the WIA route.

## Next steps (information needed)
1. Exact Windows version (`winver`) and the exact error message when installing the scan driver.
2. Is the printer connected by USB or by network? Is it the SCX-4828FN?
3. Which applications should scan directly, and do they use TWAIN or WIA?
