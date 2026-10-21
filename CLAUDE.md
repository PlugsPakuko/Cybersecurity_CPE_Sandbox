# Cybersecurity Lab Sandbox

Hands-on cybersecurity lab course for KMUTT CPE: beginner level, THM/HTB style. Learners use a Kali attacker VM in the browser (Guacamole or noVNC), check out pre-built targets from a template pool, and submit flags to a central portal. Sessions are stateless.

## Modes

The sandbox offers two modes. Both are offline: no VNet has a route out, and Kali ships with preinstalled tools only.

- Open world: the student gets a Kali VM alone and can use or run anything on it. No lab, no instructions. Served by the Kali service.
- Tutorial: a lab with an instruction guide, attached to the student's running Kali. Served by the Lab service. The syllabus defines these labs.

## Scope

This project is the sandbox maker: it creates and manages sandboxes and exposes them for the LMS to integrate with.

There are only two teams: the LMS team and this project. There is no separate platform team.

- In scope: the API gateway and everything below it (control services, state store, infrastructure adapter, Proxmox-side resources such as student VNets, Kali VMs, lab topologies and the template pool), plus observability (health, metrics, logs). This includes multi-tenant isolation between students.
- Out of scope, owned by the LMS team: the LMS platform itself (auth, flags, reports) and the remote desktop gateway, Guacamole or noVNC (browser access to the Kali desktop). The LMS consumes this project's API through the gateway.

## Repository

- Remote `origin`: https://github.com/PlugsPakuko/Cybersecurity_CPE_Sandbox
- Local `main` is checked out and tracks `origin/main`. It holds the architecture files, the syllabus, `README.md`, `CLAUDE.md` and `.gitignore`.

## Architecture files in this folder

- `Architecture.drawio`: the current layered design, and the source of truth.
- `Architecture.png`: export of the diagram. Re-export after every diagram change.
- `Architecture.md`: written description of every component and the main flows. Keep it in line with the diagram.
- `.$Architecture.drawio.bkp`: draw.io's automatic backup, not a design version; ignored by git.

Design points:

- Kali service and Lab service are separate services behind the gateway, sharing one Postgres state store (sessions, job queue, VNet leases, audit). Each has an async API (202 + job id) and its own job worker.
- The Kali service owns the per-student VNet. The Lab service depends on the Kali service, never the reverse.
- Lab targets are either VMs or Docker containers, chosen per target in the lab manifest. Containers run inside one lab host VM per student session, on the student VNet, joined with macvlan so each container has its own IP and MAC. The lab host is created only when a lab with container targets starts (in parallel with Kali), kept until the session ends, and started through the QEMU guest agent via the driver. Windows, Active Directory, kernel-level and malware labs stay VMs.
- A reaper/reconciler process enforces session TTL and removes orphan VMs.
- Only the Proxmox driver holds the Proxmox API token.
- The remote desktop gateway reaches Kali through NIC0 on a firewalled access network; NIC1 sits on the student VNet.
- Malware analysis VMs run on an isolated node, pinned by placement.

## Observability requirements

- Health: live status per student, covering the Kali VM and the active lab topology.
- Metrics: usage statistics (for example peak users, sessions) and Proxmox statistics (CPU, RAM, storage).
- Logs: service logs, the audit trail (who spawned what), and security logs for malicious student activity. Attacks on lab targets inside the student's own VNet are expected and are not security events; attempts to leave the VNet, reach another student or the management network, spoof addresses, or abuse the API are.

## Syllabus files in this folder

- `Cybersecurity_Lab_Syllabus.docx`: source document.
- `Cybersecurity_Sandbox_Lab_Syllabus.pdf`: current export (2026-10-07).

The syllabus covers tutorial mode only; open world has no syllabus. It defines 10 core topics (65 labs) plus 1 optional topic (7 labs). Its module numbering runs 1.1 to 10.5 for core topics and O.1 to O.7 for the optional topic.