# Security notes

- Run only on a machine where you are authorized to administer Windows Firewall.
- The process fingerprint inventory records process names, PIDs, executable paths, file sizes, timestamps and SHA-256 hashes. Treat the reports as system inventory data.
- Reports are stored under ProgramData with ACLs restricted to local Administrators and SYSTEM. A local administrator can still read or alter them.
- A SHA-256 match proves byte identity only; it does not prove safety.
- `pktmon` ownership is separate from detecting that a logger is active. The monitor must match its ownership marker and ETL path before stopping a running capture.
- If `pktmon status` is ambiguous, the monitor must not stop the session.
