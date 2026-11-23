# Naming Standards for IT Guides

**Purpose:** Use these naming conventions consistently in future implementation guides and customer environments.

## 1. Placeholder

Use **`CUSTOMER-PREFIX`** as the reusable placeholder for a customer's abbreviation.

- Replace it with a real customer-specific prefix when implementing a guide.
- Do **not** use a customer's actual name in a generic reference guide unless specifically requested.
- In this repository, use `CUSTOMER-PREFIX`, not `CUSTOMERNAME`.

## 2. Microsoft Entra ID group names

**Pattern:**

```text
<GroupType>_<PrimaryFunction>_<Subfunction>_<SubSubfunction>_<Devices|Users>_<AdditionalInfo>
```

- `DG`: Dynamic group (dynamic **user** or dynamic **device** membership).
- `SG`: Assigned security group.
- Use an underscore (`_`) between name segments.
- `AdditionalInfo` identifies a region, target, purpose or other meaningful distinction, for example `DK`, `INT`, `All`.
- Omit non-applicable optional segments, rather than adding empty underscores.

**Examples:**

| Purpose | Example name | Membership |
| --- | --- | --- |
| Classic Autopilot Danish devices | `DG_Intune_Autopilot_v1_Devices_DK` | Dynamic device |
| Classic Autopilot registered devices, common | `DG_Intune_Autopilot_v1_Devices_All` | Dynamic device |
| Autopilot v2 international users | `DG_Intune_Autopilot_v2_Users_INT` | Dynamic user |
| Autopilot v2 international devices | `SG_Intune_Autopilot_v2_Devices_INT` | Assigned device security group |

**Important:** `SG` is a *naming convention* distinguishing assigned membership. Both `DG` and `SG` objects are Entra security groups.

## 3. Autopilot profile / Intune policy names

**Pattern:**

```text
CUSTOMER-PREFIX_<PrimaryFunction>_<Subfunction>_<SubSubfunction>_<AdditionalInfo>
```

Use `CUSTOMER-PREFIX` as a placeholder in guides. Keep the function and deployment type recognizable.

**Examples:**

| Purpose | Example name |
| --- | --- |
| Classic Autopilot Danish profile | `CUSTOMER-PREFIX_Autopilot_v1_deployment_profile_DK` |
| Classic Autopilot default international profile | `CUSTOMER-PREFIX_Autopilot_v1_deployment_profile_Default` |
| Classic Autopilot conversion profile for v2-originated devices | `CUSTOMER-PREFIX_Autopilot_v1_conversion_deployment_profile_INT` |
| Autopilot v2 international fallback | `CUSTOMER-PREFIX_Autopilot_v2_deployment_profile_INT` |

## 4. Windows computer names

**Reference pattern for both DK and international devices:**

```text
CUSTOMER-PREFIX-%SERIAL%
```

Replace `CUSTOMER-PREFIX` with the customer's actual, **short** computer-name prefix. `%SERIAL%` is the native classic Windows Autopilot variable, not a PowerShell variable.

**Example only:** With customer prefix `ABC` and serial `PF61TZZW`, the resulting computer name is `ABC-PF61TZZW`.

**For PowerShell post-enrollment naming (for unassociated v2):**

```powershell
$CustomerPrefix = "ABC" # Replace with the real short prefix
$Serial = (Get-CimInstance Win32_BIOS).SerialNumber.Trim()
$NewName = "$CustomerPrefix-$Serial"
```

**Windows computer-name constraint:** Stay within **15 characters** (including the separator and serial number) and ensure uniqueness. `CUSTOMER-PREFIX` itself is a **documentation placeholder**, not a literal prefix to deploy: it is too long to combine with typical serial numbers. If the combined name is too long, define and document a collision-safe naming strategy for that customer; do not silently truncate serial numbers unless the risk has been accepted.

## 5. Apply these rules in new guides

1. Reference this file from each implementation guide.
2. Use these group and profile naming patterns consistently in prose, tables, examples, and steps.
3. Use `CUSTOMER-PREFIX` as a placeholder, not a real company name.
4. Keep **Group Tag** values (such as `DK`) separate from **group names**.
5. For classic Autopilot where the international profile is **All devices (default)**, DK still uses its DK group; INT Group Tag is not required. A shared dynamic v1 group is optional for other policy targeting.
6. When modifying an existing tenant, distinguish **new names proposed in documentation** from existing deployed object names. Renaming a guide does not rename live objects.
