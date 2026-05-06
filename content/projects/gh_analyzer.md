---
title: "Alert Enrichment: Build vs. Buy"
date: 2026-04-04
draft: false
section: "Detection & Response"
type: "project"
tags: ["Detection engineering", "Alert tuning"]
description: "Analyzes a GitHub repository's workflow files for supply chain security risks. Takes a local repo path (or clones from URL) and outputs a security audit report."
---
What it analyzes:

1. Action pinning: Are Actions pinned to SHA or using mutable tags (e.g., @v3 vs. @abc123)?
2. Dependency tree: For each Action used, resolve its dependencies (Actions that call other Actions)
3. Permissions: Does the workflow request more permissions than needed? (permissions: write-all is a red flag)
4. Secrets exposure: Which steps have access to secrets? Are secrets passed to third-party Actions?
5. Outgoing network: Flag Actions known to make outbound connections (based on a curated list + heuristic analysis of Action source)

[GH Analyzer source code](https://github.com/CCbedair/gh_analyzer)