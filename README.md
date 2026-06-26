# Sentuh Rasa WA Bridge

WhatsApp bot untuk Sentuh Rasa — risoles frozen & goreng.

## Architecture

Baca [CLAUDE.md](CLAUDE.md) untuk arsitektur lengkap, flow message, dan state machine.

## Quick Start


added 67 packages, and audited 68 packages in 4s

23 packages are looking for funding
  run `npm fund` for details

3 vulnerabilities (1 moderate, 2 high)

To address all issues, run:
  npm audit fix

Run `npm audit` for details.
[PM2] Starting /opt/hermes/sentuh-rasa/server.js in fork_mode (1 instance)
[PM2] Done.
┌────┬───────────────────┬─────────────┬─────────┬─────────┬──────────┬────────┬──────┬───────────┬──────────┬──────────┬──────────┬──────────┐
│ id │ name              │ namespace   │ version │ mode    │ pid      │ uptime │ ↺    │ status    │ cpu      │ mem      │ user     │ watching │
├────┼───────────────────┼─────────────┼─────────┼─────────┼──────────┼────────┼──────┼───────────┼──────────┼──────────┼──────────┼──────────┤
│ 0  │ pm2-live-logs     │ default     │ N/A     │ fork    │ 0        │ 0      │ 15   │ errored   │ 0%       │ 0b       │ hermes   │ disabled │
│ 1  │ wa-bridge-sbsr    │ default     │ 1.0.0   │ fork    │ 3088621  │ 0s     │ 0    │ online    │ 0%       │ 44.8mb   │ hermes   │ disabled │
└────┴───────────────────┴─────────────┴─────────┴─────────┴──────────┴────────┴──────┴───────────┴──────────┴──────────┴──────────┴──────────┘

## Stack

- Node.js (CommonJS)
- Express
- OpenClaw (LLM backend)
- Meta WhatsApp Cloud API
- Biteship (shipping)
