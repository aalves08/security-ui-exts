# SUSE Security

Installs the SUSE Security extensions for Rancher Manager as a single unit. One install,
one version, one entry on the Extensions screen.

## What you get

| Extension | What it does |
|---|---|
| **SUSE Security Vulnerability Scanner** | Scans images, workloads and nodes for known vulnerabilities, and surfaces the results on the resources they affect. Adds a Vulnerabilities column to workload lists and a dashboard summarising scan coverage. |
| **SUSE Security Runtime Enforcer** | Reviews and applies runtime security policies, showing proposed policies alongside the ones currently active in the cluster. |

## Installing

Install it from **Extensions** like any other extension. Both members are deployed
together and appear in the cluster navigation once the page reloads.

Members are not offered individually — the bundle is the unit of installation. If you
need one without the other, install the per-extension charts instead.

## Upgrading and uninstalling

The bundle is a single Helm release, so upgrade and uninstall apply to the whole set.
Upgrading moves every member to the versions that ship with the target bundle version;
uninstalling removes all of them.

The version shown on the card is the **bundle's** version. Each member keeps its own
version internally, listed in the release notes for that bundle version.

## Compatibility

| Requirement | Version |
|---|---|
| Rancher | >= 2.11.0 |
| Extensions API | >= 3.0.0, < 4.0.0 |
| Kubernetes | >= 1.16.0 |

Requires Rancher Prime.

## Support

Issues and feature requests: https://github.com/rancher/security-ui-exts/issues
