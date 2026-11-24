# Windows Autopilot v1 + v2: Danish and International Deployment

**Last updated:** 2026-11-23  
**Purpose:** Reusable Intune implementation guide for a hybrid Windows Autopilot setup, with a default international v1 profile and DK-specific override.  
**Status:** Reusable implementation design; validate outstanding items in a pilot before rollout. **Replace `CUSTOMER-PREFIX` with an actual short customer code.** See [Naming Standards](../../naming%20standard/NAMING-STANDARDS.md).

## 1. Objective and architecture

Keep **classic Windows Autopilot (v1)** as the primary provisioning method for both Danish (DK) and international (INT) computers. Provide **Windows Autopilot device preparation (v2)** as an additional route **only for international users**, when a new computer is not registered in classic Autopilot.

| Scenario | Enrollment route | Result |
| --- | --- | --- |
| Classic-registered device, Group Tag `DK` | v1 `CUSTOMER-PREFIX_Autopilot_v1_deployment_profile_DK` | Danish OOBE |
| Classic-registered device, no Group Tag (default) | v1 `CUSTOMER-PREFIX_Autopilot_v1_deployment_profile_Default` | International OOBE |
| Unregistered device, eligible international user | v2 `CUSTOMER-PREFIX_Autopilot_v2_deployment_profile_INT` | Entra joined and Intune enrolled |
| Unregistered device, Danish user | No INT v2 assignment | Use the DK v1 registration process |

**Important:** v2 is **not an automatic recovery** mechanism for a registered classic Autopilot device whose v1 deployment fails. Registered classic devices generally receive their v1 profile; unregistered devices can use v2 when a targeted user signs in and requirements are met. Newer v2 device association can affect precedence; the unassociated fallback workflow described here does not use it.


## Implementation runbook — exact portal steps

Follow these steps in order for a new tenant. For this existing customer, **reuse groups and profiles that are already in place** instead of creating duplicate objects. Portal labels can move between Intune releases.

### Step 1 — Check the prerequisites

1. Sign in to [Microsoft Intune admin center](https://intune.microsoft.com).
2. Open **Devices → Windows → Enrollment** and confirm **Windows Autopilot** and **Windows Autopilot device preparation** are available.
3. In [Microsoft Entra admin center](https://entra.microsoft.com), open **Identity → Devices → Overview → Device settings**. Confirm the target users may join devices to Microsoft Entra ID.
4. In Intune, check **Devices → Enrollment → Automatic Enrollment** (portal location may vary). Confirm the **MDM user scope** includes both DK and INT users intended for provisioning.
5. Verify Windows licensing and Intune user licensing, supported Windows 11 build, internet access and access to Microsoft sign-in endpoints.
6. Check **Devices → Enrollment → Enrollment restrictions**. If personally owned Windows devices are blocked, plan to upload **corporate identifiers** or use **device association** for v2 before the user starts OOBE. Otherwise v2 enrollment might be blocked.
7. Check Windows Enrollment Status Page (ESP) is appropriately assigned for v1; v2 uses its own device preparation progress experience.

### Step 2 — Create the DK dynamic device group

In [Microsoft Entra admin center](https://entra.microsoft.com):

1. Go to **Identity → Groups → All groups → New group**.
2. Set **Group type = Security**.
3. Enter **Group name = DG_Intune_Autopilot_v1_Devices_DK**.
4. Set **Membership type = Dynamic Device**.
5. Open **Add dynamic query → Edit** and enter:

~~~text
(device.devicePhysicalIds -any (_ -eq "[OrderID]:DK"))
~~~

6. Save the rule and select **Create**.

**Why this works:** Classic Autopilot **Group Tag DK** is represented in Entra as `[OrderID]:DK`; the rule picks those devices for the Danish profile. The international v1 profile is assigned directly to Intune's built-in **All devices** target—no other group is necessary for that assignment.

**Optional other regions:** Group Tags can hold a country code, region, customer label or other meaningful value, for example `SE`, `US` or `BranchA`. Create a matching dynamic device group and specific deployment profile **only if you need a different configuration**. This guide implements the **DK** tag only. Group Tags are not Intune *Scope tags* or Entra *Device categories*.

### Step 3 — Create / verify classic Autopilot deployment profiles

In [Intune](https://intune.microsoft.com):

1. Go to **Devices → Windows → Enrollment → Deployment profiles** under **Windows Autopilot**.
2. Choose **Create profile → Windows PC**. If **CUSTOMER-PREFIX_Autopilot_v1_deployment_profile_DK** already exists, open it and choose **Settings** to edit OOBE values instead.
3. Enter **Name = CUSTOMER-PREFIX_Autopilot_v1_deployment_profile_DK**.
4. Set **Convert all targeted devices to Autopilot = No** on both DK and the All devices default profile. In particular, do **not** enable conversion on the All devices default or it could register unrelated corporate Intune devices.
5. In **Out-of-box experience**, set:

| Setting | CUSTOMER-PREFIX_Autopilot_v1_deployment_profile_DK |
| --- | --- |
| Deployment mode | User-driven |
| Join to Microsoft Entra ID as | Microsoft Entra joined |
| User account type | Standard |
| Language (Region) | Danish (Denmark) |
| Automatically configure keyboard | Yes |
| Apply device name template | **Yes → CUSTOMER-PREFIX-%SERIAL%** |
| Microsoft Software License Terms | Hide |
| Privacy settings | Hide |
| Hide change account options | Hide (requires Entra company branding) |

6. On **Assignments**, choose **Selected groups → Select groups to include → DG_Intune_Autopilot_v1_Devices_DK**. Review and create/save. This specific group assignment overrides the default All devices profile for DK devices.
7. Create or edit **CUSTOMER-PREFIX_Autopilot_v1_deployment_profile_Default** with these differences:

| Setting | CUSTOMER-PREFIX_Autopilot_v1_deployment_profile_Default |
| --- | --- |
| Language (Region) | Operating system default |
| Automatically configure keyboard | No |
| Apply device name template | **Yes → CUSTOMER-PREFIX-%SERIAL%** |
| Assignment | **All devices** (default); do not require Group Tag |

8. Keep the other deployment settings aligned with the DK profile unless your customer requires differences.
9. **DK devices are deliberately targeted by both profiles:** the DK profile via the specific dynamic group and the international default via All devices. The specifically assigned DK profile should win; confirm the actual assigned profile on a pilot DK computer before OOBE. Avoid any *additional* overlapping specifically targeted classic profiles.

**Important:** Windows computer names can be up to 15 characters long. Test serial lengths for the actual Lenovo models. Native v1 naming and post-enrollment v2 truncation may not behave identically for unusually long serials.

### Step 4 — Register a device for classic v1

**Supplier/OEM registration (preferred where available):**

1. Ask the supplier to register the device in the correct tenant.
2. In **Intune → Devices → Windows → Enrollment → Windows Autopilot → Devices**, find its serial number.
3. For Danish devices, select it and set **Group tag = DK**. Leave the tag blank for the default international deployment.
4. Verify the correct deployment profile shows **Assigned**. DK must resolve to the DK profile; an untagged international device must resolve to the All devices default.

**Manual hardware-hash registration:**

1. At Windows OOBE, connect to the internet and press **Shift + F10**.
2. Enter `powershell` in Command Prompt.
3. Connect a USB drive and check its letter with `Get-Volume`. The example below uses `D:` **only when D: is your USB**.
4. Run:

~~~powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass -Force
Install-Script -Name Get-WindowsAutoPilotInfo -Force
Get-WindowsAutoPilotInfo.ps1 -OutputFile "D:\Autopilot.csv"
~~~

5. From an administrative computer, open **Intune → Devices → Windows → Enrollment → Windows Autopilot → Devices → Import** and upload `Autopilot.csv`.
6. Select **Sync/Refresh** if needed and wait for registration.
7. For a Danish device, configure **Group tag = DK** and verify DK group membership; for international devices, leave the Group Tag empty.
8. Confirm the correct v1 profile is assigned **before** proceeding through OOBE.

The process-scoped execution policy is temporary; an enterprise-enforced policy can still block execution. `Install-Script` requires internet access.

**Extending the tagging approach:** This guide only uses **DK**. Other projects can select any appropriate Group Tag and create a matching dynamic membership rule to deliver a specialized profile.

### Step 5 — Create the dynamic international user group for v2

In [Entra admin center](https://entra.microsoft.com):

1. Navigate to **Identity → Groups → All groups → New group**.
2. Choose **Security**, name **GRP-CUSTOMER-PREFIX_Autopilot_v2_deployment_profile_Users_INT** (or reuse the international user group already created).
3. Set **Membership type = Dynamic User** and open the advanced rule editor.
4. Paste:

~~~text
(user.accountEnabled -eq true) -and
(user.companyName -ne null) -and
(user.companyName -ne "") -and
(user.usageLocation -ne null) -and
(user.usageLocation -ne "DK")
~~~

5. Save the rule and create the group.
6. Open the group, check **Members**, and validate multiple real INT and DK users before assigning a deployment policy.

**Rule semantics:** The example requires an enabled user, nonempty company name and a defined non-DK usage location. If you intend to include users with *blank* Usage Location, remove only the **(user.usageLocation -ne null)** clause. Confirm that `usageLocation` is accurate before using it to separate your users.

### Step 6 — Create the v2 assigned device group and its service principal owner

In **Entra admin center → Identity → Groups → All groups → New group**:

1. Enter **Group name = GRP-CUSTOMER-PREFIX_Autopilot_v2_deployment_profile_Devices_INT**.
2. Set **Group type = Security** and **Membership type = Assigned**.
3. Set **Microsoft Entra roles can be assigned to the group = No**.
4. On **Owners**, choose **Add owners** and search **Intune Provisioning Client**.
5. Verify **Application ID = f1346770-5b25-470b-88bd-d5744ab7952c**. It might show as **Intune Autopilot ConfidentialClient** instead.
6. Select it as the **Owner** and create/save the group.
7. Leave **Members empty**. v2 populates this group during device preparation.

**If the v2 policy refuses to save the group:** verify the group is an **Assigned** security group, the provisioning service principal is an **Owner**, and the admin has the Intune RBAC **Enrollment time device membership assignment** permission.

Reference: [Microsoft — Create assigned device group](https://learn.microsoft.com/en-us/autopilot/device-preparation/tutorial/user-driven/entra-join-device-group).

### Step 7 — Assign apps and prepare the v2 device preparation policy

**Required app targeting:** First assign critical apps as **Required** to **GRP-CUSTOMER-PREFIX_Autopilot_v2_deployment_profile_Devices_INT** in **Intune → Apps → All apps → [App] → Properties → Assignments**. Choose system-context installation where applicable. Keep the preparation-stage list short.

Now create the v2 policy:

1. Open **Intune → Devices → Windows → Enrollment → Device preparation policies**.
2. Select **Create → User Driven**.
3. **Basics:** enter **Name = CUSTOMER-PREFIX_Autopilot_v2_deployment_profile_INT**, then **Next**.
4. **Device group:** choose **GRP-CUSTOMER-PREFIX_Autopilot_v2_deployment_profile_Devices_INT** (this is the *device* group, not the users group).
5. **Configuration settings → Deployment settings:**
   - **Deployment mode = User-driven**
   - **Deployment type = Single user**
   - **Join type = Microsoft Entra joined**
   - **User account type = Standard User**
6. **Out-of-box experience settings:**

| Setting | International fallback |
| --- | --- |
| Minutes allowed before showing installation error | 60 |
| Custom error message | Contact your IT support |
| Allow users to skip setup after multiple attempts | No |
| Show link to diagnostics | Yes |
| Language (Region) | Operating system default |
| Automatically configure keyboard | No |
| Hide Microsoft Software License Terms | Hide |
| Hide privacy settings | Hide |
| Hide change account options | Hide |
| Apply device name template | **No** (for the unassociated v2 route) |

7. **Applications:** add only the critical, already-assigned required apps you want tracked during device preparation.
8. **Scripts:** select only scripts that must finish *during OOBE*. **Do not add the post-enrollment computer rename script here**.
9. **Scope tags:** retain Default unless RBAC requires a custom scope tag.
10. **Assignments:** select **GRP-CUSTOMER-PREFIX_Autopilot_v2_deployment_profile_Users_INT** (**user** group, not devices group).
11. Select **Review + create → Save**. Confirm the policy shows in **Device preparation policies**.

**Selection logic:** An INT user with a classic-registered, assigned v1 PC receives its v1 profile. An eligible INT user on a PC without classic registration can receive v2 when they sign in. **This is not a fallback for a v1 deployment that has already failed.**

### Step 8 — Deploy post-enrollment naming for unassociated v2 devices

Standard unassociated v2 deployments do not apply the built-in naming template during OOBE. Rename the PC **after** Intune enrollment using a Platform script.

1. Save the following script as **Rename-Customer-Serial.ps1**. The script uses `CUSTOMER_PREFIX-PC` as a placeholder; replace `CUSTOMER_PREFIX` with the actual short customer code before deployment.

~~~powershell
# Predefined variables for naming standard for CUSTOMER_NAME
$CUSTOMER_NAME = "CUSTOMER_PREFIX"

# Get the device serial number
$Serial = (Get-CimInstance Win32_BIOS).SerialNumber.Trim()

# Change name to CUSTOMER_PREFIX-PC Serial number
$NewName = "$CUSTOMER_NAME-$Serial"
$NewName = $NewName.Substring(0, [Math]::Min(15, $NewName.Length))

if ($env:COMPUTERNAME -ne $NewName) {
    Rename-Computer -NewName $NewName -Force
}
~~~

**Script blocks:** (1) `$CUSTOMER_NAME` provides the customer-specific naming prefix. (2) CIM reads the hardware BIOS serial number. (3) The rename block constructs the computer name, limits it to 15 characters, and only invokes `Rename-Computer` when a change is needed.

2. Open **Intune → Devices → Scripts and remediations → Platform scripts → Add → Windows 10 and later**.
3. Enter **CUSTOMER-PREFIX_Autopilot_v2_deployment_profile_INT-Rename-Serial** and upload `Rename-Customer-Serial.ps1`.
4. Set **Run this script using the logged-on credentials = No**.
5. Set **Run script in 64-bit PowerShell host = Yes**.
6. Assign the script to **GRP-CUSTOMER-PREFIX_Autopilot_v2_deployment_profile_Devices_INT**.
7. Complete creation and review **Device status** after v2 enrollment.

**No forced restart:** `Rename-Computer -Force` suppresses confirmation; the script does not call `Restart-Computer`. The changed name takes full effect on the next normal restart.

**Naming length:** Windows computer names should stay within 15 characters for compatibility. Truncation can produce duplicates (especially for long serial numbers), so check real device prefixes and serial lengths. `CUSTOMER_PREFIX-PC` is a placeholder, not a literal Windows computer name; substitute a short prefix before deployment.

### Step 9 — Automatically register v2-enrolled PCs in classic Autopilot

**Purpose:** An international PC can use v2 for its first enrollment and become a **classic Autopilot-registered** device afterward, so later resets can use a classic deployment profile. This is a **planned configuration requiring pilot verification**, not a confirmed production result.

1. In **Intune → Devices → Windows → Enrollment → Windows Autopilot → Deployment profiles**, choose **Create profile → Windows PC**.
2. **Name:** `CUSTOMER-PREFIX_Autopilot_v1_conversion_deployment_profile_INT`.
3. Set **Convert all targeted devices to Autopilot = Yes**.
4. Configure **User-driven**, **Microsoft Entra joined**, **Standard user**, **Language = Operating system default**, **Automatically configure keyboard = No** and the customer-specific `CUSTOMER-PREFIX-%SERIAL%` naming template.
5. Match the remaining international OOBE settings to `CUSTOMER-PREFIX_Autopilot_v1_deployment_profile_Default`.
6. On **Assignments**, select **GRP-CUSTOMER-PREFIX_Autopilot_v2_deployment_profile_Devices_INT** as the specifically targeted device group.
7. Leave **Convert all targeted devices to Autopilot = No** on the main **All devices** default profile. This avoids unintentionally converting unrelated corporate Intune devices.
8. Enroll an international test PC through v2; verify **Devices → Windows → Windows devices → [device] → Ownership = Corporate**. Automatic classic registration requires an eligible corporate-owned Intune device.
9. Allow up to **48 hours** for classic Autopilot registration, then review **Windows Autopilot → Devices** and the **Profile status** of the device.
10. Validate which **classic deployment profile** is selected. The conversion profile targets a specific device group whereas the default targets All devices; confirm the expected precedence on the actual pilot.
11. Test a **subsequent reset** only after verifying registration, assignment, naming and continued device-group membership.

**No Group Tag is required for international devices in this design.** The conversion profile uses the v2 **device group** for targeting, not a tag-based dynamic group. Do not add a second dynamic group merely for conversion. Note that a device's conversion registers it in classic Autopilot; it does not run a new OOBE or rename the running installation immediately.

Microsoft references: [Windows Autopilot profile assignment](https://learn.microsoft.com/en-us/autopilot/profiles) and [device groups / automatic conversion](https://learn.microsoft.com/en-us/autopilot/enrollment-autopilot).

### Step 10 — Test and validate the three expected flows

| Pilot | Before OOBE | Expected initial result | After deployment |
| --- | --- | --- | --- |
| DK-v1 | Classic registered, Group Tag DK | CUSTOMER-PREFIX_Autopilot_v1_deployment_profile_DK (specific assignment wins) | Device name **<CUSTOMER-PREFIX>-<serial>** |
| INT-v1 | Classic registered, **no Group Tag** | CUSTOMER-PREFIX_Autopilot_v1_deployment_profile_Default (All devices) | Device name **<CUSTOMER-PREFIX>-<serial>** |
| INT-v2 | **Not** classic registered; eligible INT user | CUSTOMER-PREFIX_Autopilot_v2_deployment_profile_INT | Post-enrollment rename to **<CUSTOMER-PREFIX>-<serial>**, then optional v1 auto-registration |

For each pilot:

1. Check **Entra ID → Groups → [Target group] → Members**.
2. On v1 devices: check **Intune → Windows → Enrollment → Windows Autopilot → Devices** shows **Profile status = Assigned**.
3. On v2 devices: check **Intune → Devices → Windows → Windows devices** and **Windows Autopilot device preparation deployment reports**.
4. Verify intended required apps, standard-user rights, Defender/security profiles and correct serial-based naming.
5. For v2, perform a normal restart later, confirm the new computer name, then monitor corporate ownership and classic registration.
6. Document failures before enabling production-scale enrollment.

## 2. Group design

| Group | Membership | Usage |
| --- | --- | --- |
| `DG_Intune_Autopilot_v1_Devices_DK` | Dynamic device | DK-tagged classic Autopilot PCs |
| `GRP-CUSTOMER-PREFIX_Autopilot_v2_deployment_profile_Users_INT` | Dynamic user | International user assignment to v2 |
| `GRP-CUSTOMER-PREFIX_Autopilot_v2_deployment_profile_Devices_INT` | Assigned security group | Receives v2 devices during enrollment |

**Built-in Intune All devices** is the target for the default international classic profile. It is **not** a separately created Entra dynamic group.

### Dynamic device rules and Group Tags

The **only Group Tag required in this guide is `DK`**:

```text
(device.devicePhysicalIds -any (_ -eq "[OrderID]:DK"))
```

A Group Tag can be set to any meaningful country code, location or organizational name. If a future deployment needs different OOBE settings, create a dynamic device group and a specifically assigned classic profile using the desired tag, for example `[OrderID]:SE` for Swedish devices. Otherwise, devices use the default international profile without any Group Tag.

### International dynamic user rule

Set the following in **Entra → Identity → Groups → Dynamic User membership rule**:

```text
(user.accountEnabled -eq true) -and
(user.companyName -ne null) -and
(user.companyName -ne "") -and
(user.usageLocation -ne null) -and
(user.usageLocation -ne "DK")
```

This matches **enabled users** with a populated company name and a defined usage location other than Denmark. Verify the selected users before assigning production policies.

### Required v2 device-group configuration

- **Security** group with **Assigned** membership and no required initial members.
- Add the **Intune Provisioning Client** enterprise application/service principal as an **Owner**, application ID `f1346770-5b25-470b-88bd-d5744ab7952c`.
- The administering role must have the Intune permission **Enrollment time device membership assignment**.

If v2 cannot save the assigned device group, validate group type, owner and RBAC permissions.

## 3. Classic Autopilot v1 configuration

| Profile | Assignment | Language | Keyboard | Device-name template |
| --- | --- | --- | --- | --- |
| `CUSTOMER-PREFIX_Autopilot_v1_deployment_profile_DK` | `DG_Intune_Autopilot_v1_Devices_DK` | Danish (Denmark) | Automatic | `CUSTOMER-PREFIX-%SERIAL%` |
| `CUSTOMER-PREFIX_Autopilot_v1_deployment_profile_Default` | **All devices** | Operating system default | User-selectable | `CUSTOMER-PREFIX-%SERIAL%` |

Both are **user-driven Microsoft Entra joined** profiles. The specifically assigned DK profile takes precedence over the default All devices profile. Confirm the **Assigned** profile before OOBE.

### Classic manual hardware-hash registration

At Windows OOBE, open PowerShell with **Shift + F10**, and use a USB drive confirmed to be `D:`:

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass -Force
Install-Script -Name Get-WindowsAutoPilotInfo -Force
Get-WindowsAutoPilotInfo.ps1 -OutputFile "D:\Autopilot.csv"
```

In **Intune → Devices → Windows → Enrollment → Windows Autopilot → Devices → Import**, upload the CSV and check successful registration. Set **Group Tag DK** only on Danish PCs, then verify the correct classic profile. The execution-policy change is temporary and doesn't override enterprise policy.

### Partner Center background

Partner Center registration with only serial/manufacturer/model details can depend on OEM/supplier registration prerequisites. For devices that can't be registered this way, manual hardware-hash collection is an available v1 route; international users can also use v2 on otherwise eligible, unregistered PCs.

## 4. Autopilot v2 configuration: international fallback

Intune → **Devices → Windows → Enrollment → Device preparation policies** → Create `CUSTOMER-PREFIX_Autopilot_v2_deployment_profile_INT`.

| Configuration | Selection |
| --- | --- |
| Join type / mode | **User-driven Microsoft Entra join** |
| User account type | **Standard** |
| **Assignment** | Dynamic international **user** group |
| **Device group** | `GRP-CUSTOMER-PREFIX_Autopilot_v2_deployment_profile_Devices_INT` (Assigned; initially empty) |
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

The optional conversion profile `CUSTOMER-PREFIX_Autopilot_v1_conversion_deployment_profile_INT` is assigned to **GRP-CUSTOMER-PREFIX_Autopilot_v2_deployment_profile_Devices_INT**, with **Convert all targeted devices to Autopilot = Yes**.

A v2-only device is managed in Intune and Entra ID but does **not** automatically appear in the classic Autopilot device inventory. Eligible **corporate-owned** devices can be registered in classic Autopilot using the conversion setting; allow up to **48 hours**.

The default international profile targets **All devices**, but keep its conversion setting **No**. The separately assigned conversion profile is specifically targeted to v2-enrolled devices and must be tested for correct precedence. **This workflow requires no international Group Tag or dynamic device group.**

## 6. Consistent customer-prefix serial-based naming

### Classic v1 naming

Both DK and international classic Autopilot profiles use the same **customer-specific naming template**, substituting a short actual customer prefix:

```text
CUSTOMER-PREFIX-%SERIAL%
```

**Template example:** `CUSTOMER_PREFIX-PC-%SERIAL%` is a placeholder. Replace `CUSTOMER_PREFIX` with a short customer code; the resulting device name must fit the 15-character limit.

### Unassociated v2 fallback naming

The built-in v2 naming template applies during OOBE only when device association is used. For the **unassociated fallback** described here, use an Intune **post-enrollment Platform script** and assign it to **GRP-CUSTOMER-PREFIX_Autopilot_v2_deployment_profile_Devices_INT**.

```powershell
# Predefined variables for naming standard for CUSTOMER_NAME
$CUSTOMER_NAME = "CUSTOMER_PREFIX-PC"

# Get the device serial number
$Serial = (Get-CimInstance Win32_BIOS).SerialNumber.Trim()

# Change name to CUSTOMER_PREFIX-PC Serial number
$NewName = "$CUSTOMER_NAME-$Serial"
$NewName = $NewName.Substring(0, [Math]::Min(15, $NewName.Length))

if ($env:COMPUTERNAME -ne $NewName) {
    Rename-Computer -NewName $NewName -Force
}
```

**Block explanation:** The first block defines the customer-specific prefix (`CUSTOMER_PREFIX-PC` is a placeholder to replace). The second obtains the BIOS serial. The third forms the desired name, limits its length to 15 characters and renames only if necessary.

Run the script in **SYSTEM** context (**Run using logged-on credentials = No**) and **64-bit PowerShell**. Deploy it after enrollment, not as an OOBE-required device preparation script.

There is **no forced restart**. The rename becomes fully active after the next normal reboot; `Rename-Computer -Force` does not restart the computer.

**Uniqueness:** Long serial numbers may be truncated. This risks naming collisions; verify that the configured prefix/serial combinations remain unique for supported hardware.

## 7. Operational verification and monitoring

### After v1 registration

1. Device appears in **Windows Autopilot devices**.
2. Group Tag is `DK` for Danish; INT devices need no Group Tag.
3. DK devices are in the DK dynamic group.
4. Correct `CUSTOMER-PREFIX_Autopilot_v1_deployment_profile_DK` or `CUSTOMER-PREFIX_Autopilot_v1_deployment_profile_Default` profile is assigned.
5. Complete OOBE, verify apps/security controls.

### After v2 enrollment

1. Device shows in **Intune Windows devices** and **Entra ID devices**.
2. Device is a member of `GRP-CUSTOMER-PREFIX_Autopilot_v2_deployment_profile_Devices_INT`.
3. Deployment reports show the v2 provisioning results.
4. Post-enrollment script stages `<CUSTOMER-PREFIX>-<serial>` rename; check after next normal reboot.
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
| v2 device does not receive the conversion profile | Verify membership in the v2 assigned device group and confirm the intended profile takes priority over All devices |
| Serial rename does not display yet | Restart needed for name to take full effect; verify script result and check-in |
| `Install-Script` says script execution disabled | Process-scoped execution-policy bypass (if not centrally enforced) |
| Partner Center registration returns `ZtdDeviceNotFound` | Vendor/supplier registration prerequisites; fall back to classic hardware hash or v2 |

## 9. Production checklist

- [ ] DK profile assigned to the DK dynamic group; international default assigned to **All devices**. Both use `CUSTOMER-PREFIX-%SERIAL%` with a valid short customer prefix.
- [ ] Classic profile precedence tested: DK group wins over All devices; v2 conversion targeting checked separately.
- [ ] Dynamic INT user rule has been validated using representative users.
- [ ] v2 device group is Assigned, initially empty, and service principal is an owner.
- [ ] v2 policy targets **only** international users.
- [ ] Device enrollment restrictions allow intended fallback hardware.
- [ ] Required apps, baseline policies, and security controls are assigned correctly to v2 devices.
- [ ] Post-enrollment rename script validated with no forced restart.
- [ ] Corporate device ownership validated for conversion.
- [ ] `CUSTOMER-PREFIX_Autopilot_v1_conversion_deployment_profile_INT` profile piloted, including future reset.
- [ ] Report and document exceptions, failures and device-name collisions.

## References

- [Windows Autopilot device preparation overview](https://learn.microsoft.com/en-us/autopilot/device-preparation/overview)
- [Device preparation user-driven workflow](https://learn.microsoft.com/en-us/autopilot/device-preparation/tutorial/user-driven/entra-join-workflow)
- [Create a device preparation policy](https://learn.microsoft.com/en-us/autopilot/device-preparation/tutorial/user-driven/entra-join-autopilot-policy)
- [Device preparation troubleshooting FAQ](https://learn.microsoft.com/en-us/autopilot/device-preparation/troubleshooting-faq)
- [Classic Autopilot manual device registration](https://learn.microsoft.com/en-us/autopilot/add-devices)
- [Automatic registration of existing devices](https://learn.microsoft.com/en-us/autopilot/automatic-registration)
- [Dynamic group membership rules](https://learn.microsoft.com/en-us/entra/identity/users/groups-dynamic-membership)
