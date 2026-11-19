# Windows Autopilot v1 + v2: Danish and International Deployment

**Last updated:** 2026-11-19  
**Purpose:** Reusable Intune implementation guide for a hybrid Windows Autopilot setup.  
**Status:** Documents decisions from the implementation discussion; verify outstanding items in a pilot before rolling out.

## 1. Objective and architecture

Keep **classic Windows Autopilot (v1)** as the primary provisioning method for both Danish (DK) and international (INT) computers. Provide **Windows Autopilot device preparation (v2)** as an additional route **only for international users**, when a new computer is not registered in classic Autopilot.

| Scenario | Enrollment route | Result |
| --- | --- | --- |
| Classic-registered device, Group Tag `DK` | v1 `AP-W11-DK` | Danish OOBE |
| Classic-registered device, Group Tag `INT` | v1 `AP-W11-INT` | International OOBE |
| Unregistered device, eligible international user | v2 `AP-v2-INT` | Entra joined and Intune enrolled |
| Unregistered device, Danish user | No INT v2 assignment | Use the DK v1 registration process |

**Important:** v2 is **not an automatic recovery** mechanism for a registered classic Autopilot device whose v1 deployment fails. Registered classic devices generally receive their v1 profile; unregistered devices can use v2 when a targeted user signs in and requirements are met. Newer v2 device association can affect precedence; the unassociated fallback workflow described here does not use it.

## 2. Group design

| Group | Group type | Usage |
| --- | --- | --- |
| `GRP-AP-DK` | Dynamic device | Classic DK devices |
| `GRP-AP-INT` | Dynamic device | Classic INT devices |
| `GRP-AP-Productions` | Dynamic device | Shared classic Autopilot registered devices |
| `GRP-AP-v2-INT-Users` (example) | Dynamic user | Assignment target for v2 policy |
| `GRP-AP-v2-INT-Devices` | **Assigned** security group | Empty initially; v2 adds enrolled devices |

### Classic device dynamic rules

DK:

```text
(device.devicePhysicalIds -any (_ -eq "[OrderID]:DK"))
```

INT:

```text
(device.devicePhysicalIds -any (_ -eq "[OrderID]:INT"))
```

Common registered Autopilot devices:

```text
(device.devicePhysicalIds -any (_ -startsWith "[ZTDid]"))
```

### International dynamic user rule

Entra ID → Groups → New group → Security → Membership type **Dynamic User**. Use:

```text
(user.accountEnabled -eq true) -and
(user.companyName -ne null) -and
(user.companyName -ne "") -and
(user.usageLocation -ne null) -and
(user.usageLocation -ne "DK")
```

This selects enabled users with a populated company name and a configured usage location other than Denmark. Validate membership against actual users: *Usage location* is an account attribute, not proof of where someone works. If company matching must be stricter, replace “not empty” with the intended company name.

### Required v2 device-group configuration

- Security group, **Assigned** membership, not role-assignable.
- No members needed before deployment.
- Add **Intune Provisioning Client** as group **Owner**, application ID `f1346770-5b25-470b-88bd-d5744ab7952c`.
- The admin editing the v2 policy needs the **Enrollment time device membership assignment** permission.

If adding the empty device group to v2 fails, first check its type, owner and the Intune RBAC permission.

## 3. Classic Autopilot v1 configuration

Maintain both existing profiles:

| Profile | Assigned group | OOBE region/language | Auto keyboard | Device name |
| --- | --- | --- | --- | --- |
| `AP-W11-DK` | `GRP-AP-DK` | Danish (Denmark) | Yes | Keep existing DK naming |
| `AP-W11-INT` | `GRP-AP-INT` | Operating system default/user-selected | No | `INT-%SERIAL%` |

These are user-driven Microsoft Entra joined deployments.

**Avoid overlapping classic deployment profile assignments.** Use `GRP-AP-Productions` for common management where useful, but do not assign a competing classic Autopilot profile through it.

### Classic manual hardware-hash registration

During Windows OOBE, open Command Prompt with `Shift + F10`, then start PowerShell. With writable USB at `D:\`:

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass -Force
Install-Script -Name Get-WindowsAutoPilotInfo -Force
Get-WindowsAutoPilotInfo.ps1 -OutputFile "D:\Autopilot.csv"
```

Verify that `D:\` is really the USB drive; it is not guaranteed on every PC. Script installation requires internet access. The execution-policy bypass only applies to the current process and does not override an enforced enterprise policy.

**Do not specify `-Partner`** for standard Intune classic Autopilot import. The `-Partner` export is intended for Partner Center.

In Intune: **Devices → Windows → Enrollment → Windows Autopilot devices → Import**, upload CSV, refresh, set Group Tag `DK` or `INT`, then verify the right profile is **Assigned**.

### Partner Center lesson learned

Partner Center registration using manufacturer/model/serial can fail unless prerequisites for the hardware/vendor identity are satisfied. We saw `ZtdDeviceNotFound` when testing such registration for a Lenovo device. Where the supplier has not provided/registered the needed identity, use manual hardware-hash import to classic Autopilot—or the v2 international fallback where appropriate.

## 4. Autopilot v2 configuration: international fallback

Intune → **Devices → Windows → Enrollment → Device preparation policies** → Create `AP-v2-INT`.

| Configuration | Selection |
| --- | --- |
| Join type / mode | **User-driven Microsoft Entra join** |
| User account type | **Standard** |
| **Assignment** | Dynamic international **user** group |
| **Device group** | `GRP-AP-v2-INT-Devices` (Assigned; initially empty) |
| Installation timeout | 60 minutes (chosen for this configuration) |
| Allow skip after repeated failures | No (chosen for mandatory provisioning) |
| Diagnostics link | Yes |
| Language | Operating system default |
| Automatically configure keyboard | No |
| License terms / privacy prompts | Hide where available |
| Apply device name template | **No** for unassociated fallback devices |

Select only required applications and scripts for the **device preparation stage**. Other standard Intune apps/policies can be assigned to common management groups. Ensure the requirements for successful device preparation (supported Windows 11, licensing, network, Microsoft Entra join rights, automatic Intune enrollment) are satisfied.

**Enrollment restrictions:** If personal Windows enrollment is blocked, check Microsoft's corporate device identifiers or device association requirements. A new PC being company-owned in practice does not automatically mean Intune classifies it as corporate-owned.

**Expected initial flow:** A new PC that is *not* classic Autopilot-registered reaches supported OOBE; an eligible international employee signs in; the v2 policy applies; the PC Entra joins and enrolls in Intune; the provisioning service adds it to the assigned v2 device group.

## 5. Converting v2 devices into classic v1 registrations

**Goal:** A PC initially deployed through v2 should be registered in classic Windows Autopilot for *subsequent* resets.

Native feature: **Convert all targeted devices to Autopilot = Yes** on a classic Autopilot deployment profile. This registers eligible, **corporate-owned** Intune-managed devices automatically; Microsoft advises allowing up to 48 hours.

Because `GRP-AP-INT` relies on a pre-existing **Group Tag** `INT`, a v2-only PC will **not** automatically join that tag-based group.

### Recommended conversion approach

1. Keep `AP-W11-INT` and `GRP-AP-INT` for originally registered INT devices.
2. Create a dedicated **classic** profile named `AP-v1-INT-Conversion`, matching the international OOBE and `INT-%SERIAL%` naming configuration.
3. Set **Convert all targeted devices to Autopilot = Yes**.
4. Assign this classic conversion profile to `GRP-AP-v2-INT-Devices`.
5. Verify the v2-enrolled devices are classified **corporate-owned**. If not, correct the enrollment identification/ownership route before relying on automatic registration.
6. Allow conversion and verify each device appears under **Windows Autopilot devices**, with appropriate classic profile assignment.

**Important limitations:**
- Conversion is **registration**, not a fresh OOBE run, and it does not immediately replace the running Windows installation.
- **No Group Tag `INT` is automatically assigned.** Devices registered by conversion remain in the v2 group and use the conversion profile for subsequent classic deployment, assuming their group targeting remains applicable.
- If the business explicitly requires converted devices to be members of `GRP-AP-INT`, implement and test separate **Microsoft Graph** automation to apply Group Tag `INT` after classic registration. This was discussed but **not implemented**.
- Pilot this configuration before production. Verify that targeting and policy precedence remain correct after reset and throughout group membership changes.

## 6. Consistent `INT-%SERIAL%` naming

### Classic v1

Set the v1 international deployment profile device-name template:

```text
INT-%SERIAL%
```

### Unassociated v2 fallback

The v2 device naming template requires **device association** to rename within OOBE. For devices using v2 *without* preregistration/association, leave the v2 template disabled and rename after enrollment.

Intune platform script (PowerShell), targeted to **`GRP-AP-v2-INT-Devices`**:

```powershell
$Serial = (Get-CimInstance Win32_BIOS).SerialNumber.Trim()
$NewName = "INT-$Serial"

# Windows computer names are limited to 15 characters.
$NewName = $NewName.Substring(0, [Math]::Min(15, $NewName.Length))

if ($env:COMPUTERNAME -ne $NewName) {
    Rename-Computer -NewName $NewName -Force
}
```

Intune → **Devices → Scripts and remediations → Platform scripts**:
- Run using logged-on credentials: **No** (SYSTEM).
- Run in 64-bit PowerShell host: **Yes**.
- Target **v2 device group**.
- Treat it as a **post-enrollment** script, not a required device preparation script.

**No forced restart:** `Rename-Computer -Force` does **not** force a reboot. The new name becomes fully effective on the **next normal restart**. Do not add `Restart-Computer -Force` unless operations explicitly require an immediate reboot. Intune/Entra name displays may update after a later restart/check-in.

**Serial-number caveat:** Truncation may cause duplicate names if long serial numbers share the first 11 characters after the `INT-` prefix. Test across hardware models and choose a collision-resistant strategy if needed. PowerShell interpolates `"INT-$Serial"`; single quotes would not expand `$Serial`.

## 7. Operational verification and monitoring

### After v1 registration

1. Device appears in **Windows Autopilot devices**.
2. Group Tag is `DK` or `INT`.
3. Device is in the corresponding dynamic group.
4. Correct `AP-W11-DK` or `AP-W11-INT` profile is assigned.
5. Complete OOBE, verify apps/security controls.

### After v2 enrollment

1. Device shows in **Intune Windows devices** and **Entra ID devices**.
2. Device is a member of `GRP-AP-v2-INT-Devices`.
3. Deployment reports show the v2 provisioning results.
4. Post-enrollment script stages `INT-<serial>` rename; check after next normal reboot.
5. Device ownership is **Corporate** if relying on automatic v1 conversion.
6. Within the Microsoft-documented registration window, confirm classic Autopilot registration and assignment of the **conversion profile**.
7. Test a later reset on a pilot to validate classic profile behavior.

**Classic Autopilot devices list:** v2-only enrollments do **not** appear there until classic registration/conversion succeeds.

## 8. Troubleshooting

| Symptom | Check |
| --- | --- |
| v2 policy cannot save selected empty device group | Group is Assigned Security; **Intune Provisioning Client** owner; correct Intune RBAC |
| Unregistered INT PC does not receive v2 | User group membership, Windows OOBE version, enrollment eligibility, personal-device blocking, licensing |
| Registered v1 PC does not use v2 | Normal behavior: classic registration/profile takes precedence in standard setup |
| v2 device does not auto-register in v1 | Corporate ownership, classic conversion profile targeting, group membership, elapsed sync time |
| v2 device is missing from `GRP-AP-INT` | No Group Tag is assigned automatically on v2-to-v1 conversion |
| Serial rename does not display yet | Restart needed for name to take full effect; verify script result and check-in |
| `Install-Script` says script execution disabled | Process-scoped execution-policy bypass (if not centrally enforced) |
| Partner Center registration returns `ZtdDeviceNotFound` | Vendor/supplier registration prerequisites; fall back to classic hardware hash or v2 |

## 9. Production checklist

- [ ] Existing `AP-W11-DK` and `AP-W11-INT` remain assigned to their own Group Tag groups.
- [ ] No overlapping classic Autopilot profile assignments.
- [ ] Dynamic INT user rule has been validated using representative users.
- [ ] v2 device group is Assigned, initially empty, and service principal is an owner.
- [ ] v2 policy targets **only** international users.
- [ ] Device enrollment restrictions allow intended fallback hardware.
- [ ] Required apps, baseline policies, and security controls are assigned correctly to v2 devices.
- [ ] Post-enrollment rename script validated with no forced restart.
- [ ] Corporate device ownership validated for conversion.
- [ ] `AP-v1-INT-Conversion` profile piloted, including future reset.
- [ ] Report and document exceptions, failures and device-name collisions.

## References

- [Windows Autopilot device preparation overview](https://learn.microsoft.com/en-us/autopilot/device-preparation/overview)
- [Device preparation user-driven workflow](https://learn.microsoft.com/en-us/autopilot/device-preparation/tutorial/user-driven/entra-join-workflow)
- [Create a device preparation policy](https://learn.microsoft.com/en-us/autopilot/device-preparation/tutorial/user-driven/entra-join-autopilot-policy)
- [Device preparation troubleshooting FAQ](https://learn.microsoft.com/en-us/autopilot/device-preparation/troubleshooting-faq)
- [Classic Autopilot manual device registration](https://learn.microsoft.com/en-us/autopilot/add-devices)
- [Automatic registration of existing devices](https://learn.microsoft.com/en-us/autopilot/automatic-registration)
- [Dynamic group membership rules](https://learn.microsoft.com/en-us/entra/identity/users/groups-dynamic-membership)
