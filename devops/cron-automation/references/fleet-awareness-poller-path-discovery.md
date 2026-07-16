# Fleet Awareness Poller Path Discovery

Session note: when a read-only fleet poll is asked to inspect coordination files, the live runtime may not mount the same home tree as the authoring machine.

Observed in this session:
- The active container reported `pwd=/root`, `whoami=root`, `HOME=/root`.
- Searches for the requested coordination paths found nothing in the live runtime.
- `/Users`, `/Users/jack.reis`, and the expected `Documents/Coordination` tree were absent.
- Local poller artifacts still existed under `~/.hermes/cubby/`, including the poll log and all-null state file.

Operational takeaway:
- Treat absent coordination paths as an environment mismatch first, not as evidence of no changes.
- Probe the live runtime directly before escalating or returning `[SILENT]`.
- If the poll state is all-null and the tree is unreachable, report the mismatch rather than inventing a delta.