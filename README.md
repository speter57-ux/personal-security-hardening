# Personal Security Hardening: Password Manager Migration and a Clean Windows 11 Install
 
*November 2026 · Sebastian Peterson*
 
## Summary
 
When Windows 10 reached end of support on **October 13, 2026**, my PC stopped getting security patches. I was also reusing passwords across many accounts. So I hardened my personal setup in two phases:
 
1. **Credential security:** I moved every password into Bitwarden, replaced reused passwords, and kept two-factor codes separate from passwords.
2. **Firmware and OS hardening:** I updated the motherboard firmware, enabled TPM 2.0 and Secure Boot, converted from legacy CSM to UEFI boot, and did a clean install of Windows 11.
This write-up covers what I did, why I did it, and the problems I solved along the way. I did all hands-on work myself, using an AI assistant as a troubleshooting partner (details in "How I used AI" below).
 
| Component | Details |
| --- | --- |
| CPU | AMD Ryzen 7 3700X |
| Motherboard | MSI X470 Gaming Plus Max (MS-7B79) |
| Storage | Samsung 860 EVO 1 TB SSD |
| Wi-Fi | TP-Link Wi-Fi 6 PCIe card (MediaTek MT7922 chip) |
| Before | Windows 10 Education, BIOS from 2020 (H61), legacy CSM boot, no TPM, Secure Boot off |
| After | Windows 11 Education (clean install), latest BIOS (HN6), UEFI boot, TPM 2.0 on, Secure Boot on |
 
---
 
## Phase 1: Credential security
 
### Choosing a password manager
 
I compared **Bitwarden** and **KeePassXC**:
 
- **KeePassXC** keeps an encrypted vault file offline that you control, but you have to handle syncing between devices yourself.
- **Bitwarden** is open source, independently audited, and syncs across devices on the free plan.
I chose Bitwarden because I use both a phone and a desktop, and a tool that syncs automatically is one I'll actually use. I also decided against building my own password manager for real use: it's easy to get encryption subtly wrong, and a homemade tool has never been audited by anyone else.
 
### Migrating safely
 
- I exported my saved passwords from Chrome and imported them into Bitwarden.
- **The export is a plain-text file containing every password**, so I destroyed it with a file shredder instead of a normal delete. I also checked that cloud sync services (OneDrive, MEGA) hadn't uploaded a copy.
- I removed the passwords from Chrome's built-in manager and turned off its offer to save passwords, so only one system holds my credentials.
### Hardening autofill
 
Autofill is usually a security win: a password manager won't fill a password on a look-alike phishing domain. One mode is risky, though:
 
- **Autofill on page load stays off.** It fills credentials with no user action, which a compromised page or hidden form can abuse. I fill logins with a click or keyboard shortcut instead.
- **Stricter matching for sensitive accounts.** Banking logins use "Host" matching, so they only fill on the exact site address I saved.
### Separating passwords and 2FA codes
 
I kept two-factor authentication codes **out of** my password manager, in a separate authenticator. If one app is ever compromised, an attacker still doesn't have both parts of the login.
 
### Fixing reused passwords
 
Reused passwords are what make **credential stuffing** work: attackers take email and password pairs leaked from one site and try them on others. I fixed mine in order of risk:
 
1. **Email first**, because it can reset every other account
2. Banking and payment apps
3. School and Microsoft, Google and Apple accounts
4. Any account found in a known data breach
5. Everything else, as I next log in to it
Each account got a randomly generated password, plus 2FA where available.
 
---
 
## Phase 2: Firmware and OS hardening
 
### Why a clean install
 
Windows 10 no longer gets security patches, so any newly discovered vulnerability would stay open on my PC permanently. I chose a **clean install** over an in-place upgrade to remove six years of leftover software, drivers and clutter.
 
### Backups first
 
Before changing anything, I backed up my files to MEGA and OneDrive and confirmed they opened from the web. I also checked for things that don't sync automatically, like local game saves and project folders. My Windows license was a digital license linked to my Microsoft account, so I confirmed that before wiping too.
 
### Updating the BIOS
 
My firmware was from 2020. I updated it to the latest version:
 
- MSI's M-Flash tool only reads **FAT32** drives, and Windows won't format a 128 GB drive as FAT32, so I used **Rufus** with its "Large FAT32" option.
- I updated the firmware **before** installing Windows. A BIOS update resets the firmware TPM, which can break Windows Hello PINs or trigger BitLocker recovery on an existing install.
### Getting into the firmware
 
This took more troubleshooting than I expected:
 
- My 60% keyboard has no dedicated Delete key, and Windows **Fast Startup** made the PC boot too quickly to catch the key. Turning off Fast Startup fixed that.
- I learned that `shutdown /r /fw /t 0` restarts straight into the UEFI firmware by setting a firmware flag:
  - **Error 1314** ("A required privilege is not held by the client") meant the command wasn't running from an administrator prompt.
  - **Error 203** happened because the board was still in legacy **CSM** mode, which doesn't support that flag. After switching to UEFI, the command worked.
### TPM 2.0, UEFI and Secure Boot
 
PC Health Check first failed because **no TPM was found**. In the firmware I:
 
1. Enabled **AMD fTPM**, the TPM built into the Ryzen CPU.
2. Switched **BIOS UEFI/CSM Mode** from CSM to **UEFI**.
3. Enabled **Secure Boot**. It first reported **Setup Mode**, meaning no keys were installed: the Platform Key, KEK and db all showed "No Key". I enrolled the factory default keys, which moved it to **User Mode**, where Secure Boot actually checks what loads at startup.
Afterward, `tpm.msc` showed the TPM as ready, `msinfo32` showed **Secure Boot State: On**, and PC Health Check passed.
 
### Clean install
 
- I made a bootable installer with Microsoft's **Media Creation Tool** and booted it from the firmware's **Boot Override** menu.
- On the partition screen, I confirmed which disk was the Windows SSD (Disk 0) and which was the flash drive (Disk 1). I deleted only Disk 0's partitions and installed to the unallocated space.
- **Network drivers:** I had no Ethernet access, so I copied the Wi-Fi driver for my card's MediaTek MT7922 chip onto the installer drive in advance, with phone USB tethering as a second backup.
- **Windows Education setup** expects a work or school account. I deliberately **did not** sign in with my university account, which could have enrolled my personal PC in the school's device management. I created a local account and linked my personal Microsoft account afterward, which kept the digital license activated.
### Post-install hygiene
 
- **Privacy:** I turned off optional diagnostic data, advertising ID, tailored experiences and location.
- **Debloating:** the motherboard's driver installer preselected more than a dozen extras, including trial antivirus, a VPN promotion and bundled apps. I installed only the chipset and audio drivers.
- **Outdated software:** when restoring backups, I found a 2022 copy of Tor Browser. I deleted it instead of restoring it, because an outdated privacy browser with years of missing security fixes is a liability. The current version should always come from the official source.
- **BitLocker:** I confirmed my drive's BitLocker recovery key is saved to my Microsoft account.
---
 
## Lessons learned
 
- **Back up and verify before changing anything.** A backup you haven't tested isn't a backup.
- **Order matters.** Firmware update → firmware settings → OS install avoids TPM resets breaking an installed OS.
- **Error codes point to the cause.** 1314 meant missing privileges, and 203 meant an unsupported firmware mode.
- **Defaults aren't always secure.** Autofill on page load, preselected bundled software and default privacy settings all needed changing.
- **Separate the factors.** Passwords and 2FA codes in different places means one breach isn't enough.
## How I used AI
 
I used an AI assistant as a troubleshooting partner throughout this project. I described my hardware and shared screenshots of BIOS screens, error messages and system tools, and used its suggestions to plan each step. I verified everything before making changes, for example confirming which disk was which before deleting partitions, and checking the results with `tpm.msc`, `msinfo32` and PC Health Check. When a step didn't work, such as error 203 when restarting into the firmware, I used the error details to find the cause and fix it. All hands-on work and final decisions were mine.
 
## Skills practiced
 
Password management and credential hygiene · two-factor authentication · secure data deletion · UEFI vs. legacy BIOS · TPM 2.0 · Secure Boot key management (PK, KEK, db, dbx) · firmware updates · Windows installation and disk partitioning · driver management · Windows privilege and boot troubleshooting · backup and recovery planning
 
## Next steps
 
- Build a home lab in **Hyper-V** (included in Windows 11 Education) for hands-on security practice
- Use **Windows Sandbox** to test unknown files safely
- Work through beginner rooms on TryHackMe and document them here
---
 
**Doing this yourself?** Back up your files first, check your motherboard maker's manual for the exact BIOS menu names, and run Microsoft's PC Health Check to see what you need to enable.
