# Naming Standard

**Purpose:** A reusable naming convention for groups, configuration profiles, policies, scripts and computers across customers and IT projects.

This is a naming reference, **not an implementation guide**. Service-specific guides define which objects to create and how to configure them.

## 1. General principles

- Use descriptive, predictable names that explain **what an object is for**.
- Separate name components with underscores (`_`); use a hyphen (`-`) within a customer prefix or where the platform requires it.
- Keep the same spelling, capitalization and segment order for objects of the same type.
- Add detail only when it distinguishes an object, such as `DK`, `INT`, `Default`, `Production` or `Pilot`.
- Omit unused optional segments rather than writing empty underscores.
- Keep names short enough for the target platform's restrictions.
- In reusable documentation, write `CUSTOMER-PREFIX` as a placeholder. Replace it with the actual customer's abbreviation when implementing a guide.

## 2. Entra ID group names

**Format:**

```text
<GroupType>_<PrimaryFunction>_<Subfunction>_<SubSubfunction>_<Devices|Users>_<AdditionalInfo>
```

| Segment | Meaning | Examples |
| --- | --- | --- |
| GroupType | Membership convention | `DG`, `SG` |
| PrimaryFunction | Service or management domain | `Intune`, `Entra`, `M365` |
| Subfunction | Main objective | `Compliance`, `Security`, `Applications` |
| SubSubfunction | More specific purpose, optional | `Windows`, `Baseline`, `Office` |
| Devices or Users | Membership target | `Devices`, `Users` |
| AdditionalInfo | Scope or distinction, optional | `DK`, `INT`, `Pilot`, `All` |

**Group prefixes:**
- **`DG`** — dynamic membership group (user or device membership rules).
- **`SG`** — assigned-membership security group.

Both are Microsoft Entra security groups; the prefixes are **organizational conventions**, not separate Entra group types.

**Generic examples:**

```text
GRP-CUSTOMER-PREFIX_Autopilot_v2_deployment_profile_Devices_INT
DG_Intune_Compliance_Windows_Devices_DK
DG_Entra_Identity_Users_INT
SG_Intune_Applications_Office_Devices_Pilot
SG_M365_Licensing_Users_Standard
```

**Order requirement:** `Devices` or `Users` is always the **second-to-last segment**, immediately before `AdditionalInfo` (such as `DK`, `INT` or `Pilot`). The middle segments can be adjusted for the actual service and function; the order remains consistent.

## 3. Policies, profiles and configuration objects

**Format:**

```text
CUSTOMER-PREFIX_<PrimaryFunction>_<Subfunction>_<SubSubfunction>_<AdditionalInfo>
```

**Generic examples:**

```text
CUSTOMER-PREFIX_Intune_Compliance_Windows_Default
CUSTOMER-PREFIX_Intune_Security_Baseline_Production
CUSTOMER-PREFIX_Entra_ConditionalAccess_MFA_AllUsers
CUSTOMER-PREFIX_Intune_Applications_Office_Pilot
```

- Replace `CUSTOMER-PREFIX` with the customer's chosen abbreviation.
- The function names describe **what** the policy/configuration does, not only where it is stored.
- Include versions only where a version is meaningful for deployment and maintenance.
- Do not rename existing tenant objects solely because a documentation template changes without first assessing dependencies.

## 4. Computer names

**General template:**

```text
CUSTOMER-PREFIX-%SERIAL%
```

- `CUSTOMER-PREFIX` is a documentation placeholder and must be replaced by the customer's **short** device prefix.
- `%SERIAL%` means the hardware serial number **where the chosen provisioning system supports that token**. Other systems may use different variables.
- Example: customer prefix `ABC`, serial `PF61TZZW` → `ABC-PF61TZZW`.
- For compatibility with Windows naming and legacy systems, keep the resulting computer name within **15 characters**, including separators.
- Ensure names remain unique. When a prefix plus serial exceeds the length limit, define a safe shortening strategy; do not silently truncate names if that could create duplicates.

For PowerShell-based naming, variable expansion uses `$Serial`, **not** `%SERIAL%`:

```powershell
$CustomerPrefix = "ABC"  # Example: replace with actual short prefix
$Serial = (Get-CimInstance Win32_BIOS).SerialNumber.Trim()
$NewName = "$CustomerPrefix-$Serial"
```

## 5. Applying this standard in future guides

1. Reference this file early in each guide.
2. Use the prescribed naming format consistently in headings, steps, configuration tables and scripts.
3. Clearly distinguish **placeholders** from real values to be entered in a tenant.
4. Describe exceptions and platform-specific limits inside the relevant implementation guide.
5. Separate the **names of groups or policies** from independent metadata such as tags, categories, filters and assignments.
6. Do not assume a rename in a GitHub guide changes existing objects in Microsoft 365, Intune or other platforms.

> This standard is deliberately service-agnostic so it can be reused in future guides without adopting terminology from one deployment scenario.
