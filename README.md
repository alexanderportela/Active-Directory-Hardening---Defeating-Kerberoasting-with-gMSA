📖 Overview

This project demonstrates the deployment of a Group Managed Service Account (gMSA) within a custom Windows Server 2022 Active Directory environment. The objective was to permanently mitigate a previously executed Kerberoasting vulnerability against an MSSQL service account. By converting the standard service account to a gMSA, Active Directory now manages the credential, utilizing an automatically rotating 120-character password that makes offline dictionary and brute-force attacks mathematically impossible.

🏗️ Lab Architecture

    Domain Controller: Windows Server 2022 (DC-01)

    Member Server: Windows Server 2022 File/SQL Server (FILE-01)

    Service Target: MSSQL Service Principle Name (MSSQLSvc/FILE-01.lab.local:1433)

    Administrative Access: Executed via Tier 0 Domain Admin credentials conforming to strict AD tiering boundaries.

⚙️ Deployment Lifecycle
1. Key Distribution Services (KDS) Initialization

To allow Active Directory to generate secure cryptographic passwords, a KDS Root Key was deployed to the domain forest.

    Command: Add-KdsRootKey -EffectiveTime ((Get-Date).AddHours(-10))

    Note: Backdating the effective time allowed for immediate lab deployment, bypassing the standard 10-hour replication wait period.

2. Access Control & Account Generation

Active Directory requires strict authorization determining which hosts are allowed to retrieve the managed password.

    Created a Global Security Group (gMSA-SQL-Servers) and added the target member server (FILE-01$).

    Stripped the vulnerable SPN from the legacy service account using setspn -D.

    Generated the gMSA and bound the SPN directly to the new secure identity:
    New-ADServiceAccount -Name "gmsa_sql" -PrincipalsAllowedToRetrieveManagedPassword "gMSA-SQL-Servers" -ServicePrincipalNames "MSSQLSvc/FILE-01.lab.local:1433"

3. Server Installation & Cryptographic Validation

The target server was configured to securely query Active Directory for the password blob via the Netlogon secure channel.

    Installed the RSAT Active Directory PowerShell module on the member server to facilitate AD communication.

    Executed Install-ADServiceAccount to locally stage the account.

    Validated the secure communication channel using Test-ADServiceAccount, returning a True boolean.

    Verified the backend cryptographic attributes (msDS-ManagedPasswordInterval) to confirm the automated 30-day rotation schedule was active.

4. Service Binding

    Configured the target Windows Service to run as LAB\gmsa_sql$.

    Left all password GUI fields entirely blank, allowing the Windows Local Security Authority (LSA) to seamlessly inject the 120-character managed password into memory upon service startup.

🛡️ Strategic Impact (Blue Team Perspective)

Kerberoasting exploits fundamental Active Directory Ticket Granting Service (TGS) behavior rather than a software flaw, making it difficult to patch. The ultimate defense is removing the human element from password creation. By implementing a gMSA, the password complexity far exceeds the computational limits of modern cracking rigs (like Hashcat), effectively neutralizing the threat of offline ticket cracking while completely eliminating administrative password management overhead.
