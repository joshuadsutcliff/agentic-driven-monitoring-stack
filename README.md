# Pi 5 Monitoring Stack

A complete home-network monitoring and management stack for a Raspberry Pi 5, installed by your AI coding agent instead of a shell script.

There is no installer here. The repo's centerpiece is [`CHARTER.md`](CHARTER.md): a phase-by-phase setup charter written to be handed to an AI agent (Claude Code, or any agent that can run shell commands). The agent reads the charter, interviews you for your network details, then builds and verifies the stack on your hardware.

## What you get

One Raspberry Pi 5 running, in Docker:

- **Pi-hole**: LAN-wide DNS, ad-blocking, and local hostnames for every device
- **InfluxDB 2.x + Telegraf + Grafana**: metrics collection and dashboards, extensible to every machine in the house
- **ntfy**: push alerts to your phone, self-hosted
- **Uptime Kuma**: up/down monitoring with alert routing
- **MeshCentral**: self-hosted remote desktop and management for your other machines
- **Homepage + Homarr**: two dashboard front-ends integrating all of the above

Plus Tailscale (native) for secure remote access, with everything LAN-only or tailnet-only: nothing is exposed to the public internet.

## Why a charter instead of an install script

A script knows nothing about your network and dies the first time reality disagrees with its assumptions. An agent working from a charter can probe your actual hardware, adapt addressing and storage to what it finds, verify each phase before starting the next, and troubleshoot in place. The charter encodes the design and the hard-won gotchas; the agent supplies the adaptation.

Every phase ends with explicit verification commands and expected results. The riskiest step (taking over the house's DNS) ships with a written rollback plan.

## How to use it

1. Set up a Raspberry Pi 5 (64-bit Raspberry Pi OS) with an attached SSD, reachable over SSH.
2. Open Claude Code (or your agent of choice) with access to the Pi: either running on the Pi directly, or on another machine that can SSH to it.
3. Give it this file: `Read CHARTER.md and follow it.`
4. Answer its questions (static IP, router access, Tailscale account, which machines to manage), then let it work phase by phase.

Requirements: Raspberry Pi 5 (4 GB+ RAM, 8 GB recommended), an external SSD (USB or NVMe), a Tailscale account (free tier is fine), and router admin access if you want network-wide ad-blocking.

## Provenance

The design is distilled from three production deployments (a homelab monitoring VM and two small-business monitoring appliances) run and refined over months. The embedded warnings are real failure modes those deployments hit, including the MeshCentral silent agent-enrollment trap, the dashboard dual-URL rule, and USB storage UAS quirks on Pi hardware.

## License

MIT. Use it, fork it, adapt the charter for your own stack.
