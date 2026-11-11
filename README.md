# Zero-Trust-Firewall-Monitor
Zero-Trust Firewall Monitor applies a strict Windows Firewall profile and monitors new Windows Filtering Platform block events. When a supported blocked connection is detected, the script displays the connection details and offers a set of explicit actions.

Built around four practical priorities:

1. **Executable identity with SHA-256** — fingerprint applications, check Authenticode signatures and publishers, and detect changes to executables associated with managed allow rules.
2. **Explicit firewall decisions** — inspect newly blocked connections and decide whether to allow, block, or ignore the exact connection.
3. **Native packet capture** — capture network traffic with Windows `pktmon`, preserve the original ETL, and convert the capture to PCAPNG.
4. **Direct Wireshark control** — launch the Wireshark desktop application against the latest PCAPNG for interactive packet inspection.

The project aims to keep the workflow lean: one PowerShell script, Windows' built-in firewall and packet-monitoring tools, and Wireshark as an optional GUI analyzer.
