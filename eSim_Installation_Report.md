# eSim-2.5 Installation Report — Ubuntu 25.04

**Task:** eSim Semester Long Internship – Autumn 2026, Task 4 (eSim Upgradation)
**Environment:** Ubuntu 25.04 (Plucky Puffin), 64-bit, VirtualBox VM
**eSim Version:** 2.5

## Overview

This report documents the problems encountered while installing eSim-2.5 on a fresh Ubuntu 25.04 system, along with the root-cause analysis and fixes applied. Two issues were identified and fully resolved. A third, deeper dependency conflict was identified and root-caused but left unresolved, as it stems from an upstream package version mismatch rather than a scripting bug.

## Issue 1: Ubuntu 25.04 Not Recognized as a Supported Version

### Problem
Running `./install-eSim.sh --install` immediately failed with:

Detected Ubuntu Version:
Unsupported Ubuntu version: 25.04 ()


### Root Cause
`install-eSim.sh` detects the OS version from `/etc/os-release`, then uses a `case` statement to route to a version-specific installer script. Ubuntu 25.04 is not a listed case, so it falls through to the catch-all `*)` branch and exits, even though a compatible installer (`install-eSim-24.04.sh`) already exists.

### Fix
Added a new case mapping Ubuntu 25.04 to the existing 24.04 installer script:
```bash
"25.04")
    SCRIPT="$SCRIPT_DIR/install-eSim-24.04.sh"
    ;;
```

### Result
The installer now correctly detects Ubuntu 25.04 and proceeds instead of aborting.

## Issue 2: KiCad PPA Has No Build for Ubuntu 25.04 or 24.04

### Problem
After fixing Issue 1, adding the KiCad PPA and running `apt update` failed:

404 Not Found — .../ubuntu plucky Release

Pointing it at `noble` (24.04) produced the same class of 404 error.

### Root Cause
The KiCad 6.0 PPA (`kicad/kicad-6.0-releases`) only publishes builds for Lunar, Kinetic, Jammy, Focal, Bionic, and Xenial — it was never updated for `noble` or `plucky`.

### Fix
Manually edited `/etc/apt/sources.list` to point the KiCad PPA at `jammy` (22.04), the newest codename this PPA supports:
```bash
sudo sed -i 's#kicad-6.0-releases/ubuntu noble#kicad-6.0-releases/ubuntu jammy#' /etc/apt/sources.list
```

### Result
`sudo apt update` completed successfully with no errors, fetching the KiCad PPA's jammy package index.

## Issue 3 (Identified, Unresolved): KiCad/OCCT Library Version Conflict

### Problem
With the PPA resolving correctly, `apt-get install` surfaced a dependency conflict:

libocct-visualization-7.8 : Depends: occt-misc (= 7.8.1+dfsg1-3)
but 1:7.5.2+dfsg1-0~202107020155~ubuntu22.04.1 is to be installed


### Root Cause
The KiCad PPA now serves KiCad 8.0.8 for `jammy` (not 6.0 as the PPA name implies), which requires a newer OCCT library than what's available on this Ubuntu 25.04 system. This is a genuine cross-repository version mismatch upstream, not a script bug.

### Status
Unresolved. Possible future fixes: pin an older KiCad package version if still in the PPA pool, use the Flatpak KiCad build instead, or wait for an eSim installer update targeting a current KiCad release.

## Summary

| # | Issue | Status |
|---|---|---|
| 1 | Ubuntu 25.04 not recognized | Fixed |
| 2 | KiCad PPA missing 24.04/25.04 builds | Fixed |
| 3 | KiCad 8.0.8 vs system OCCT conflict | Identified, unresolved |

**Technologies used:** Bash scripting, apt/dpkg package management, Ubuntu 25.04, Git.
