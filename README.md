# Zero-Trust Firewall Monitor — Build 0.7.7

**A compact, single-file Windows Firewall control and inspection tool.**

Built around four practical priorities:

1. **Executable identity with SHA-256** — fingerprint applications, check Authenticode signatures and publishers, and detect changes to executables associated with managed allow rules.
2. **Explicit firewall decisions** — inspect newly blocked connections and decide whether to allow, block, or ignore the exact connection.
3. **Native packet capture** — capture network traffic with Windows `pktmon`, preserve the original ETL, and convert the capture to PCAPNG.
4. **Direct Wireshark control** — launch the Wireshark desktop application against the latest PCAPNG for interactive packet inspection.

The project aims to keep the workflow lean: one PowerShell script, Windows' built-in firewall and packet-monitoring tools, and Wireshark as an optional GUI analyzer. It does not require TShark.

---

## 1. Overview

Zero-Trust Firewall Monitor applies a strict Windows Firewall profile and monitors new Windows Filtering Platform block events. When a supported blocked connection is detected, the script displays the connection details and offers a set of explicit actions.

The monitor is designed to leave existing firewall rule objects alone. It does not rewrite, re-enable, disable, or delete existing rules. Before creating a new rule, it checks the effective and local persistent firewall stores for an equivalent rule. A match is skipped; a name collision or failed safety check prevents creation.

The script is intended for technically capable Windows users who want to see what is being blocked, make deliberate rule decisions, fingerprint executables, capture traffic, and inspect the resulting packets in Wireshark.

## 2. Main menu

| Key | Function | What it does |
|---|---|---|
| **1** | Start monitor | Applies and verifies strict firewall profile settings, then monitors new Security event 5157 records. |
| **2** | Firewall policy backup / restore | Opens the dedicated `rulle.wfw` save/restore submenu. |
| **E** | DNS / DoH audit | Read-only review of DNS servers on active adapters, available DoH mappings, and a normal Windows DNS resolution probe. |
| **R** | Review all firewall rules | Read-only display of all effective rules returned by Windows `ActiveStore`, including policy-source information and associated filters. |
| **M** | Managed-rule health | Reports expired managed rules and executable-integrity warnings. Firewall rules remain unchanged. Expired remembered-ignore entries may be cleaned from the separate decision store. |
| **I** | Application intelligence | Inspect one executable, review remembered ignore decisions, or export SHA-256 fingerprints for running processes. |
| **C** | Packet capture | Start/stop the integrated `pktmon` capture, convert ETL to PCAPNG, check capture ownership, and open the capture in Wireshark. |
| **Q** | Quit | Exit the monitor. |

## 3. Strict firewall mode

Choose **[1]** to start the monitor. The script saves a baseline when needed and applies these settings to the Domain, Private, and Public profiles:

- Firewall enabled.
- Default inbound action: **Block**.
- Default outbound action: **Block**.
- Local firewall rules allowed to merge: **True**.

It then verifies the profile settings. If an effective policy does not match the requested settings—for example, because Group Policy overrides a local setting—the script reports the problem rather than claiming strict mode was successfully applied.

**Important:** strict mode changes firewall profile settings. It does not automatically create allow rules for every application. Applications without an effective allow rule may lose network access. Save a known-good firewall policy before making changes.

## 4. Blocked-connection decisions

The monitor reads new **Windows Security event 5157** records. It does not replay older records from before the monitor's starting event cursor.

For a supported inbound or outbound event, the decision prompt offers:

| Choice | Action |
|---|---|
| **1** | Allow the application for the reported direction. |
| **2** | Allow the remote IP, and remote port when applicable. |
| **3** | Block the application for the reported direction. |
| **4** | Block the remote IP, and remote port when applicable. |
| **T** | Create an application ALLOW rule with an expiry reminder recorded in its description. |
| **V** | Create an application BLOCK rule with an expiry reminder recorded in its description. |
| **I** | Ignore this exact connection for the current monitor session. No firewall rule is created. Repeated matching events are suppressed until the monitor stops. |
| **J** | Remember an ignore decision for this exact connection. The default duration is seven days; the decision is tied to the executable's SHA-256 identity. |

Allowing an executable requires a readable executable path and a SHA-256 fingerprint. The script displays the Authenticode status and publisher information. An executable without a currently valid Authenticode signature requires the additional explicit `TRUST` confirmation before an allow rule can be created.

### Duplicate protection and existing-rule policy

Before creating a managed rule, the script checks both `ActiveStore` and `PersistentStore` for an equivalent rule using the application path, direction, action, profile, protocol, remote address, and remote port.

- An equivalent rule is skipped, including when the matching rule is disabled.
- Existing descriptions and settings are not rewritten.
- Existing rules are not re-enabled, disabled, or deleted.
- A stable-name collision blocks creation rather than overwriting the existing rule.
- If a store cannot be read safely for the duplicate check, rule creation is stopped.
- Only an explicit rule action can create a new rule. Choosing **[I]** or **[J]** does not create a firewall rule.

### Temporary-rule behavior

The **[T]** and **[V]** choices record an expiry time as metadata for later review. They do **not** create self-deleting Windows Firewall rules. When a temporary rule passes its recorded expiry, the maintenance report flags it but leaves the firewall rule unchanged. Review and remove such a rule manually using Windows Firewall management tools if it is no longer needed.

## 5. SHA-256 executable control

Executable identity is a central part of this monitor.

### Inspect one executable — `I` → `1`

Enter the full path to an executable to display:

- SHA-256 file hash.
- Authenticode signature status.
- Publisher and certificate thumbprint when available.
- Company, product, file description, and file version metadata.
- A path/signature-based classification, such as Windows directory, Program Files, user-writable location, or other/custom location.

The classification is a review aid, not a malware verdict or reputation score.

### Fingerprint every running process — `I` → `3`

The inventory scans running processes and records, where accessible:

- Process ID and process name.
- Executable path.
- SHA-256 hash.
- File length and last-write time.
- Scan time and a status explaining unavailable paths or hash failures.

Each unique executable path is hashed once per inventory run and reused for processes running from that same file. Protected processes, access-denied paths, or processes that exit during collection are reported with an unavailable/error status instead of being silently treated as verified.

Two reports are saved for each run: **CSV** for spreadsheet review and **JSON** for structured analysis.

### Managed allow-rule integrity checks

Managed allow-rule descriptions contain the executable SHA-256 recorded when the rule was created. The **[M]** health audit compares the current file hash with that stored hash and reports a warning if the file is missing, cannot be read, or has changed.

**A mismatch does not automatically disable or delete the rule.** The rule is retained unchanged so the user can investigate before taking action.

A SHA-256 match proves only that the file bytes match the recorded hash at the time of comparison. It does not prove that the executable is safe. Authenticode signature verification and publisher information provide additional context, not a guarantee of benign behavior.

## 6. Packet capture and Wireshark

Packet inspection is integrated into the **[C]** menu and uses Windows' built-in `pktmon.exe` for capture. Wireshark is used as the interactive GUI analyzer.

### Capture workflow

1. Select **[C]** from the main menu.
2. Choose **[1] Start capture**.
3. Use the computer normally while the capture is running.
4. Return to the capture menu and choose **[2] Stop owned capture + convert to PCAPNG**.
5. Open the newest PCAPNG in Wireshark with **[5] Open the latest PCAPNG in Wireshark**.

The capture menu also provides **[3] Show pktmon / ownership status**.

### Capture settings and safety

The script starts `pktmon` with these settings:

- Capture component: network adapters (NICs).
- Packet snap length: **512 bytes**.
- Capture file size limit: **128 MB**.
- Logging mode: **circular**.
- Original ETL is retained after conversion to PCAPNG.

The script uses a global mutex and a SHA-256-protected ownership marker. It checks that `pktmon` is stopped before starting a new capture. It will not take ownership of a capture already running outside this monitor. When stopping, it compares the active ETL path with the recorded owned path and refuses to stop a different or unverified capture.

These safeguards are intended to prevent the monitor from stopping another tool's packet capture by mistake.

### Launch Wireshark

Build 0.7.7 uses **`Wireshark.exe` only** for interactive packet review. It does not search for or launch `tshark.exe`, and it does not generate a separate automatic TShark text audit.

The Wireshark lookup checks the configured portable path first:

`C:\Software\WiresharkPortable64\App\Wireshark\Wireshark.exe`

It also checks the `$env:Software` portable location, the executable in `PATH`, and standard Program Files installation folders. Before launch, the script checks the executable's embedded Windows version metadata. Menu **[5]** opens the latest completed PCAPNG directly in the Wireshark GUI.

Wireshark is optional for firewall monitoring. It is required only for the integrated GUI launch workflow. The monitor does not install Wireshark or Npcap.

### What to inspect in Wireshark

Use Wireshark to inspect packet timestamps, endpoints, protocols, DNS exchanges, TCP handshakes, TLS handshakes, retransmissions, and other packet-level details. Display filters can help narrow the capture, for example:

```text
dns
tls
tcp
udp
ip.addr == 192.168.0.170
```

The packet capture and Security event 5157 log serve different purposes. Use timestamps, addresses, ports, and protocol information to correlate them; the PCAPNG does not automatically prove which Windows process generated every packet.

**Capture privacy:** PCAPNG and ETL files can contain sensitive network metadata and packet contents. Store and share them carefully.

## 7. DNS / DoH configuration audit

Choose **[E]** for a read-only configuration review. Depending on the Windows build and available cmdlets, it reports:

- DNS servers assigned to network adapters that are currently up.
- DoH server mappings and relevant configuration fields, where exposed by Windows.
- A normal Windows DNS resolution probe for `www.microsoft.com`.

This is a configuration and resolver check—not packet-level proof that a particular DNS request was encrypted or that all DNS traffic uses DoH. Use packet capture and Wireshark when you need to inspect observed network traffic.

## 8. Complete local firewall backup and restore

Choose **[2]** from the main menu to open the backup/restore submenu:

| Key | Action |
|---|---|
| **1** | Save the current complete local firewall policy to `rulle.wfw`. |
| **2** | Restore the saved local firewall policy from `rulle.wfw`. |
| **Q** | Return to the main menu. |

The backup uses Windows' native `netsh advfirewall export` and stores the `.wfw` file with SHA-256 and an integrity manifest. The manifest also records the local profile settings and a fingerprint of the persistent local firewall-rule inventory.

Before importing a backup, the script checks the saved file's length and SHA-256. It creates a temporary recovery export of the currently active local policy, imports the saved file, and verifies the restored profile settings and local persistent-rule inventory. The temporary recovery file is removed after a verified restore; if automatic recovery fails, the recovery file is retained for manual use. The saved `rulle.wfw` remains available after successful restore.

**Restore warning:** Restoring `rulle.wfw` replaces the current local firewall rules and profile settings with the saved local policy. Save a fresh backup before restoring if you may need the current state. Domain/Active Directory Group Policy is controlled separately and may reapply its settings.

The `.wfw` backup is a firewall-policy backup. It is not a general Windows backup and does not include packet captures, process fingerprint reports, application files, or the Security event log. The separate Filtering Platform audit-state record is not part of `rulle.wfw`.

## 9. Files and storage locations

All working data is stored under:

`C:\ProgramData\ZeroTrustFirewallMonitor\`

| Path | Purpose |
|---|---|
| `rulle.wfw` | Complete exported local firewall policy. |
| `FirewallState.json` | Integrity-protected backup manifest, profile values, and local rule inventory fingerprint. |
| `ZeroTrust_Monitor.log` | Operational log and warnings. |
| `FilteringPlatformAuditState.json` | Separate integrity-protected record of the original Filtering Platform Connection Failure auditing state, when the monitor needs to enable it. |
| `RememberedDecisions.json` | Integrity-protected exact-connection ignore decisions. |
| `Fingerprints\` | Running-process SHA-256 inventory reports in CSV and JSON. |
| `Captures\` | Original ETL captures, converted PCAPNG files, and packet-capture ownership/status markers. |

The script restricts access to its state directory, capture directory, and fingerprint directory to local Administrators and SYSTEM using Windows ACLs. Keep administrator access to this data limited to trusted accounts.

## 10. Requirements

- Windows with the built-in Windows Defender Firewall PowerShell module (`NetSecurity`).
- **Windows PowerShell 5.1** is the target runtime.
- Run from an **elevated Administrator** session. The script checks for administrative privileges and stops if it is not elevated.
- Windows Security auditing and access to the Security event log for event 5157 monitoring.
- Windows `auditpol.exe` for checking/enabling Filtering Platform Connection Failure auditing.
- Windows `pktmon.exe` for integrated packet capture and ETL-to-PCAPNG conversion.
- Wireshark desktop (`Wireshark.exe`) for interactive packet inspection. Wireshark is optional for the firewall monitor itself.

When Filtering Platform Connection Failure auditing is disabled, starting the monitor asks for the explicit `ENABLE` approval before changing that audit setting. If approval is not given, the event-driven monitor does not start. Group Policy may prevent the setting from being changed.

## 11. Installation and launch

1. Save `ZeroTrust_Firewall_Monitor_0.7.7.ps1` to a folder of your choice.
2. Open **Windows PowerShell as Administrator**.
3. Run the script by its full path, for example:

```powershell
powershell.exe -NoProfile -File "C:\Tools\ZeroTrust_Firewall_Monitor_0.7.7.ps1"
```

4. Create or refresh `rulle.wfw` from main menu **[2] → [1]** before applying strict mode.
5. Start the monitor with **[1]** when ready.
6. Use **[C]** to capture traffic and **[C] → [5]** to inspect the latest completed capture in Wireshark.

If your organization's execution policy prevents script execution, follow its approved process rather than weakening the machine-wide policy.

## 12. Operational notes and limitations

- The monitor processes new Security event 5157 records from its starting cursor. It does not backfill historical blocked events.
- Event auditing must be enabled for the monitor to receive the expected event stream. Audit configuration and firewall profile configuration are separate from packet capture.
- Some protected processes do not expose an executable path; they are reported as unavailable and cannot be fingerprinted by path.
- A valid Authenticode signature identifies a signed file and publisher; it is not a safety guarantee.
- SHA-256 is a file-integrity identifier, not malware detection.
- Temporary allow/block firewall rules are retained after their recorded expiry. The script warns; it does not automatically remove the rule.
- Existing firewall rules are preserved by the monitor's rule-decision and maintenance logic. The explicit **[2] Restore** action is different: it intentionally replaces the current local firewall policy with the saved `rulle.wfw` policy.
- Group Policy can override local firewall settings and can reapply after a restore.
- Packet captures are limited to the NIC capture configuration and 512-byte snap length; they are not a full forensic acquisition of every host data source.
The script is for technically capable Windows users who want to see what is blocked, make rule solutions, federal fingerprint executables, traffic capture, and inspect the packing results in Wireshark.

The script is for technically capable Windows users who want to see what is blocked, make rule solutions, federal fingerprint executables, traffic capture, and inspect the packing results in Wireshark.
I wish you all a good time with this perfect script.

