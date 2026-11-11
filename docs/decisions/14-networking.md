# Networking

Settled decisions for this area. Cite them by ID in specs, design notes and commit messages. To change one, follow the process in [README.md](README.md).

| ID | Question | Options considered | Decision | Decided by |
| --- | --- | --- | --- | --- |
| N1 | When does networking arrive? | Phase 3 · Phase 6 | Phase 6. Nothing before it needs the network. | Claude's pick, approved |
| N2 | Where does the stack come from? | Our own · Ported lwIP | Our own: Ethernet, ARP, IPv4, ICMP, UDP, TCP. | Claude's pick, approved |
| N3 | Where does it run? | In the kernel · As a userspace server | In the kernel. | Claude's pick, approved |
| N4 | What about IPv6? | From the start · Second · Never | Second. Address types are family-neutral from the start so it slots in. | Claude's pick, approved |
| N5 | What API do programs use? | BSD sockets · Our own | BSD sockets. | Claude's pick, approved |
| N6 | How is the network configured? | Static only · DHCP client service | DHCP client service, with static config as the fallback. | Claude's pick, approved |
| N7 | Where does name resolution live? | In libc · A resolver service with a cache | In libc first. A caching service later. | Claude's pick, approved |
| N8 | How good does TCP need to be? | Works on a quiet link · Correct retransmission and standard congestion control | Correct retransmission and standard congestion control, tested against the host with packet captures. | Claude's pick, approved |
| N9 | Which network tools ship first? | `ping` · A netcat-style tool · Name lookup · HTTP fetcher | All four. | Claude's pick, approved |
| N10 | Is there remote login? | Our own · Ported SSH · None | Ported SSH, after TLS-grade cryptography is in (X8). | Claude's pick, approved |
| N11 | Is the clock synced over the network? | Yes · No | Yes, a small time client. | Claude's pick, approved |
| N12 | Is there a firewall? | Yes · Not before 1.0 | Not before 1.0. | Claude's pick, approved |
