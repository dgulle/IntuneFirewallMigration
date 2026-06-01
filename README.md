---
title: Intune Firewall Migration
os: Windows
microsoftPlatform: Intune
language: PowerShell
category: Provisioning
runContext: Local
elevation: Administrator
readOnly: false
tags: [firewall, settings-catalog, group-policy, migration, microsoft-graph]
---

# Intune Firewall Migration

`IntuneFirewallMigration.ps1` captures the Windows Defender Firewall rules applied to the local machine (via Group Policy and, optionally, locally defined rules) and creates equivalent Settings Catalog firewall rule profiles in Microsoft Intune.

It is an updated, streamlined version of the retired [Microsoft Endpoint Manager Firewall Rule Migration Tool](https://learn.microsoft.com/en-us/mem/intune/protect/endpoint-security-firewall-rule-tool), which was removed in June 2024. This version uses Settings Catalog as the native target, supports filtering by direction and profile, and removes the legacy tooling and telemetry from the original.

This solution has been recognised as part of the [MEM Official Community Tools](https://www.memcommunity.com/official-community-tool-oct).

> [!NOTE]
> This script was authored by [Nick Benton](https://github.com/ennnbeee) of [odds+endpoints](https://www.oddsandendpoints.co.uk/) and is vendored into this repo from the upstream [ennnbeee/IntuneFirewallMigration](https://github.com/ennnbeee/IntuneFirewallMigration) project. See [LICENSE](LICENSE) for the original MIT license.

---

## What the Script Does

1. Reads Windows Defender Firewall rules applied to the local machine from Group Policy (and optionally locally defined rules).
2. Filters the rule set based on the parameters supplied — direction (`inbound` / `outbound`), profile (`domain`, `private`, `public`, `all`, `notconfigured`), and enabled / disabled state.
3. Deduplicates rule names and converts each rule into the Settings Catalog representation expected by the Intune Graph API.
4. Connects to Microsoft Graph using either an interactive sign-in or an Entra ID app registration (tenant + client + secret).
5. Creates one or more Settings Catalog firewall rule profiles in Intune, named with the supplied prefix and split into chunks of `splitRules` rules each (default `100`).
6. Optionally creates legacy **Endpoint Security** firewall profiles instead of Settings Catalog profiles when `-legacyProfile` is supplied.

---

## Key Differences From the Original Microsoft Tool

- Creates **Settings Catalog** firewall rule policies natively.
- Allows selection of only specific firewall profiles (`Domain`, `Private`, `Public`).
- Supports importing only **inbound** or **outbound** rules.
- Removes the dependency on the old Microsoft GitHub repository.
- Uses the supported `Microsoft.Graph.Authentication` module and `Invoke-MgGraphRequest`.
- Disables and removes all telemetry functions and calls.
- Fixes profile-name matching when no existing firewall rule policies exist.
- Resolves compatibility issues with `Microsoft.Graph` 2.26.1 on PowerShell 5.

![Firewall Migration Tool](img/mstool.png)

---

## Requirements

### PowerShell

- Windows PowerShell 5.1 or PowerShell 7, running **on Windows**.

### Modules

The script auto-detects and installs the following modules if they are missing:

- `Microsoft.Graph.Authentication`
- `ImportExcel`

### Permissions

- Local administrator on the source Windows device (required to read Group Policy firewall rules).
- Intune permissions to create configuration profiles via Microsoft Graph:
  - `DeviceManagementConfiguration.ReadWrite.All`

Either:

- An **Entra ID app registration** with the Graph application permission above, **or**
- An interactive sign-in with an account that holds the equivalent delegated permission (typically Intune Administrator).

### Source Device

The script reads firewall rules from the **local machine**. Run it on a representative Windows device that already has the Group Policy firewall rules you want to migrate applied to it.

---

## Usage

Clone or download this repository to the Windows machine where the firewall rules are applied, then run the script from inside the `Windows-IntuneFirewallMigration` folder.

### Authenticating With an App Registration

```powershell
$tenantId  = '<your-tenant-id>'
$appId     = '<your-app-id>'
$appSecret = '<your-app-secret>'

.\IntuneFirewallMigration.ps1 -profileName TestMigration -tenantId $tenantId -appId $appId -appSecret $appSecret
```

If `-tenantId`, `-appId`, and `-appSecret` are omitted, the script falls back to an interactive Graph sign-in.

### Test Run

Creates Settings Catalog firewall rule profiles prefixed `TestMigration` using only the first **20 enabled Group Policy** rules:

```powershell
.\IntuneFirewallMigration.ps1 -profileName TestMigration -mode Test
```

### General Usage

Creates Settings Catalog firewall rule profiles prefixed `FirewallRules`, **100** rules per profile, from all enabled Group Policy rules:

```powershell
.\IntuneFirewallMigration.ps1 -profileName FirewallRules
```

### Inbound Rules Only

```powershell
.\IntuneFirewallMigration.ps1 -profileName InboundFirewallRules -ruleDirection inbound
```

### Outbound Rules Only

```powershell
.\IntuneFirewallMigration.ps1 -profileName OutboundFirewallRules -ruleDirection outbound
```

### Domain Profile Rules Only

```powershell
.\IntuneFirewallMigration.ps1 -profileName DomainFirewallRules -firewallProfile domain
```

### Private Profile Rules Only

```powershell
.\IntuneFirewallMigration.ps1 -profileName PrivateFirewallRules -firewallProfile private
```

### Public Profile Rules Only

```powershell
.\IntuneFirewallMigration.ps1 -profileName PublicFirewallRules -firewallProfile public
```

### Include Locally Defined Rules

Includes rules defined locally on the device in addition to Group Policy rules, with 70 rules per profile:

```powershell
.\IntuneFirewallMigration.ps1 -profileName LocalFirewallRules -includeLocalRules -splitRules 70
```

### Include Disabled Rules

Includes disabled rules alongside enabled ones, with 50 rules per profile:

```powershell
.\IntuneFirewallMigration.ps1 -profileName DisabledFirewallRules -includeDisabledRules -splitRules 50
```

### Legacy Endpoint Security Profiles

Creates the legacy Endpoint Security firewall rule profiles instead of Settings Catalog profiles:

```powershell
.\IntuneFirewallMigration.ps1 -profileName LegacyProfileFirewallRules -legacyProfile
```

> [!IMPORTANT]
> Legacy Endpoint Security profiles do not appear in the Intune portal immediately — they are processed and converted in the background.

---

## Parameter Reference

| Parameter | Description |
|---|---|
| `-profileName` | Name prefix used for the created Intune profiles. |
| `-mode` | `Test` to limit to the first 20 rules; omit for a full run. |
| `-ruleDirection` | `inbound` or `outbound`. Omit to include both. |
| `-firewallProfile` | `domain`, `private`, `public`, `all`, or `notconfigured`. Omit to include all profiles. |
| `-includeLocalRules` | Include rules defined locally on the device in addition to GPO rules. |
| `-includeDisabledRules` | Include disabled rules in addition to enabled rules. |
| `-splitRules` | Maximum rules per Settings Catalog profile (default `100`). |
| `-legacyProfile` | Create legacy Endpoint Security firewall profiles instead of Settings Catalog. |
| `-tenantId`, `-appId`, `-appSecret` | Entra ID app registration credentials. All three are required for non-interactive authentication. |

---

## Notes

- This script **creates** Intune configuration profiles in the target tenant. It does not assign them to any group — review and assign the resulting profiles manually before relying on them.
- Run the script on a device that already has the source GPO firewall rules applied, otherwise there will be nothing to migrate.
- The upstream project is currently in **Public Preview**. Issues and contributions should be raised against the [upstream repository](https://github.com/ennnbeee/IntuneFirewallMigration/issues).

---

## Version History

- **v0.4.2** — Better error handling; improved support for German-language rules.
- **v0.4.1** — Resolved issues with rules containing local and remote address ranges.
- **v0.4.0** — Inbound/outbound filtering; non-English rule descriptions; updated Graph scopes; reordered rule filtering for performance.
- **v0.3.1** — Resolved an issue with missing file paths on rules.
- **v0.3.0** — Per-profile rule selection; duplicate rule names suffixed `(1)`, `(2)`; improved Settings Catalog conversion.
- **v0.2.1** — Only unique rules created in Settings Catalog policies; better duplicate handling.
- **v0.2.0** — Settings Catalog as the default; `-legacyProfile` switch for Endpoint Security policies.
- **v0.1.0** — Initial release.

---

## License

MIT — see [LICENSE](LICENSE). Original work © [Nick Benton](https://github.com/ennnbeee).
