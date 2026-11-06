# Crestron .NET 8 Patterns

Cross-project reference for Crestron SimplSharpPro (.NET 8) control system patterns. Covers reusable components, driver conventions, and project layout distilled from three production projects: `solventum` (Canteen), `st-pius-x` (Treacy), and `unsw` (L5 Level 3 East).

## Contents

- [docs/crestron-dotnet8-patterns.md](docs/crestron-dotnet8-patterns.md) — main reference
- [docs/cp4-deployment-gotchas.md](docs/cp4-deployment-gotchas.md) — CP4 deployment pitfalls (build, paths, SSH)

## Quick Reference

| Topic | Summary |
|---|---|
| Stack | SimplSharpPro (.NET 8), `HttpListener` WebSocket, MS DI, xUnit |
| Canonical source | `c:\repos\solventum\canteen\src\` |
| Project layout | `Contracts / Core / Drivers / Transport / Composition / Host / DevHost` |
| Wire format | Full state on connect, JSON Patch diffs on update |

### High-Priority Components

WebSocket server · Static file host · Protocol · State store · Action dispatcher · PIN service · Config stack · Inactivity monitor · Device status poller · Composition root

### Device Driver Catalog

Epson projector · Grandview screen · Crestron AirMedia / NVX · Samsung MDC · Lumens camera · PTZ (VISCA) · Biamp Tesira (via Q-SYS QRC) · Shure ULXD · SNMP monitor

## Starting a New Project

See the [bootstrapping checklist](docs/crestron-dotnet8-patterns.md#starting-a-new-project) at the end of the reference doc.
