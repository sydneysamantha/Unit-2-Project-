# Securing Servers Using Honeypots and IP Blocking

**Scope:** This document walks through configuring a honeypot, simulating connection attempts, monitoring the logs, and creating a firewall rule to block the attacker.

A honey pot is a decoy system made to look like a real vulnerability, set up to lure attackers into interacting with it.

## Executive Summary 
This walkthrough covers configuring and testing SSH access, logging, and firewall rules on a Windows Server. Different methods of logging will result in different logon types, which is useful for identifying how a user or attacker is accessing a system. The firewall rule initially did not work as expected. Through multiple tests and troubleshooting, it was revealed that this was the result of the firewall being disabled by default settings on all profiles (Domain, Private, and Public), even though the rule itself was configured correctly.After enabling the profiles, the rule was retested and confirmed to work.

**Prerequisites**
- Windows machine (Server or Windows 10/11) with admin access
- Know your Windows machine's IP address (`ipconfig`) and username (`whoami`)
- A second machine to simulate the attacker (Kali was used here)

---

## Step 1: Configure an OpenSSH server on a Windows VM

On the Windows virtual machine, navigate to `Settings` > `Apps`, click `Optional features`, then `Add a feature`, and add `OpenSSH Server`.

<img width="1516" height="917" alt="image" src="https://github.com/user-attachments/assets/1f8e1060-1b40-4d66-9708-0b96dd7d2467" />

OpenSSH is a program that allows users remote access into a server and encrypts all data between the client and server. It will act as the honeypot in this project.

Run the following commands in PowerShell to start OpenSSH:

```
Start-Service sshd
Set-Service -Name sshd -StartupType Automatic
```

`Start-Service sshd` starts the service, and `Set-Service -Name sshd -StartupType Automatic` sets it to launch automatically on every boot, so you won't need to manually start it after a restart.

<img width="570" height="106" alt="image" src="https://github.com/user-attachments/assets/877017cd-2f24-4a4d-8428-600b4270c9e5" />

---

## Step 2: Connect from another machine

Simulate an attacker connecting to the honeypot from the Kali VM, which serves as the attacking machine in this exercise. Using the Windows machine's IP address and username, connect from Kali using the command below.

```
ssh labuser01@10.0.0.7
```

If this is the first time connecting to this machine, you may be prompted with:

<img width="647" height="173" alt="image" src="https://github.com/user-attachments/assets/950ab1a2-5c07-4193-804a-ea22acd827e1" />

This is just asking you to verify the server.

Once you enter the password, you're connected.

<img width="432" height="162" alt="image" src="https://github.com/user-attachments/assets/41682704-f22e-4d0d-910d-5aafa74d1dd8" />

---

## Step 3: Configure logging

Return to the Windows Server to confirm that OpenSSH is running.

### Verify OpenSSH is running

Press Windows button + R to open the Run dialog, and type `services.msc`.

<img width="788" height="354" alt="image" src="https://github.com/user-attachments/assets/7581affe-7a75-4545-a135-7042a62d03ed" />

The Services window will appear, allowing you to search for OpenSSH Server and confirm it's running.

<img width="1270" height="683" alt="Screenshot 2026-07-11 at 2 57 47 PM" src="https://github.com/user-attachments/assets/d9aac2dc-4f4d-4c36-bb65-9c9ab7e13c59" />


Next, verify that logging is enabled. In PowerShell, run the command:

```
New-ItemProperty -Path "HKLM:\SOFTWARE\OpenSSH" -Name "DefaultShell" -Value "C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe" -PropertyType String -Force
```

<img width="962" height="192" alt="Screenshot 2026-07-11 at 3 04 53 PM" src="https://github.com/user-attachments/assets/fff5caa9-ab24-450d-bd2b-58bf31419991" />

### Reviewing SSH logs in Event Viewer

Press Windows button + R and type `eventvwr.msc`. Once Event Viewer opens, click `Applications and Services Logs`, then `Open SSH`, then `Operational`.

<img width="1199" height="594" alt="image" src="https://github.com/user-attachments/assets/c9321208-015d-4ab3-9501-e4a579cf58ed" />

Look specifically for these two event types:

| Event ID | Meaning |
|---|---|
| 4 | Login attempt |
| 10 | Session opened |

At this point, only Event ID 4 appears, even though the login from Kali was successful and confirmed to be dropping into labuser01@10.0.0.7.

<img width="663" height="362" alt="Screenshot 2026-07-11 at 6 01 51 PM" src="https://github.com/user-attachments/assets/54b72fa3-5217-43a9-8eb6-fc67bc0a9e81" />


### Troubleshooting missing Event ID 10

If Event ID 10 isn't appearing despite a successful login, the log level may be set too low to capture it. Follow these steps to check whether the log level on OpenSSH is set to INFO or VERBOSE.

1. In PowerShell, type `notepad C:\ProgramData\ssh\sshd_config` to open the OpenSSH configuration file.

   <img width="974" height="507" alt="image" src="https://github.com/user-attachments/assets/93751288-5e13-42b2-bc0b-6b11d47c68e7" />

2. Look for the line called `#LogLevel`.

   <img width="732" height="839" alt="Screenshot 2026-07-11 at 3 36 52 PM" src="https://github.com/user-attachments/assets/b6542694-050d-40ce-a955-c6d84d346e6c" />

   If it's set to INFO, change it to VERBOSE.

   <img width="194" height="91" alt="image" src="https://github.com/user-attachments/assets/61159437-bf12-4ec4-93be-40501e1efa7f" />

3. Restart OpenSSH by running the command `Restart-Service sshd` in PowerShell.

   <img width="726" height="188" alt="image" src="https://github.com/user-attachments/assets/82150d3b-8b64-4b9e-932a-bbb8d8468cea" />

4. Attempt to remote in again from Windows Server 2.

   <img width="386" height="184" alt="image" src="https://github.com/user-attachments/assets/32712e76-c019-4837-bc98-653b1c91e249" />

   Check the log to see if it captured an Event ID 10.

   <img width="662" height="475" alt="image" src="https://github.com/user-attachments/assets/39057939-c657-4060-b3c3-962f6301b1e3" />

If there are still no Event ID 10 entries logged, review the details of one of the Event ID 4 entries.

<img width="727" height="275" alt="image" src="https://github.com/user-attachments/assets/ccdb7211-59b5-48a2-bb80-1e00de082975" />

This may indicate that the login attempt and session opening are being combined under Event ID 4 once the password is accepted.

### Cross-checking Windows Security logs

Next, check the Windows Security logs by clicking `Windows Logs` then `Security`. Use the filter feature to help sort through the logs and look for these events:

<img width="1337" height="646" alt="Screenshot 2026-07-11 at 4 43 52 PM" src="https://github.com/user-attachments/assets/cbfe651c-f578-4bd5-aef2-eb71699fedc7" />


| Event ID / Field | Meaning |
|---|---|
| 4624 | Successful log in |
| 4625 | Failed log in |
| LogonType 10 | Remote logins (RDP/Terminal Services) |


<img width="992" height="485" alt="image" src="https://github.com/user-attachments/assets/cc7f66a3-e5fe-4c60-92e8-38695eae3a36" />

<img width="958" height="681" alt="image" src="https://github.com/user-attachments/assets/123d8e66-907d-4327-b233-4286b08df591" />

When logging in with a password over SSH, you won't see LogonType 10, since that's reserved for RDP (Remote Display Protocol) or Terminal Services, which are GUI-based. SSH connections through PowerShell will instead show Logon Type 8.

### Exporting the logs

Right-click Security and select `Save All Events As...` to export the logs as a .evtx or .txt file for later review.

<img width="345" height="401" alt="Screenshot 2026-07-11 at 4 47 53 PM" src="https://github.com/user-attachments/assets/d10f4232-9ee7-469c-905f-13775c87cfd1" />

<img width="755" height="471" alt="image" src="https://github.com/user-attachments/assets/21ab4602-0ee0-4775-852b-f971ebe8c5a7" />

---

## Step 4: Block the Kali IP via a firewall rule

On the Windows Server, open Windows Defender Firewall with Advanced Security. This can be done one of two ways:

- Type "Windows Defender Firewall with Advanced Security" into the taskbar search box.
- Press Windows button + R, then type wf.msc and press Enter.

<img width="1046" height="783" alt="image" src="https://github.com/user-attachments/assets/6392a698-2cc0-4774-b811-c9d95a7b1bdb" />

Once there, create a new Inbound Rule:
1. Click Inbound Rules.
2. Click New Rule...

The New Inbound Rule Wizard window will appear.

<img width="1046" height="784" alt="image" src="https://github.com/user-attachments/assets/205e6f2b-af87-4716-9644-a7e6605e785c" />

3. Select Port, then click Next.
4. Make sure TCP is selected and type 22 into Specific local ports, then click Next.

<img width="712" height="579" alt="image" src="https://github.com/user-attachments/assets/fa8e9f6e-30b3-43fc-ab63-115991b330ff" />

5. Select Block the Connection, then click Next.
6. Make sure Domain, Private, and Public are all selected, then click Next.

<img width="537" height="476" alt="image" src="https://github.com/user-attachments/assets/afa14801-61cf-4e75-a9e5-a438e1db8a1b" />

7. Name the rule "Block SSH from Specific IP" and click Next.

<img width="497" height="421" alt="image" src="https://github.com/user-attachments/assets/20ef8ff7-d01f-4d46-a570-8d095aeaa687" />

Once complete, the rule will appear at the top of the Inbound Rules list.

<img width="1014" height="366" alt="image" src="https://github.com/user-attachments/assets/04fffabc-d93f-4c05-b530-26958a329f97" />

Now apply the rule to the IP address you're SSHing in from.

1. Right-click the rule and select Properties.

<img width="538" height="175" alt="image" src="https://github.com/user-attachments/assets/5654c7a9-2cb5-40cc-99d7-db9f7c9338bd" />

2. Navigate to the Scope tab, and under Remote IP address, select These IP Addresses.

<img width="431" height="547" alt="image" src="https://github.com/user-attachments/assets/830aa8e5-a38c-40de-a796-ae0a45981a80" />

3. Type in the IP address to block, then click OK.

<img width="334" height="363" alt="image" src="https://github.com/user-attachments/assets/af27813c-8012-4090-9c80-aa9684c97f1c" />

4. Click OK to apply.

<img width="428" height="574" alt="image" src="https://github.com/user-attachments/assets/b14624d5-80d1-474f-8307-c08e6e5b9739" />

To verify the rule, try connecting from the blocked IP address over SSH.

If the rule fails and you're still able to log on over SSH, double-check these things:

1. **Is the rule enabled?**
   - Right-click on the rule and click Properties.

2. **Is the rule correct?**
   - Block connection (should be selected on the General tab)
   - TCP port 22 (listed under Ports and Protocols)
   - Domain, Public, and Private all selected (listed under Advanced)

3. **Is the IP address correct?** (Listed under Scope)
   - Remote IP address selected
   - These IP addresses selected

If everything appears correct, try these commands in PowerShell to troubleshoot.

```
Get-NetFirewallRule -DisplayName "*SSH*" | Select DisplayName, Enabled, Direction, Action
```
This lists all the SSH rules currently running.

<img width="922" height="168" alt="Screenshot 2026-07-11 at 6 02 47 PM" src="https://github.com/user-attachments/assets/debd5ff1-480c-43aa-aa40-81d08f3757bd" />

```
Get-NetFirewallProfile | Select Name, Enabled
```
This checks whether the firewall profiles are enabled.

<img width="603" height="142" alt="Screenshot 2026-07-11 at 6 03 15 PM" src="https://github.com/user-attachments/assets/0c919646-4c30-4ae6-852d-bea4325b1356" />

```
Get-NetConnectionProfile
```
This checks which profile (Domain, Public, Private) the machine is using.

<img width="478" height="168" alt="image" src="https://github.com/user-attachments/assets/5a6d64de-78ff-41ee-90f9-c84d433d2072" />

If the second command shows that Domain, Public, and Private are all disabled, that's the reason the rule isn't working.

To enable all profiles, run this command:

```
Set-NetFirewallProfile -Profile Domain,Private,Public -Enabled True
```

Re-test by logging in via SSH from Kali.

```
ssh labuser01@10.0.0.7
```

<img width="666" height="346" alt="image" src="https://github.com/user-attachments/assets/2b0da61c-046a-4155-8345-eaa7d2d4293a" />

No response confirms that the firewall is now working.

---

## References

The following sources were used to research and compile this report. Steps were tested and adapted based on results in this environment.

1. **How to install OpenSSH on Windows** — Reddit post
2. **Logging and Viewing Remote SSH Connections on Windows (OpenSSH Server)** — Forum post
3. **How to configure Windows Firewall rules** — Vendor/official documentation
4. **How to take screenshots** — Reference guide






























