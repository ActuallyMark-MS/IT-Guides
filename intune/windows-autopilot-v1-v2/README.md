# Windows Autopilot v1 + v2: Danish and International Deployment

**Last updated:** 2026-11-23  
**Purpose:** Reusable Intune implementation guide for a hybrid Windows Autopilot setup, with a default international v1 profile and DK-specific override.  
**Status:** Reusable implementation design; validate outstanding items in a pilot before rollout. **Replace `CUSTOMER-PREFIX` with an actual short customer code.** See [Naming Standards](../../standards/NAMING-STANDARDS.md).

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

### Step 2 — Create the DK device group and optional common classic group

In [Entra admin center](https://entra.microsoft.com):

1. Go to **Identity → Groups → All groups → New group**.
2. Set **Group type = Security**.
3. Enter **Group name = DG_Intune_Autopilot_v1_Devices_DK**.
4. Set **Membership type = Dynamic Device**.
5. Open **Add dynamic query → Edit** (rule syntax editor) and enter:

~~~text
(device.devicePhysicalIds -any (_ -eq "[OrderID]:DK"))
~~~

6. Save the rule and create the group.
7. **Do not create an INT dynamic device group just for deployment.** The international v1 profile is the default, targeted to **All devices**. An older `DG_Intune_Autopilot_v1_Devices_INT` group can be retained only if another device-specific policy still needs it.
8. Optionally retain the shared **DG_Intune_Autopilot_v1_Devices_All** dynamic device group with:

~~~text
(device.devicePhysicalIds -any (_ -startsWith "[ZTDid]"))
~~~

9. Do **not** assign a competing classic deployment profile to the shared group. Assign common compliance/security profiles to suitable existing device groups instead.

**How membership works:** In classic Autopilot, Group Tag **DK** maps to `[OrderID]:DK` and adds that device to the DK dynamic group. The default INT profile does **not** require an INT Group Tag. This is separate from Entra **Device categories** and from an Intune **Scope tag**.

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
| Assignment | **All devices** (default); do not require Group Tag INT |

8. Keep the other deployment settings aligned with the DK profile unless your customer requires differences.
9. **DK devices are deliberately targeted by both profiles:** the DK profile via the specific dynamic group and the international default via All devices. The specifically assigned DK profile should win; confirm the actual assigned profile on a pilot DK computer before OOBE. Avoid any *additional* overlapping specifically targeted classic profiles.

**Important:** Windows computer names can be up to 15 characters long. Test serial lengths for the actual Lenovo models. Native v1 naming and post-enrollment v2 truncation may not behave identically for unusually long serials.

### Step 4 — Register a device for classic v1

**Supplier/OEM method (preferred for preregistration):**
1. Ask the supplier to register the device for the correct tenant. Request **Group Tag DK** only for Danish computers; international computers use the default profile without a tag.
2. In **Intune → Devices → Windows → Enrollment → Devices** (under Windows Autopilot), check that the serial appears.
3. For a **Danish** device, select the device, edit **Group tag** and enter **DK**. Leave the group tag unset for the default international profile.
4. Wait for dynamic membership and the appropriate deployment profile to reach **Assigned**.

**Manual hardware hash method when supplier cannot register it:**
1. Start a new Windows PC into OOBE, connect it to the internet, and press **Shift + F10**.
2. Start **powershell** in the Command Prompt.
3. Insert a USB drive. Confirm its drive letter in File Explorer or with **Get-Volume**; **D:** is used below only if it is your USB.
4. Run:

~~~powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass -Force
Install-Script -Name Get-WindowsAutoPilotInfo -Force
Get-WindowsAutoPilotInfo.ps1 -OutputFile "D:\Autopilot.csv"
~~~

5. On an admin PC, open **Intune → Devices → Windows → Enrollment → Devices → Import** under Windows Autopilot.
6. Upload **Autopilot.csv**, wait for import, and use **Sync/Refresh** if necessary.
7. For **DK** hardware, set **Group Tag = DK** and wait for the DK dynamic group and DK profile to apply. For **INT** hardware, leave the group tag blank; confirm the **Default** classic profile is assigned.
8. Only then hand the computer to the end user for v1 OOBE. If already partway through OOBE, return to a clean OOBE state as needed.

**Do not use -Partner** in the import command for direct Intune registration. In OOBE, a PowerShell execution-policy bypass may be necessary; it does not override centrally enforced policy.

### Step 5 — Create the dynamic international user group for v2

In [Entra admin center](https://entra.microsoft.com):

1. Navigate to **Identity → Groups → All groups → New group**.
2. Choose **Security**, name **GRP-CUSTOMER-PREFIX_Autopilot_v2_deployment_profile_INT-Users** (or reuse the international user group already created).
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

1. Enter **Group name = GRP-CUSTOMER-PREFIX_Autopilot_v2_deployment_profile_INT-Devices**.
2. Set **Group type = Security** and **Membership type = Assigned**.
3. Set **Microsoft Entra roles can be assigned to the group = No**.
4. On **Owners**, choose **Add owners** and search **Intune Provisioning Client**.
5. Verify **Application ID = f1346770-5b25-470b-88bd-d5744ab7952c**. It might show as **Intune Autopilot ConfidentialClient** instead.
6. Select it as the **Owner** and create/save the group.
7. Leave **Members empty**. v2 populates this group during device preparation.

**If the v2 policy refuses to save the group:** verify the group is an **Assigned** security group, the provisioning service principal is an **Owner**, and the admin has the Intune RBAC **Enrollment time device membership assignment** permission.

Reference: [Microsoft — Create assigned device group](https://learn.microsoft.com/en-us/autopilot/device-preparation/tutorial/user-driven/entra-join-device-group).

### Step 7 — Assign apps and prepare the v2 device preparation policy

**Required app targeting:** First assign critical apps as **Required** to **GRP-CUSTOMER-PREFIX_Autopilot_v2_deployment_profile_INT-Devices** in **Intune → Apps → All apps → [App] → Properties → Assignments**. Choose system-context installation where applicable. Keep the preparation-stage list short.

Now create the v2 policy:

1. Open **Intune → Devices → Windows → Enrollment → Device preparation policies**.
2. Select **Create → User Driven**.
3. **Basics:** enter **Name = CUSTOMER-PREFIX_Autopilot_v2_deployment_profile_INT**, then **Next**.
4. **Device group:** choose **GRP-CUSTOMER-PREFIX_Autopilot_v2_deployment_profile_INT-Devices** (this is the *device* group, not the users group).
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
10. **Assignments:** select **GRP-CUSTOMER-PREFIX_Autopilot_v2_deployment_profile_INT-Users** (**user** group, not devices group).
11. Select **Review + create → Save**. Confirm the policy shows in **Device preparation policies**.

**Selection logic:** An INT user with a classic-registered, assigned v1 PC receives its v1 profile. An eligible INT user on a PC without classic registration can receive v2 when they sign in. **This is not a fallback for a v1 deployment that has already failed.**

### Step 8 — Deploy post-enrollment naming for unassociated v2 devices

V1 supports its own naming template. Standard unassociated v2 does not apply the name template during OOBE; rename the computer after its Intune enrollment.

1. Save this as **Rename-Customer-Serial.ps1** on your admin computer, and replace the placeholder with a **short, real customer prefix**:

~~~powershell
$Serial = (Get-CimInstance Win32_BIOS).SerialNumber.Trim()
$CustomerPrefix = "ABC" # Replace with actual short customer prefix
$NewName = "$CustomerPrefix-$Serial"

# Windows computer names must be <=15 characters.
$NewName = $NewName.Substring(0, [Math]::Min(15, $NewName.Length))

if ($env:COMPUTERNAME -ne $NewName) {
    Rename-Computer -NewName $NewName -Force
}
~~~

2. Open **Intune → Devices → Scripts and remediations → Platform scripts → Add → Windows 10 and later**.
3. Name it **CUSTOMER-PREFIX_Autopilot_v2_deployment_profile_INT-Rename-Serial** and upload **Rename-Customer-Serial.ps1**.
4. Set **Run this script using the logged-on credentials = No**.
5. Set **Run script in 64-bit PowerShell host = Yes**.
6. On **Assignments**, target **GRP-CUSTOMER-PREFIX_Autopilot_v2_deployment_profile_INT-Devices**. Review/create.
7. Verify execution under the script's **Device status** or the device's script reporting after enrollment.

**No forced restart:** The script intentionally does not call Restart-Computer. The computer will use its new name after its next normal reboot. The -Force on Rename-Computer suppresses prompts; it does not reboot. The complete name must be at most 15 characters. The example truncates overlong names, which can create collisions; adopt a collision-safe scheme for models with long serials.

### Step 9 — Optionally register v2 machines in classic Autopilot automatically

This section implements the requested **v2 first deployment → classic v1 registration for later resets** workflow. It was a **design recommendation**, not a completed end-to-end tested deployment.

1. Open **Intune → Devices → Windows → Enrollment → Deployment profiles** under **Windows Autopilot**.
2. Select **Create profile → Windows PC**.
3. Enter **Name = CUSTOMER-PREFIX_Autopilot_v1_conversion_deployment_profile_INT**.
4. On **Basics**, set **Convert all targeted devices to Autopilot = Yes**.
5. Set **Deployment mode = User-driven**, **Join type = Microsoft Entra joined**, **User account type = Standard**.
6. Set **Language (Region) = Operating system default**, **Automatically configure keyboard = No** and **Apply device name template = Yes → CUSTOMER-PREFIX-%SERIAL%**.
7. Configure remaining OOBE options to match **CUSTOMER-PREFIX_Autopilot_v1_deployment_profile_Default**.
8. On **Assignments**, include **GRP-CUSTOMER-PREFIX_Autopilot_v2_deployment_profile_INT-Devices** only; don't overlap classic-registered **DG_Intune_Autopilot_v1_Devices_INT**.
9. Create the profile. After a v2 enrollment, check **Intune → Devices → Windows → Windows devices → [Device]** and verify **Ownership = Corporate**.
10. Eligible corporate-owned Intune devices can be registered in classic Autopilot automatically. Allow up to **48 hours** and check **Windows → Enrollment → Windows Autopilot → Devices** for the device and profile assignment.
11. Test a **subsequent reset** of a pilot device to verify the classic profile takes over correctly.

**Do not assume automatic Group Tag assignment.** Conversion does **not** add **Group Tag INT**, so the converted device won't automatically enter **DG_Intune_Autopilot_v1_Devices_INT**. It can use the dedicated conversion profile via **GRP-CUSTOMER-PREFIX_Autopilot_v2_deployment_profile_INT-Devices**. Applying the Group Tag automatically would require an extra Microsoft Graph workflow that is **not part of this guide**.

Also, a device classified as **Personal** won't be registered via this conversion setting. Confirm ownership/identification in the pilot.

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

| Group | Group type | Usage |
| --- | --- | --- |
| `DG_Intune_Autopilot_v1_Devices_DK` | Dynamic device | Classic DK devices |
| `DG_Intune_Autopilot_v1_Devices_INT` | Optional legacy dynamic device | Not required for the All devices default INT deployment |
| `DG_Intune_Autopilot_v1_Devices_All` | Optional dynamic device | Shared classic Autopilot registered devices; **not needed** for assigning the default profile |
| `GRP-CUSTOMER-PREFIX_Autopilot_v2_deployment_profile_INT-Users` (example) | Dynamic user | Assignment target for v2 policy |
| `GRP-CUSTOMER-PREFIX_Autopilot_v2_deployment_profile_INT-Devices` | **Assigned** security group | Empty initially; v2 adds enrolled devices |

### Classic device dynamic rules

DK:

```text
(device.devicePhysicalIds -any (_ -eq "[OrderID]:DK"))
```

INT (optional legacy device group only, not required for deployment):

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
| `CUSTOMER-PREFIX_Autopilot_v1_deployment_profile_DK` | `DG_Intune_Autopilot_v1_Devices_DK` | Danish (Denmark) | Yes | `CUSTOMER-PREFIX-%SERIAL%` |
| `CUSTOMER-PREFIX_Autopilot_v1_deployment_profile_Default` | **All devices** | Operating system default/user-selected | No | `CUSTOMER-PREFIX-%SERIAL%` |

These are user-driven Microsoft Entra joined deployments.

**Intended precedence:** the DK profile is specifically assigned to `DG_Intune_Autopilot_v1_Devices_DK`, and the international default is assigned to **All devices**. The DK assignment takes precedence. Avoid additional overlapping specific profiles except the carefully tested v2 conversion profile.

### Classic manual hardware-hash registration

During Windows OOBE, open Command Prompt with `Shift + F10`, then start PowerShell. With writable USB at `D:\`:

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass -Force
Install-Script -Name Get-WindowsAutoPilotInfo -Force
Get-WindowsAutoPilotInfo.ps1 -OutputFile "D:\Autopilot.csv"
```

Verify that `D:\` is really the USB drive; it is not guaranteed on every PC. Script installation requires internet access. The execution-policy bypass only applies to the current process and does not override an enforced enterprise policy.

**Do not specify `-Partner`** for standard Intune classic Autopilot import. The `-Partner` export is intended for Partner Center.

In Intune: **Devices → Windows → Enrollment → Windows Autopilot devices → Import**, upload CSV and refresh. Set Group Tag **DK only for Danish devices**; international computers receive the **All devices** default profile without a tag. Verify the actual assigned profile.

### Partner Center lesson learned

Partner Center registration using manufacturer/model/serial can fail unless prerequisites for the hardware/vendor identity are satisfied. We saw `ZtdDeviceNotFound` when testing such registration for a Lenovo device. Where the supplier has not provided/registered the needed identity, use manual hardware-hash import to classic Autopilot—or the v2 international fallback where appropriate.

## 4. Autopilot v2 configuration: international fallback

Intune → **Devices → Windows → Enrollment → Device preparation policies** → Create `CUSTOMER-PREFIX_Autopilot_v2_deployment_profile_INT`.

| Configuration | Selection |
| --- | --- |
| Join type / mode | **User-driven Microsoft Entra join** |
| User account type | **Standard** |
| **Assignment** | Dynamic international **user** group |
| **Device group** | `GRP-CUSTOMER-PREFIX_Autopilot_v2_deployment_profile_INT-Devices` (Assigned; initially empty) |
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

The main international v1 profile is assigned to **All devices**, so it doesn't depend on Group Tag `INT`. To keep automatic conversion scoped specifically to v2 fallback devices, use the separate conversion profile assigned to the v2 device group and verify actual classic profile precedence in the pilot.

### Recommended conversion approach

1. Keep `CUSTOMER-PREFIX_Autopilot_v1_deployment_profile_Default` assigned to **All devices** for originally registered INT devices. No `INT` Group Tag is needed.
2. Create a dedicated **classic** profile named `CUSTOMER-PREFIX_Autopilot_v1_conversion_deployment_profile_INT`, matching the international OOBE and `CUSTOMER-PREFIX-%SERIAL%` naming configuration.
3. Set **Convert all targeted devices to Autopilot = Yes**.
4. Assign this classic conversion profile to `GRP-CUSTOMER-PREFIX_Autopilot_v2_deployment_profile_INT-Devices`.
5. Verify the v2-enrolled devices are classified **corporate-owned**. If not, correct the enrollment identification/ownership route before relying on automatic registration.
6. Allow conversion and verify each device appears under **Windows Autopilot devices**, with appropriate classic profile assignment.

**Important limitations:**
- Conversion is **registration**, not a fresh OOBE run, and it does not immediately replace the running Windows installation.
- **No Group Tag `INT` is automatically assigned.** The separately targeted conversion profile can be used for subsequent classic deployment, provided membership and precedence are validated against the All devices default.
- If an extra `INT`-tag device group is required for unrelated policy assignments, it needs separate tag automation. It is **not required** for the default international v1 deployment. This was discussed but **not implemented**.
- Pilot this configuration before production. Verify that targeting and policy precedence remain correct after reset and throughout group membership changes.

## 6. Consistent customer-prefix serial-based naming

### Classic v1

Use the **same template** for DK and international classic v1 profiles after replacing the placeholder with a short customer code:

~~~text
CUSTOMER-PREFIX-%SERIAL%
~~~

The placeholder `CUSTOMER-PREFIX` must be replaced before implementation; it is **not** intended literally.

### Unassociated v2 fallback

The v2 device naming template requires **device association** to rename within OOBE. For devices using v2 *without* preregistration/association, leave the v2 template disabled and rename after enrollment.

Intune platform script (PowerShell), targeted to **`GRP-CUSTOMER-PREFIX_Autopilot_v2_deployment_profile_INT-Devices`**:

```powershell
$Serial = (Get-CimInstance Win32_BIOS).SerialNumber.Trim()
$CustomerPrefix = "ABC" # Replace with actual short customer prefix
$NewName = "$CustomerPrefix-$Serial"

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

**Serial-number caveat:** Truncation may cause duplicate names if long serial numbers share a leading sequence. Test across hardware models and choose a collision-resistant strategy if needed. PowerShell interpolates `"$CustomerPrefix-$Serial"`; single quotes would not expand the variables.

## 7. Operational verification and monitoring

### After v1 registration

1. Device appears in **Windows Autopilot devices**.
2. Group Tag is `DK` for Danish; INT devices need no Group Tag.
3. DK devices are in the DK dynamic group.
4. Correct `CUSTOMER-PREFIX_Autopilot_v1_deployment_profile_DK` or `CUSTOMER-PREFIX_Autopilot_v1_deployment_profile_Default` profile is assigned.
5. Complete OOBE, verify apps/security controls.

### After v2 enrollment

1. Device shows in **Intune Windows devices** and **Entra ID devices**.
2. Device is a member of `GRP-CUSTOMER-PREFIX_Autopilot_v2_deployment_profile_INT-Devices`.
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
| v2 device is missing from `DG_Intune_Autopilot_v1_Devices_INT` | No Group Tag is assigned automatically on v2-to-v1 conversion |
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
