# Maven monorepo — incremental build with GIB
Date: 2026-04-12
Project: monorepo-cicd
Tags: maven, cicd, bitbucket, gib, incremental-build

## Problem
Naive use of `-pl`/`-am`/`-amd` Maven flags in a monorepo pipeline caused two
problems: upstream module artifacts missing from `~/.m2` (build failures), and
upstream tests re-running even when only a downstream module changed (slow builds).

## What you tried
- `-pl <changed> -amd` alone: downstream modules fail because upstream artifacts
  aren't in `~/.m2`
- `-pl <changed> -am -amd`: upstream tests still run, defeating the purpose
- Custom shell scripting to detect changed modules via `git diff`: brittle,
  doesn't understand Maven dependency graph

## Decision / outcome
Two-pass pattern with gitflow-incremental-builder (GIB):

**Pass 1** — install upstreams without running their tests:
```
mvn install -DskipTests \
  -Dgib.argsForUpstreamModules="-DskipTests" \
  -Dgib.skipTestsForUpstreamModules=true
```

**Pass 2** — verify only changed modules and their dependents:
```
mvn verify \
  -Dgib.enabled=true \
  -pl <changed-modules> -amd
```

## Rationale
GIB understands the Maven dependency graph natively — no custom shell scripting.
`skipTestsForUpstreamModules` prevents upstream tests running while still
producing artifacts in `~/.m2`. Two-pass approach keeps the changed-module
tests running in full verify mode.

Gradle was assessed as more elegant for this out of the box, but migration cost
is too high given existing Maven investment.

## Open questions
- Does `gib.argsForUpstreamModules` compose correctly when modules also use
  `-DskipTests` in their own profiles?
- Cache hygiene: need to exclude org artifacts from Bitbucket Pipelines cache
  to avoid stale upstream artifacts bleeding across pipeline runs.

## Key snippets
```yaml
# bitbucket-pipelines.yml (sketch)
- step:
    caches:
      - maven
    script:
      - mvn install -DskipTests -Dgib.skipTestsForUpstreamModules=true
      - mvn verify -Dgib.enabled=true -pl ${CHANGED_MODULES} -amd
    after-script:
      - find ~/.m2/repository/com/yourorg -name "*.jar" -delete
```
