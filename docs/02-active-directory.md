# 02 – Active Directory and DNS

## Goal

Add identity and name resolution to the lab: a domain controller in Management and a domain-joined client in Endpoints.

AD DS was installed **after** network segmentation between Endpoints and Management had been verified. This meant fewer possible sources of error when troubleshooting.

## Domain controller

1. Changed the private IP of `vm-dc01` from dynamic to static in Azure (10.0.0.132). A DC that also acts as DNS server must not change address.
2. Installed Active Directory Domain Services, DNS Server, Group Policy Management and the AD DS admin tools.
3. Promoted the server to the first DC in a new forest.

| Setting | Value |
| --- | --- |
| Domain | `ad.lab` |
| NetBIOS name | `AD` |
| Forest / domain functional level | Windows Server 2025 |
| Global Catalog | Enabled |
| DNS Server | Enabled |

A separate DSRM password was created for recovery mode.

### Verification on the DC

```powershell
whoami                 # ad\labadmin
$env:USERDOMAIN        # AD
Get-Service DNS,NTDS   # both Running
nslookup ad.lab        # 10.0.0.132
```

## DNS for the VNet

The VNet DNS setting was changed from Azure-provided DNS to a custom DNS server, **10.0.0.132**.

Why: a client has to use the DC as DNS server to find the domain, the DC and the AD-related DNS records (Kerberos, LDAP). Azure-provided DNS doesn't know about the internal domain `ad.lab`.

Verification on `vm-win11-01` after `ipconfig /renew`:

```
ipconfig /all     → DNS Server: 10.0.0.132
nslookup ad.lab   → 10.0.0.132
```

## Domain join

`vm-win11-01` was joined to `ad.lab` and restarted.

```
systeminfo | findstr /B /C:"Domain"   → Domain: ad.lab
whoami (after domain logon)           → ad\labadmin
```

The identity chain now works:

```
vm-win11-01 → DNS via 10.0.0.132 → vm-dc01 → ad.lab → domain authentication
```

## Troubleshooting notes

- Logging in to `vm-win11-01` through Bastion with the local account failed. Resetting the password for the local account in the Azure portal solved it, so the problem was the account credentials and not Bastion or the network.
- The Bastion login field didn't accept `DOMAIN\username`. The UPN format (`labadmin@ad.lab`) worked.

## Security perspective

- Identity and directory services (Management) are separated from the client zone (Endpoints).
- The NSG rule that blocks RDP from Endpoints to Management stays in place. AD traffic between the zones is not yet restricted.
- Next step for the network: identify which ports domain traffic actually needs, and write more precise NSG rules based on that.

> **Lab vs. production:** The lab uses a single admin account (`labadmin`) for both domain administration and logons on the client. In production you would separate admin accounts from daily-use accounts and restrict where Domain Admins can log on (for example with a tiering model).
