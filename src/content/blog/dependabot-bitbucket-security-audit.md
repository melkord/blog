---
title: "A homemade Dependabot for Bitbucket: automated security audits for a Yarn monorepo"
description: "How I replaced a broken audit step with a 400-line TypeScript bot that finds vulnerabilities, updates what it can, and opens a PR with a summary table."
pubDate: 2026-02-20
tags: ["ci-cd", "security", "bitbucket", "typescript"]
---

We had `yarn npm audit` in our PR pipeline. One line, right before `yarn install`:

```yaml
- yarn npm audit --all --severity moderate --environment production
```

Then we commented it out. The reason is simple: every time a CVE showed up on a transitive dependency (a dependency of a dependency), every PR in the team got blocked. Nobody could merge until someone manually fixed the vulnerability.

Result: audit disabled, vulnerabilities piling up silently, nobody checking.

## The alternatives

Before writing code I evaluated the existing solutions.

**Dependabot**: doesn't support Bitbucket. End of story.

**Renovate**: supports Bitbucket, but configuring it for a monorepo with Yarn workspaces is an obstacle course. I spent an afternoon reading GitHub issues from people with setups similar to mine who couldn't get it to work. I set it aside.

**Snyk**: works, but it's a paid service for the tiers a team actually needs. For our use case (periodic audit, automatic PR) it seemed like overkill.

**Custom script**: the least elegant solution on paper, but the one that gave me full control. And in the end it turned out to be the simplest too.

## The solution

I wrote a TypeScript script of about 400 lines that does all the work. It runs as a custom pipeline on Bitbucket, triggerable manually or on a schedule. The flow:

1. Checks out the main branch and installs dependencies
2. Runs `yarn npm audit` and captures the output
3. Parses the result (which is in NDJSON format)
4. For each vulnerable package, tries to update it
5. If there are changes, commits, pushes, and creates (or updates) a PR on Bitbucket with a summary table

If there are no vulnerabilities and there's an open PR from a previous run, it leaves a comment noting that it may no longer be needed.

The full script is available on [GitHub](https://github.com/melkord/security-audit-bot). What follows are the interesting parts.

## Parsing the audit output

The first obstacle was understanding the output format. `yarn npm audit --json` doesn't return a single JSON object. It returns NDJSON: one JSON line per vulnerability found. Each line looks like this:

```json
{
  "value": "package-name",
  "children": {
    "Severity": "high",
    "Vulnerable Versions": ">=2.0.0 <2.3.1",
    "Tree Versions": ["2.1.0", "2.2.3"],
    "Dependents": ["workspace-frontend@workspace:apps/frontend"]
  }
}
```

The `Dependents` field is interesting: it contains the workspaces that depend (directly or transitively) on the vulnerable package. The same package can appear multiple times if it's used in different workspaces, so the parser aggregates the results:

```typescript
function parseAuditOutput(raw: string): AuditEntry[] {
  const packages = new Map<string, AuditEntry>();

  for (const line of raw.split('\n')) {
    const trimmed = line.trim();
    if (!trimmed) continue;

    let parsed: YarnAuditLine;
    try {
      parsed = JSON.parse(trimmed);
    } catch {
      continue;
    }

    if (!parsed.value) continue;

    const children = parsed.children ?? {};
    const workspaces = (children.Dependents ?? [])
      .map(d => d.replace(/@workspace:.+$/, ''));
    const existing = packages.get(parsed.value);

    if (existing) {
      for (const ws of workspaces) {
        if (!existing.workspaces.includes(ws)) {
          existing.workspaces.push(ws);
        }
      }
    } else {
      packages.set(parsed.value, {
        name: parsed.value,
        severity: (children.Severity ?? 'unknown').toLowerCase(),
        vulnerableVersions: children['Vulnerable Versions'] ?? 'unknown',
        currentVersions: children['Tree Versions'] ?? [],
        workspaces,
      });
    }
  }

  return [...packages.values()];
}
```

The `try/catch` inside the loop is intentional: `yarn npm audit` sometimes mixes JSON output with plain text logs. Ignoring unparseable lines is the most robust choice.

## Updating packages and handling transitive dependencies

Once we have the list of vulnerable packages, the script tries to update them one by one with `yarn up`:

```typescript
for (const { name } of auditEntries) {
  console.log(`Updating ${name}...`);

  if (runSafe(`yarn up ${name}`) !== null) {
    updated.push(name);
  } else {
    failed.push(name);
    console.log('  FAILED (may be a transitive dependency)');
  }
}
```

This is where the distinction between direct and transitive dependencies matters. If a vulnerable package is a direct dependency, `yarn up` updates it without issues. If it's transitive, the command fails: you can't directly update a dependency that's not in your `package.json`.

The script doesn't stop in that case. It marks the package as "failed" and moves on. The PR will clearly show which packages were updated and which weren't, with a note explaining why.

This was exactly the problem that had made the audit unusable in the PR pipeline: a vulnerable transitive dependency blocked the entire team. Here the vulnerability gets reported, but the flow continues.

## The PR

The script creates (or updates) a single PR on a dedicated branch. If an open PR from a previous run already exists, it gets updated with the new state. Force push to the same branch, so there's never more than one audit PR open.

The PR description includes a markdown table that makes it immediately clear what was touched:

```
| Package    | Severity | Vulnerable     | Installed | Workspace    |
|------------|----------|----------------|-----------|--------------|
| some-lib   | high     | >=2.0 <2.3.1   | 2.1.0     | `frontend`   |
| other-pkg  | moderate | <1.5.0         | 1.4.2     | `backend`    |
```

Packages that couldn't be updated appear in a separate section with a note explaining that they're transitive dependencies and the only fix is waiting for the parent package to update.

## Pipeline configuration

The step is defined as a custom pipeline in `bitbucket-pipelines.yml`, triggerable manually from the Bitbucket UI or schedulable through repository settings:

```yaml
definitions:
  steps:
    - step: &security-audit
        name: Security Audit & Fix
        script:
          - git config user.email "bot@bots.bitbucket.org"
          - git config user.name "Security Audit Bot"
          - git remote set-url origin "https://x-token-auth:${AUDIT_TOKEN}@bitbucket.org/${WORKSPACE}/${REPO}.git"
          - git fetch origin develop
          - yarn install --immutable
          - npm install --prefix /tmp/audit-tools tsx bitbucket
          - NODE_PATH=/tmp/audit-tools/node_modules /tmp/audit-tools/node_modules/.bin/tsx scripts/securityAuditAndFix.ts

pipelines:
  custom:
    security_audit:
      - step: *security-audit
```

A few notes on this setup.

The script needs `tsx` (to run TypeScript) and the `bitbucket` library (for the APIs), but I don't want them as project dependencies. They're CI tools, not codebase tools. That's why they get installed in a temporary directory (`/tmp/audit-tools`) with `NODE_PATH` making them available. The project stays clean.

The bot authenticates with a Bitbucket Repository Access Token that only has the minimum required permissions: repository write (for pushing) and pull request write (for creating/updating PRs). The token is stored as a secured variable in the repository settings, not in the code.

The script also accepts `AUDIT_SEVERITY` to set the minimum severity threshold (defaults to `moderate`) and `AUDIT_DRY_RUN` to run the full flow without pushing or creating PRs.

We scheduled the pipeline to run once a week. It's been running since, and it works. Instead of blocking every PR when a transitive dependency has a CVE, the bot runs on its own, updates what it can, and opens a dedicated PR. The team reviews and merges it when ready, without pressure. Maintenance cost has been close to zero.
