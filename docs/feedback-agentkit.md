# AgentKit Feedback — Robinhood Stock Prediction Market

This is our feedback for the AgentKit Continuity prize, covering the four areas requested: docs and integration flow, Developer Portal, Sandbox App, and what we found confusing or broken.

## AgentKit docs and integration flow

The core integration pattern (verify SIWE signature, look up `humanId` via AgentBook, sign an attestation, consume it on-chain with `ecrecover`) is documented clearly enough that we built our relayer (`relayer/src/relayer.js`) and agent transaction builder (`decision-engine/src/agent-tx.js`) directly from it, without needing outside examples.

One gap we ran into: the docs describe the three-state `lookupHuman` result (`backed` / `unbacked` / `unknown`) at a high level, but don't give a reference implementation for handling the `unknown` case (an AgentBook RPC failure) distinctly from `unbacked` (a real "not registered" result). We followed the pattern from [poh-aggregator](https://github.com/andrevalenm/poh-aggregator)'s `lookupHumanBacking` instead, and credit that as prior art in our own code comments.

## Developer Portal

We didn't use the Developer Portal directly for this integration. Our AgentBook interaction goes through direct contract calls (`lookupHuman()`, `ecrecover` verification in our own contract) and the `agentkit-cli` tool, not through Developer Portal app registration. We can't give informed feedback on navigation or discoverability there.

## Sandbox App states, proof flows, errors, edge cases

We requested Sandbox access on 2026-09-05 because Taiwan has no stable Orb location. Access arrived on 2026-09-09, four days before the submission deadline.

Once we tested it, we found two separate paths, and both led back to a physical Orb requirement:

1. **The Sandbox App itself.** Its verification flow asks for a passport scan, then asks the user to find a nearby physical Orb. We didn't find a simulated or remote Orb path inside the app.

2. **The `agentkit-cli register` command.** We traced this back to its source (npm package 0.2.0 and the `main` branch on GitHub). It hardcodes a production `app_id`, not a staging one. `idkit-core` decides whether to route to a staging environment by checking if the `app_id` string contains `"staging"`. Since the hardcoded one doesn't, every QR code this CLI generates points to the real World App flow, regardless of which app scans it.

We want to be clear that this isn't a complaint about the wait for access itself, plenty of teams were in the same position, and the team was actively responding in Discord throughout. The specific finding is that once access arrived, we couldn't find a path through either the Sandbox App or the CLI that avoided the physical Orb requirement, which is the exact constraint that made us request Sandbox access in the first place.

## What was confusing, missing, broken, or hard to test

- `cli/REGISTRATION.md` documents a `--network base-sepolia` flag. It doesn't exist in the shipped code (checked both the published npm package and the `main` branch). Running `register` with this flag fails because the CLI's argument parser treats it as unknown.
- The npm page states registration is "gasless by default" and uses a hosted relay, but doesn't specify which chain that relay operates on versus which chain AgentBook itself lives on (World Chain, `eip155:480`). We had to read the source to confirm the actual registration transaction lands on World Chain, not Base, despite some docs implying otherwise.
- We could not find documentation describing what a Sandbox-issued identity is actually meant to satisfy on-chain. Specifically: does a Sandbox verification produce a credential that AgentBook's `groupId=1` (Orb-specific) check accepts, or is Sandbox access meant for something else (API integration testing, UI testing) that doesn't touch the real on-chain registration path? We couldn't determine this from the docs we had access to, and testing it directly led us back to the physical Orb requirement described above.

Given this, our agent wallet remains in an `unbacked` state on AgentBook as of submission. Our relayer correctly refuses to sign an attestation for it (`agent_not_human_backed`), and the demo video shows this refusal directly, along with our manual-override CLI path (`--manual-decision`) that lets us test the rest of the pipeline independent of AgentBook status. The `backed` branch of our code is verified by code review and unit tests only, not by a live on-chain bet. This is documented in `prompts/08-world-id-orb-availability-constraint.md`.
