# Windows firewall prompts for local AI servers: what a rule covers, and how to read yours

*Development build, October 2026, for the UCompute part. The first part applies to any program on Windows that listens for other devices. UCompute is in development. There is no public release or download yet.*

When a local model server on Windows is meant to answer another computer, the Windows firewall decides whether that computer's traffic gets in. Most people meet this as a dialog and click through it. Behind the dialog is a firewall rule, and a rule has a small number of fields that decide what it lets in. Knowing those fields tells you what you agreed to, and the commands below show you the rules stored on your machine (add `-PolicyStore ActiveStore` to `Get-NetFirewallRule` to include rules that come from Group Policy). Run them in PowerShell; only `Set-NetConnectionProfile` and `Remove-NetFirewallRule` change anything, and they belong in an elevated PowerShell window.

## The fields that decide what a rule lets in

Microsoft's reference for `New-NetFirewallRule` describes them. Four matter most for a model server:

- **Direction.** "The acceptable values for this parameter are: Inbound or Outbound. The default value is Inbound." A server that answers other devices needs an inbound rule.
- **Program.** "Specifies the path and file name of the program for which the rule allows traffic." A rule tied to a program only lets traffic reach that program; a rule with no program condition is not tied to any one program, so it matches traffic to any program that fits its other conditions.
- **Profile.** "Specifies one or more profiles to which the rule is assigned. The rule is active on the local computer only when the specified profile is currently active." The values are Any, Domain, Private, Public or NotApplicable, and "The default value is Any."
- **Remote address.** Which devices the rule accepts, by address, subnet, range or a keyword such as `LocalSubnet`. If the address filter shows `Any`, the rule accepts any remote address.

Port and protocol are the other common conditions; a rule can combine them with a program condition, and a port rule without one applies to any program listening on that port.

## What "private" means to Windows

Which firewall profile is active follows the network category Windows assigned to your connection; the reference notes that "Only one profile is applied at a time." Microsoft's reference for `Set-NetConnectionProfile` defines the two you will see at home and on the road:

- "*Public* - Networks in a public place such as an airport or coffee shop. Your PC is hidden from other devices on the network and can't be used for printer and file sharing."
- "*Private* - Networks at home or work, where you know and trust the people and devices on the network. Your PC is discoverable and can be used for printer and file sharing if you set it up."

So a rule scoped to Private is active only while your connection is categorised as Private. On a network categorised as Public, that rule does nothing. The flip side: every network you mark Private gets the same rules, so "private" is a promise about the network you are on, not about which devices may connect.

See what your connection is categorised as:

```powershell
Get-NetConnectionProfile
```

The `NetworkCategory` line is the answer. Changing it uses `Set-NetConnectionProfile` (elevated window), for example `Set-NetConnectionProfile -InterfaceAlias "Wi-Fi" -NetworkCategory Private`; only do that for a network whose other devices you would let see your PC.

## Reading the rules on your machine

Firewall rules keep their conditions in separate "filter" objects. Microsoft's reference for `Get-NetFirewallRule` notes that "the common conditions like addresses or ports do not appear" in its output and that they "are represented in separate objects called filters". That is why the commands below pipe a rule into a filter command.

Enabled inbound allow rules:

```powershell
Get-NetFirewallRule -Direction Inbound -Action Allow -Enabled True |
  Format-List DisplayName, Profile, Direction, Action
```

All rules scoped to the Public profile (Microsoft's example; it includes disabled and outbound rules):

```powershell
Get-NetFirewallProfile -Name Public | Get-NetFirewallRule
```

Which program, which ports, which addresses a given rule covers (replace the display name with one from the first list):

```powershell
Get-NetFirewallRule -DisplayName "Example rule" | Get-NetFirewallApplicationFilter
Get-NetFirewallRule -DisplayName "Example rule" | Get-NetFirewallPortFilter
Get-NetFirewallRule -DisplayName "Example rule" | Get-NetFirewallAddressFilter
```

Every rule tied to one program (use the program's full path):

```powershell
Get-NetFirewallApplicationFilter -Program "C:\full\path\to\server.exe" | Get-NetFirewallRule
```

If nothing is returned, the rule may store the path with an environment variable such as `%SystemRoot%`; list all application filters and look for the program name instead.

Removing a rule you no longer want takes an elevated window and its display name. Display names accept wildcards and several rules can share one, so check what will be removed first:

```powershell
Remove-NetFirewallRule -DisplayName "Example rule" -WhatIf
Remove-NetFirewallRule -DisplayName "Example rule"
```

Be exact here. Microsoft's own first example for this command is `Remove-NetFirewallRule` with no arguments, and the example's description reads: "This example removes all of the static local firewall rules."

## Questions to ask of a rule for a model server

1. Is it an enabled inbound allow rule? If not, it is not what lets other devices in.
2. Does it name the server's program? If not, it lets traffic reach anything that listens on its ports.
3. Which profiles? Any (the default) includes the Public profile.
4. Does it limit remote addresses? If not, any device on a network with that profile can reach the program on the ports and protocol the rule allows, and what happens next depends on whether the server asks for a key.

A narrow rule is still only a network rule. It decides which traffic reaches the program, not who may send prompts; that is up to the server.

---

## What UCompute asks on Windows (development builds, October 2026)

*This section describes UCompute. UCompute is in development. There is no public release or download yet.*

UCompute Mesh lets you pair your own devices with a short code and choose which one runs a request. Each request runs whole on one device; UCompute does not split a model across devices. (Tested with development builds between one Apple M2 Mac and one Windows 11 laptop (Intel Arc 140V) on the same Wi-Fi network or phone hotspot, and between that Mac and one Linux desktop computer (Ubuntu, processor only, a development build of the background program compiled and started by hand; no app or installer) on a home Wi-Fi network. Linux is a planned platform, not a supported one. Pairing between two Macs is not yet shown to work.)

For a paired device to reach a Windows computer, the firewall has to let it in. On Windows, UCompute asks before it lets your paired devices reach it through the firewall, and the rule it adds covers only its own program, on private networks. That was shown in October 2026 on one Windows 11 laptop with a development build (later builds not yet re-checked); Windows asks for administrator approval.

In the terms above, the rule UCompute added in that test named only UCompute's own program and applied only on private networks (shown in October 2026 on one Windows 11 laptop with a development build; later builds not yet re-checked).

Your prompts and responses stay on your device unless you pair another device and choose to run a request there. Local-only mode refuses every off-device request. Downloading a model or the engine uses the internet. The device that runs your request processes your prompt. Pair only devices you control or trust.

No platform is declared supported yet. macOS 26.2 on Apple M2 and Windows 11 on an Intel Core Ultra 7 258V with Arc 140V are experimental; Linux, other GPUs and phones (as clients) are planned. UCompute's code is licensed under Apache-2.0. The source has not been published yet.

---

*Published by the UCompute project. This note was drafted and checked by UCompute's automated growth workstream against the project's engineering evidence. Corrections and questions: hello@ucompute.org. Please send anything security-related there rather than posting it publicly. UCompute is in development. There is no public release or download yet.*
