# Active Directory Home Lab — Domain Design & Permission Segregation

A self-built Windows Server Active Directory environment demonstrating domain
promotion, organizational unit design, and permission management using
security groups — built and documented as part of my hands-on cybersecurity
learning path.

## Problem

A growing company can't manage user accounts machine-by-machine. With even a
modest headcount, creating and maintaining separate local accounts on every
PC becomes unmanageable — password resets, onboarding, offboarding, and
security policy all have to happen individually, per machine, per person.
It's also nearly impossible to investigate a security incident across
dozens of disconnected local logs with no central record.

Active Directory solves this by centralizing identity and policy: one
Domain Controller holds every user and computer account, and every machine
on the network authenticates against it rather than trusting its own local
account list.

This lab builds a small, realistic AD environment for a fictional company
("Acme Ltd" / `labcorp.local`) with three departments, and solves a common
real-world permission problem: granting a specific individual cross-cutting
access without breaking their departmental organization.

## Architecture

```
labcorp.local (domain)
│
├── OU: IT
├── OU: Finance
│   ├── User: Sarah Jones (sjones)
│   └── User: Raj Patel (rpatel)  ← Finance department, IT specialist role
└── OU: Sales

Security Group: Software-Install-Allowed (domain root, not tied to any OU)
└── Member: Raj Patel (rpatel)
```

**Domain Controller:** `DC_01.labcorp.local` — Windows Server 2025 Standard
(Server Core), running AD DS + DNS, static IP `192.168.1.50`, running on
VirtualBox with a bridged network adapter on my home LAN.

## Skills Demonstrated

- Windows Server installation and configuration (Server Core, no GUI —
  managed entirely via SConfig and PowerShell)
- Static IP/DNS configuration and subnetting (`/24` addressing)
- Active Directory Domain Services (AD DS) installation and domain promotion
- Organizational Unit (OU) design
- User account creation and management via PowerShell
- Security Group creation and membership management
- VM networking troubleshooting (NAT vs. Bridged, promiscuous mode)
- Systematic debugging: isolating failures by splitting compound commands
  into individual steps

## The Core Design Decision: OUs vs. Groups

A common early mistake is treating OUs and Security Groups as
interchangeable — they solve different problems:

| | Organizational Unit (OU) | Security Group |
|---|---|---|
| **Answers** | Where does this account organizationally live? | What permissions does this account need? |
| **Membership** | One OU per account | Can belong to many groups at once |
| **Used for** | Applying baseline policy (Group Policy) to a department | Granting specific, often cross-cutting, access |

**Worked example — Raj Patel:** Raj is an IT specialist embedded in the
Finance team. His account correctly lives in `OU=Finance` — that's his real
department, and where department-wide policy should apply to him. But he
also needs software-install permission, normally reserved for IT.

Instead of moving him into the IT OU (which would incorrectly apply IT's
department-wide policies to him, and misrepresent his actual team), he was
added individually to the `Software-Install-Allowed` security group — a
group that sits at the domain root, independent of any OU, and can contain
members from any department. His organizational home stays accurate, and
he gets exactly the one extra permission he needs.

```powershell
# Raj's account: correctly placed by department
New-ADUser -Name "Raj Patel" -SamAccountName "rpatel" `
  -Path "OU=Finance,DC=labcorp,DC=local"

# Permission handled separately, via group membership
New-ADGroup -Name "Software-Install-Allowed" -GroupScope Global `
  -Path "DC=labcorp,DC=local"

Add-ADGroupMember -Identity "Software-Install-Allowed" -Members "rpatel"
```

## Build Log: Problems Encountered & How I Solved Them

Documenting the real troubleshooting, not just the commands that worked —
this is where most of the actual learning happened.

**1. VM couldn't reach the network — nmap scans hung indefinitely**
Root cause: the VM's network adapter was set to VirtualBox's "NAT Network"
mode (an isolated virtual network that coincidentally showed an IP in the
same range as my home network) with Promiscuous Mode set to Deny, silently
blocking the low-level packet visibility scanning tools need. Fixed by
switching to a Bridged Adapter (attached to the real physical WiFi adapter,
identified via `ipconfig /all`) with Promiscuous Mode set to Allow All.

**2. Accidentally cleared the Windows Security event log**
While exploring Event Viewer, I cleared the log instead of just clearing a
filter. Rather than a disaster, this turned into a useful lesson: Windows
immediately logs the clearing action itself (Event ID 1102, "The audit log
was cleared"), including the account and exact timestamp responsible — a
built-in tamper-evidence mechanism specifically so an attacker can't
silently erase their tracks. This is also the practical reason centralized
logging (SIEM) matters: a local log, even with this protection, can still
be cleared by anyone with local admin rights — shipping copies off-machine
in real time is what makes that unrecoverable.

**3. `New-ADUser` with an inline password failed with "server unwilling to
process the request"**
Combining account creation and password-setting into a single command
failed inconsistently. Rather than guessing at the cause, I isolated it by
splitting the operation into three separate steps: create the account with
no password (succeeded), set the password with `Set-ADAccountPassword`
(succeeded), then enable the account with `Enable-ADAccount` (succeeded).
This confirmed the failure was specific to the combined command on this
server build, and the split-and-isolate approach — rather than repeatedly
retrying the same failing command — is the actual transferable lesson.

**4. Repeated PowerShell syntax errors from manual retyping**
VirtualBox clipboard sharing wasn't enabled by default, so early commands
were typed manually and picked up small but breaking typos (`D=local`
instead of `DC=local`, `-AsPlaiText` instead of `-AsPlainText`). PowerShell's
error messages turned out to be precise enough to pinpoint the exact broken
parameter each time, which made these fast to diagnose once I learned to
read the error text carefully rather than just re-typing blind.

**5. Learning to trust silent success**
Several AD cmdlets (`New-ADOrganizationalUnit`, `Set-ADAccountPassword`,
`Enable-ADAccount`) return no output at all when they succeed — only
failures print anything. Early on this looked like commands "doing
nothing," until verifying with a corresponding `Get-` cmdlet each time
confirmed the action had actually completed. Now a standard habit: after
any state-changing command, verify with a read command rather than
assuming success or failure from the absence of output.

## What I'd Build Next

- Group Policy Objects (GPOs) linked to OUs — enforcing the software-install
  restriction designed above, not just modeling it
- Join a client machine (Kali or a Windows client) to the domain and
  observe authentication traffic and Event Viewer logs from the client
  side
- Extend the OU structure with nested OUs per department (e.g.
  `OU=Finance/Managers`)
