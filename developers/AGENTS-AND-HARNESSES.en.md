# Agents and harnesses — Execution Governance for developers

**[English](AGENTS-AND-HARNESSES.en.md) · [Español](AGENTS-AND-HARNESSES.es.md)**

Securing an MCP server is necessary and not enough. A coding agent also reaches the shell,
the file system, scripts, direct API calls and whatever automations its harness lets it
reach. This guide covers the technical model SecureStamp uses for that wider surface and the
tools that implement it.

> **Availability, checked 2026-10-05.** The published `@securestamp/mcp-guard@0.2.0` exposes
> `securestamp-mcp-guard`, `securestamp-mcp-doctor` and `securestamp-harness`. The separate
> `@securestamp/execution-governance` package is **not yet on npm**. Section §3 applies today
> after `npm install --save-dev @securestamp/mcp-guard`; §4's portable scenario CLI remains
> source-only until that package is published. The concepts in §1, §2 and §5–§7 apply today to
> the published Action Proof packages.

## 1. The model in one paragraph

**Agents can propose. Authority stays outside the agent.** An effect that matters — a push,
a deploy, a payment, a permission change — is admitted by an Execution Guardian that you run
and that holds the provider credentials. The agent never holds them. Coverage is limited to
the **declared and tested routes**: a route that bypasses the Guardian is not controlled, and
a route that was never probed is reported as *not evaluated*, never as protected by
inheritance from another route.

## 2. Three surfaces

| Surface | What goes wrong | Control |
| --- | --- | --- |
| **MCP servers** | A server launched through a shell, a secret written into the config or an unpinned package turns a tool into an unreviewed route of authority. | Doctor on the config file; MCP Guard mediates the call; Action Proof binds authorization to the exact effect. |
| **Agents** | The same attempt can arrive through MCP, the shell, a script or a direct API call. The agent's logs describe what it says it did, not what happened. | The Guardian admits or holds each mediated effect; an observer outside the agent records the outcome; a Task Contract bounds steps, resources, budget and validity. |
| **Harnesses** | Mounts, sockets, credential helpers, proxies and the effective network decide the agent's real authority, whatever the declared config says. | A reproducible lab probes the harness profile route by route against a permissive baseline. |

## 3. Doctor: diagnose the files you already have

Doctor is a **static** diagnosis. It reads only the files you name, never executes commands,
hooks or expressions from them, never resolves secrets or validates tokens, and never
overwrites the original: a correction is proposed as a reviewable copy.

```bash
npm install --save-dev @securestamp/mcp-guard

# JSON with an mcpServers object (stdio)
npx securestamp-mcp-doctor scan .mcp.json --propose

# Native agent-client configuration; --format names the file shape
npx securestamp-mcp-doctor scan-native path/to/settings.json --format=<settings-json-shape>
npx securestamp-mcp-doctor scan-native path/to/config.toml   --format=<mcp-toml-shape>

# A GitHub Actions workflow that runs an agent
npx securestamp-mcp-doctor scan-workflow .github/workflows/agent.yml

# A complete harness profile (HarnessProfileV1)
npx securestamp-mcp-doctor scan-harness harness-profile.json --propose
```

Run `securestamp-mcp-doctor` without arguments to print the accepted `--format` values. They name a **file shape** — a settings JSON with permissions and tools, or a TOML file with an MCP servers table — not an endorsement of any client. A format
Doctor does not recognize is reported as `UNSUPPORTED_FORMAT`; it is never silently skipped.

Findings for MCP server configuration:

| Code | Meaning |
| --- | --- |
| `INLINE_SECRET` | A token, key or password written into the config. The value is redacted and an environment-variable placeholder is proposed. |
| `SHELL_EXECUTION` | A server launched through `sh`, `bash`, `pwsh` or `cmd` with `-c`. |
| `UNPINNED_PACKAGE` | A package runner invoked without a pinned version (including `@latest`). |
| `NON_STDIO_TRANSPORT` | A transport outside the family this diagnosis covers. Reported, not guessed at. |
| `SUSPICIOUS_ARGUMENT` | An argument shaped like a credential or an escape from the declared command. |
| `MISSING_COMMAND` | A server entry with nothing to launch. |
| `INVALID_CONFIG` | The file did not parse as the declared format. |
| `UNSUPPORTED_FORMAT` | Not a format this diagnosis covers. |

Native-config and workflow modes add their own codes — for example unscoped permissions,
configuration layers that a single file cannot reveal, untrusted triggers that reach secrets,
checkout of external content, unpinned actions and reusable workflows that cannot be
resolved statically. The command prints each finding with its file, field and proposed fix.

**Limits you should know.** A single file does not reveal managed policies, command-line
overrides or inherited configuration; Doctor lists which layers it read and which remain
unknown. A pinned version does not prove integrity — comparing the artifact that actually ran
with the approved one is the executor's job. The known limitation of the current proposal:
it only replaces environment variables whose names look like secrets, so a value in an
argument, URL or header can still appear in it. **Review the patch before applying it.**
A static diagnosis never establishes isolation or protection.

## 4. The portable scenario

A run is described by one versioned JSON document, validated by the same schema in the CLI,
the library and any dashboard that consumes it. Editing it creates a new revision; a run is
immutable with respect to the revision it started from.

```json
{
  "typ": "SSPI-execution-scenario",
  "version": "1",
  "scenarioId": "exact-export-local",
  "revision": 1,
  "name": "Exact Export local fixture",
  "description": "Synthetic reference scenario; it never executes project code.",
  "parameters": { "target": "fixture://project-a", "dryRun": false },
  "source": {
    "kind": "external-project",
    "projectRef": "fixture://project-a",
    "candidateDigest": "sha256:<64 hex>",
    "baseDigest": "sha256:<64 hex>"
  },
  "runner": {
    "runnerId": "runner-lr0-reference",
    "profileId": "lr0-reference",
    "backend": "synthetic-process",
    "observerLeaseSeconds": 3,
    "controlLeaseSeconds": 10
  },
  "limits": { "maxEffects": 1, "timeoutSeconds": 30 },
  "steps": [
    {
      "stepId": "step-1",
      "operation": "customer.securestamp.draft.update",
      "effectDigest": "eff_v1:<64 hex>",
      "resourceDigest": "sha256:<64 hex>",
      "expected": "succeeded"
    }
  ],
  "createdAt": "1970-01-01T00:00:00.000Z"
}
```

```bash
# @securestamp/execution-governance is not on npm yet. Maintainers run it from a checkout:
pnpm --filter @securestamp/execution-governance build
node packages/execution-governance/dist/cli.js validate scenario.json
node packages/execution-governance/dist/cli.js run scenario.json --hold-before=step-1
```

`validate` prints the scenario id, revision and digest. `run` executes the **synthetic**
reference: it never runs project code, and its report is labeled as simulated evidence.
`--hold-before` places a hold before a step so you can see a held effect that is never
admitted.

The same package exposes the contracts as a library: scenario schemas and digests, a control
plane with idempotent `hold` / `resume` / `stop` orders, and redacted reports that can be
built, verified and compared. Importing it inside an agent hands the agent no keys, no
approval authority and no control over the observer.

## 5. Exact Export — only the reviewed candidate leaves

1. **Prepare** the change in quarantine, on a snapshot of the project. The working copy's
   `.git` is never reused as a trusted base.
2. **Freeze** the candidate before review: base, bytes and digests, ref, destination and
   prior state.
3. **Authorize** that candidate. A later change invalidates the approval; a `--yes` flag does
   not replace MFA or quorum.
4. **Execute** through the Guardian, checking the destination's state so drift and races are
   never overwritten.
5. **Verify** the postcondition independently and keep the receipt.

The initial destination is a local Git repository. With an authorized profile and
destination, the GitHub adapter creates a **new ref** only — no branch update, force-push,
merge or deploy; opening a pull request is a separate effect with its own authorization.
Custody of the export credential is claimed only for a profile that demonstrates it with the
effective identity, mounts, sockets and network, plus canaries from the agent's context.

## 6. Hold, resume, stop

| Order | What it does |
| --- | --- |
| `hold` | The Guardian closes admission of new mediated effects and keeps budget, consumed grants and history. It does not freeze calls already sent and does not queue stale effects for later. |
| `resume` | Mandate, validity, policy, profile, observer and budgets are revalidated before admission reopens; every new request is evaluated again. |
| `stop` | Terminal for the run: durably closes admission, cancels pending work and terminates supervised processes and their children. It prevails over `resume` and over a reconnection. |

The agent may keep reasoning while its effects are held; the correct display is *effects held
/ process running*. An HTTP `200` only confirms that an order was received — requested,
applied and observed are reported separately. If the remote control channel drops while the
observer is healthy, admissions stay held locally; if the observer fails, the profile cuts
egress and terminates the container. The local stop never depends on a dashboard.

Run states: `queued · running · held · stopping · stopped · completed · failed · incomplete`.

## 7. Reports and evidence levels

| Dimension | Values |
| --- | --- |
| Route result | `PASS · FAIL · SKIP` |
| Coverage | `protected · contradicted · partial · not_evaluated` |
| Integration | `simulated · integration_real · no_evaluated` |
| Completeness | `complete · incomplete` |
| Report outcome | `PASS · FAIL · INCOMPLETE` |
| Effect outcome | `succeeded · failed_no_effect · indeterminate` |

Three claims are kept apart:

- **Checksum is informational** — a report digest checks the bytes presented, but anyone who
  can rewrite the report can recompute it; it is not issuer authentication.
- **Signed evidence trusted by this operator** — a complete Action Proof bundle is verified
  offline only against grant and transparency anchors installed outside the artifact, and it
  binds the exact candidate, destination, authority and observed receipt result.
- **Reproduced by a third party** — someone else ran the same pack and got the same result.

A `FAIL`, a `SKIP` or an unevaluated route stays visible. An instrumentation failure leaves a
run `INCOMPLETE`, never `PASS`, and the absence of an event is never evidence of absence.

## 8. What this does not do

- It does not make a model safe or prove general alignment; it tests execution limits on a
  declared profile.
- It does not control routes that bypass the Guardian.
- It does not undo an effect that already happened — `hold` and `stop` are not rollback.
- A receipt shows integrity and scope under its anchors; it does not show that the approved
  code is benign or that anyone independent audited it.
- Missing or indeterminate evidence remains a limitation; it does not close the gate. A redacted
  summary identifies the evidence it supports but never inherits the signature of the full bundle.
- It is not an organization-wide kill switch: the first version controls the chosen run and
  its declared runners.
- Compatibility is stated per profile, version and environment. A result on one harness does
  not transfer to another operating system, route or client.

## 9. Where to go next

- [Getting Started](GETTING-STARTED.en.md) — connect to MCP Guard, install the published
  packages, verify a receipt offline.
- [Protocol v0.2](../protocol/SECURESTAMP-PROTOCOL-v0.2.en.md) — the normative Proof-of-Intent
  specification.
- [securestamp.org/en/docs/action-proof](https://securestamp.org/en/docs/action-proof),
  [/agent-lab](https://securestamp.org/en/agent-lab), [/kit](https://securestamp.org/en/kit) and
  [/evidence](https://securestamp.org/en/evidence) — Action Proof, the lab's profiles and
  evidence matrix, the configuration kit and the evidence levels.
- Counterexamples are welcome as issues with seed, profile and observed result. Report
  vulnerabilities privately to **security@securestamp.org**.
