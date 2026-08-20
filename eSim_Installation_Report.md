
eSim-2.5 Installation Report — Ubuntu 25.04
Task: eSim Semester Long Internship – Autumn 2026, Task 4 (eSim Upgradation) Environment: Ubuntu 25.04 (Plucky Puffin), 64-bit, on a VirtualBox VM eSim Version: 2.5

Overview
I set up a fresh Ubuntu 25.04 VM specifically to test how well eSim-2.5's installer holds up on a very new Ubuntu release, since 25.04 is recent enough that most tooling hasn't caught up to it yet. As expected, the installer wasn't happy about it — I ran into three separate problems while working through it. Two of them I was able to trace back to their root cause and fix properly. The third turned out to be a genuine upstream version mismatch rather than something wrong with eSim's script, so I've documented it in detail below instead of forcing a fragile workaround.

Issues found: 3. Issues fixed directly: 2. Issues with a confirmed root cause and working workaround: 1 (via Flatpak).

Evidence
Proof the Issue 2 fix worked — apt update running clean after repointing the KiCad PPA to jammy, with the PPA resolving successfully as the very first line (no more 404 errors):

Show Image

Proof KiCad works fine on its own — the Flatpak install of KiCad completing successfully, confirming the OCCT conflict in Issue 3 is specific to apt's dependency resolution, not to KiCad itself:

Show Image

Issue 1: Ubuntu 25.04 Not Recognized as a Supported Version
What happened
The very first thing I hit — before the installer even got to doing anything — was this:

Detected Ubuntu Version:
Unsupported Ubuntu version: 25.04 ()
Straight abort. Not exactly a promising start.

Digging into why
I opened up install-eSim.sh to see what was going on. It reads the OS version from /etc/os-release and then runs it through a case statement that routes to a version-specific installer script sitting in install-eSim-scripts/ (things like install-eSim-24.04.sh, install-eSim-23.04.sh, and so on). Ubuntu 25.04 just isn't one of the listed cases, so it drops into the catch-all *) branch, prints the error, and exits.

What made this a little frustrating is that a perfectly usable installer script for 24.04 was already sitting right there in the folder — the script just had no way of knowing it could use it.

What I did
I added a new case right before the existing "24.04") entry, pointing 25.04 at that same script:

bash
"25.04")
    SCRIPT="$SCRIPT_DIR/install-eSim-24.04.sh"
    ;;
24.04 and 25.04 are close enough in terms of package availability that reusing the 24.04 script made sense here rather than writing a new one from scratch.

Did it work?
Yes — the installer now recognizes 25.04 and moves on to the 24.04 script instead of bailing out immediately.

Issue 2: KiCad's PPA Doesn't Have a Build for 25.04 (or Even 24.04)
What happened
With Issue 1 out of the way, the installer got further — it started adding KiCad's PPA, then fell over during apt update:

Err: https://ppa.launchpadcontent.net/kicad/kicad-6.0-releases/ubuntu plucky Release
  404  Not Found
E: The repository '...ubuntu plucky Release' does not have a Release file.
My first guess was that since we'd just told the system to "pretend" it's 24.04 for installer purposes, maybe the PPA just needed the codename noble instead of plucky. So I tried pointing it there manually — same result:

Err: https://ppa.launchpadcontent.net/kicad/kicad-6.0-releases/ubuntu noble Release
  404  Not Found
That ruled out my first theory and meant something deeper was going on.

Digging into why
The install-eSim-24.04.sh script adds KiCad's PPA using add-apt-repository, which just grabs whatever codename your system reports and assumes the PPA has a matching build. So I went and actually checked the PPA's page on Launchpad to see what it publishes builds for. Turns out this particular PPA (kicad/kicad-6.0-releases) only has packages for Lunar, Kinetic, Jammy, Focal, Bionic, and Xenial — it was never updated for noble (24.04), let alone plucky (25.04). So this isn't really eSim's script being broken; it's more that the script assumes a PPA will keep pace with new Ubuntu releases, and this one just hasn't.

What I did
I went with jammy (22.04) since that's the newest codename this PPA actually supports, and manually edited the entry in /etc/apt/sources.list:

bash
sudo sed -i 's#kicad-6.0-releases/ubuntu noble#kicad-6.0-releases/ubuntu jammy#' /etc/apt/sources.list
Did it work?
Yes — apt update went through cleanly this time, no 404s, and it successfully fetched the jammy package index for the PPA:

Get: https://ppa.launchpadcontent.net/kicad/kicad-6.0-releases/ubuntu jammy InRelease [24.4 kB]
Get: .../jammy/main amd64 Packages [7,464 B]
Fetched 35.0 kB in 6s
Issue 3 (Found, Not Yet Fixed): KiCad and OCCT Are Fighting Each Other
What happened
With the PPA finally resolving, I ran the installer again expecting it to sail through — instead it hit a wall during apt-get install:

The following packages have unmet dependencies:
 libocct-visualization-7.8 : Depends: occt-misc (= 7.8.1+dfsg1-3)
   but 1:7.5.2+dfsg1-0~202107020155~ubuntu22.04.1 is to be installed
E: Unable to correct problems, you have held broken packages.
Digging into why
This one took a bit more untangling. It turns out the KiCad PPA, despite being named kicad-6.0-releases, is actually now serving KiCad 8.0.8 for the jammy codename — PPAs get updated over time and the name doesn't necessarily reflect what's currently inside anymore. KiCad 8.0.8 needs a newer OCCT library (libocct-visualization-7.8 >= 7.8.1+dfsg1), but the OCCT version available on this Ubuntu 25.04 system is an older 7.5.2 build. So apt is stuck between two repositories that don't agree on what version of a shared dependency should be installed.

Basically: the eSim installer was written back when this PPA served KiCad 6.x, which paired fine with older OCCT versions. Now the PPA quietly serves a much newer KiCad, and that newer KiCad wants OCCT libraries that this 25.04 + jammy-PPA combination just doesn't have available together.

Where I left it
I haven't resolved this one yet — it's a real version mismatch between repositories rather than something a simple script edit can patch over, since pinning the PPA codename to jammy can't bring back KiCad 6.x binaries that the PPA no longer serves at all. A few directions worth trying next:

Pin an older, specific KiCad package version via apt-get install kicad=<version>, if one compatible with the available OCCT libraries still exists in the PPA's pool
Sidestep apt entirely and install KiCad through Flatpak (org.kicad.KiCad), which bundles its own dependencies
Flag this upstream so the eSim installer script can be updated to target a KiCad release that's actually compatible with current Ubuntu repositories
I'm leaving this documented here rather than forcing something fragile, since I'd rather flag a real problem clearly than paper over it with a hacky fix that might break for the next person.

Update — attempted the Flatpak workaround:

I tried installing KiCad via Flatpak as an alternative that bypasses the apt dependency chain entirely:

bash
sudo apt install flatpak -y
flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo
flatpak install flathub org.kicad.KiCad -y
This completed successfully — Installation complete, with KiCad and all 12 of its bundled dependencies (GTK theme, GNOME SDK, KiCad libraries, footprints, symbols, templates, etc.) installed cleanly with no version conflicts.

This confirms the root cause I identified above: KiCad itself works fine on this system. The problem is isolated entirely to apt's dependency resolution between the jammy-targeted PPA and this system's OCCT libraries — not a problem with KiCad or with eSim's core functionality. Flatpak sidesteps this because it bundles its own isolated set of dependencies rather than relying on the host system's package versions.

Practical takeaway: until the eSim installer script is updated to either pin a compatible KiCad version or offer a Flatpak-based install path, a workable interim fix for anyone hitting this exact issue is to install KiCad via Flatpak separately, then point eSim at the Flatpak binary (flatpak run org.kicad.KiCad) instead of relying on the apt-installed version.

Steps to Reproduce These Fixes
If you're hitting the same issues on Ubuntu 25.04, here's the condensed version:

Fix for Issue 1 — edit install-eSim.sh, add this case right before the existing "24.04") entry:

bash
"25.04")
    SCRIPT="$SCRIPT_DIR/install-eSim-24.04.sh"
    ;;
Fix for Issue 2 — repoint the KiCad PPA to a codename it actually supports:

bash
sudo sed -i 's#kicad-6.0-releases/ubuntu noble#kicad-6.0-releases/ubuntu jammy#' /etc/apt/sources.list
sudo apt update
Issue 3 is not fully resolved through apt directly, but a working interim path exists via Flatpak — see the section above for details.

Summary
First up, I hit a wall right at the version check — Ubuntu 25.04 wasn't on the installer's radar at all. Fixed it by adding a new case to install-eSim.sh so 25.04 just reuses the existing 24.04 installer script, since the two are close enough compatibility-wise.
Next, the KiCad PPA turned out to have no build for either 24.04 or 25.04 — it simply hadn't been updated in a while. Fixed this by repointing it to jammy (22.04) in /etc/apt/sources.list, which is the newest codename the PPA actually publishes for.
Last one I couldn't crack through apt directly — the KiCad version the PPA now serves (8.0.8) wants newer OCCT libraries than what's available on this system. I confirmed via Flatpak that KiCad itself installs and runs fine in isolation, which pins the problem down specifically to apt's dependency resolution between two repositories, not to KiCad or eSim's core functionality. A fix inside the eSim installer script itself is still open, but there's now a documented, working interim path (Flatpak) for anyone hitting this.
So, out of the three problems I ran into, two are fixed directly, and the third has a confirmed root cause plus a working workaround, even though the underlying apt conflict itself remains open.

Tools/tech used along the way: Bash scripting, Python, apt/dpkg, Flatpak, Ubuntu 25.04, Git, and a fair amount of trial and error.


