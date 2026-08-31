# HotS-Block — Build & Recovery Runbook

Everything needed to modify or remove the App Control policy on this PC.
Written 2026-08-30.

---

## 1. What is deployed

| | |
|---|---|
| Policy name | `HotS-Block` |
| PolicyID | `{7B3E9A21-4C8D-4F6B-9E2A-1D5C8F3B7A44}` |
| Version | `1.0.0.0` |
| Type | Base policy, **signed**, **enforced** |
| Location | `\EFI\Microsoft\Boot\CIPolicies\Active\{7B3E9A21-...}.cip` |
| Secure Boot | On |

**What it does**

- **Denies** Heroes of the Storm (two Blizzard/DigiCert signing CAs, scoped by `FileAttrib`)
- **Allows** everything else (`Allow FileName="*"`)
- **Allows** AppControl Manager explicitly by package name

**Signing certificate**

| | |
|---|---|
| Subject | `CN=HotS-Block-Signer` |
| Thumbprint | `43C51BCFB1F672721DA62C325A9AF28175FE9930` |
| Algorithm | sha512RSA, 4096-bit |
| Expires | 2076-08-30 |
| TBS SHA-512 | `73954BB4814656F020737AFB0B49DCD83089F83C1C624242354809A6F2E3A10B97E7582F25309CF151B49579CE805DD18E1FAD56466431C7BC16144CDF623832` |

The policy names this certificate as its `UpdatePolicySigner`. **Nothing can modify or
remove this policy without its private key.** That is intentional.

---

## 2. Read before touching anything

**Never delete the `.cip` from the EFI partition.**
This is a signed policy under Secure Boot. Deleting the file does not remove the policy —
it is the documented route to an unbootable machine. Removal must go through a signed
update, every time.

**The policy has no `Enabled:Unsigned System Integrity Policy` option.**
This is deliberate. It is what makes the policy binding, and it is why removal takes two
stages with a reboot between them.

**If the key is lost**, the only realistic exits are disabling Secure Boot in firmware
(uncertain) or reinstalling Windows.

---

## 3. What you need

| Item | Where |
|---|---|
| `HotS-Block-Signer.pfx` | **the other device** |
| PFX password | **paper, stored separately** |
| Policy XML | this repo |
| GitHub Actions | this repo — does the XML to `.cip` conversion |
| `signtool.exe` | `C:\Program Files (x86)\Windows Kits\10\bin\10.0.19041.0\x64\` |
| `CiTool.exe` | `C:\Windows\System32\` (built in) |

**Windows 11 Home has no `ConfigCI`**, so `ConvertFrom-CIPolicy` does not exist locally.
That single step is what GitHub Actions exists for. Everything else runs on this PC.

---

## 4. The build pipeline

```
edit XML  ->  push to GitHub  ->  Actions runs ConvertFrom-CIPolicy  ->  download .cip
          ->  signtool (local)  ->  CiTool (local, elevated)  ->  reboot
```

**No secrets ever go to GitHub.** The XML contains only rules and the certificate's
*public* fingerprint. Signing happens locally.

### 4.1 Import the key

```powershell
$pw = Read-Host "PFX password" -AsSecureString
$c = Import-PfxCertificate -FilePath '<path>\HotS-Block-Signer.pfx' -CertStoreLocation Cert:\CurrentUser\My -Password $pw
$c | Select-Object Subject, Thumbprint, HasPrivateKey
```

Thumbprint must be `43C51BCF...`. Must run in a real interactive console.

### 4.2 Convert

Commit the XML to this repo and push. The workflow converts **every** `.xml` in the repo
root and publishes an `unsigned-cip` artifact. Download it from the Actions run and unzip.

Verify the run's first step passed — it checks `ConvertFrom-CIPolicy` exists on the runner.

> If Actions refuses with a budget error, the repo must be **public**. Public repos get
> unlimited free Actions minutes; private ones consume a monthly quota.

### 4.3 Sign

```powershell
& "C:\Program Files (x86)\Windows Kits\10\bin\10.0.19041.0\x64\signtool.exe" sign -v -n "HotS-Block-Signer" -p7 . -p7co 1.3.6.1.4.1.311.79.1 -fd sha256 "<path>\HotS-Block.cip"
```

`-p7co 1.3.6.1.4.1.311.79.1` is the OID that marks the blob as a Code Integrity policy.
Output is `<name>.cip.p7` — **rename it to `.cip`**, replacing the unsigned file.

Verify the signer before deploying:

```powershell
$raw = [IO.File]::ReadAllBytes('<path>\HotS-Block.cip')
$cms = New-Object System.Security.Cryptography.Pkcs.SignedCms
$cms.Decode($raw)
$cms.SignerInfos | ForEach-Object { $_.Certificate.Subject; $_.Certificate.Thumbprint }
```

### 4.4 Deploy — elevated

```powershell
CiTool --update-policy "<path>\HotS-Block.cip"
```

Expect `Operation Successful`. **If it errors, do not reboot** — the old policy is still
in place and safe. Then reboot.

---

## 5. Removing the policy

Two stages, **with a reboot between them**. Skipping the reboot risks a half-applied state.

### Stage 1 — deploy a permissive, removable version

Copy `HotS-Block.xml` and make exactly these changes:

1. **Bump the version** — `<VersionEx>1.0.0.1</VersionEx>` (must be higher; CI rejects same-or-lower)
2. **Keep `PolicyID` and `BasePolicyID` identical** — this replaces the policy rather than adding a second one
3. **Add** to `<Rules>`:
   ```xml
   <Rule>
     <Option>Enabled:Unsigned System Integrity Policy</Option>
   </Rule>
   ```
4. **Delete** the `<DeniedSigners>` block from the User Mode signing scenario
5. **Delete** the two Blizzard `<Signer>` entries, the two HotS `<FileAttrib>` rules, and the `<CiSigner>` references to them
6. **Keep `<UpdatePolicySigners>` and `<SupplementalPolicySigners>` intact** — losing these means losing control of the policy

Then run the pipeline: convert, sign, `CiTool --update-policy`, **reboot**.

Verify:

```powershell
CiTool --list-policies --json | ConvertFrom-Json | Select-Object -ExpandProperty Policies | Where-Object { -not $_.IsSystemPolicy } | Format-List FriendlyName,VersionString,PolicyOptions
```

`VersionString` must read `1.0.0.1` and `PolicyOptions` must include
`Enabled:Unsigned System Integrity Policy`. **Do not continue until you see both.**

### Stage 2 — remove it

Elevated:

```powershell
CiTool --remove-policy "{7B3E9A21-4C8D-4F6B-9E2A-1D5C8F3B7A44}"
```

Reboot. Confirm nothing non-Microsoft remains:

```powershell
CiTool --list-policies --json | ConvertFrom-Json | Select-Object -ExpandProperty Policies | Where-Object { -not $_.IsSystemPolicy }
```

Empty output means the policy is gone.

---

## 6. Troubleshooting

### `Access denied` when signing

`CryptographicException` from `TrySignHash`. The key container's permissions are broken —
not the key itself. **Fix:** delete the certificate from `Cert:\CurrentUser\My` and
re-import the PFX. A fresh import creates a working container.

Test signing capability before trusting it:

```powershell
$c = Get-Item Cert:\CurrentUser\My\43C51BCFB1F672721DA62C325A9AF28175FE9930
$rsa = [System.Security.Cryptography.X509Certificates.RSACertificateExtensions]::GetRSAPrivateKey($c)
$rsa.SignData([Text.Encoding]::UTF8.GetBytes('probe'), [Security.Cryptography.HashAlgorithmName]::SHA512, [Security.Cryptography.RSASignaturePadding]::Pkcs1).Length
```

512 means it works.

### AppControl Manager: "Fast Cache data not found"

`AppControlManager.exe` has no signature of its own; it relies on its MSIX package record.
A Microsoft Store update re-registers the package, and under an enforced policy the record
can fail to rebuild. Logged as CodeIntegrity event **3033**, launch error `0x80070241`.

**This is not a lockout.** AppControl Manager is not required for recovery — the pipeline
in section 4 does everything without it. Reinstall the app, or ignore it.

To prevent it: Microsoft Store, profile icon, Settings, **App updates: Off**.

### Checking what is actually blocked

```powershell
Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-CodeIntegrity/Operational'; Id=3033,3077; StartTime=(Get-Date).AddDays(-7)} | Select-Object TimeCreated,Id,Message
```

- **3077** — enforcement block
- **3033** — failed signing-level check
- **3089** — signature detail for a correlated event (noise on its own)

---

## 7. Useful facts

- AppControl Manager package family: `VioletHansen.AppControlManager_ea7andspwdn10`
- Inspect the EFI partition: `mountvol S: /s` ... `mountvol S: /d` (elevated)
- A `.cip` starts with `08 00 00 00`, then PolicyID and PlatformID as little-endian GUIDs — a quick way to confirm you have the right file
- Microsoft's own policies live in `C:\Windows\System32\CodeIntegrity\CiPolicies\Active` — leave them alone
- `winsipolicy.p7b` on the EFI partition is Microsoft's, unrelated to this policy
