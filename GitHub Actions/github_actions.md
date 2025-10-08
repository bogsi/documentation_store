---
id: github-actions
title: GitHub Actions
sidebar_position: 1
description: Automate, build, and deploy your workflows using GitHub Actions.
---

# ⚙️ GitHub Actions

**GitHub Actions** is a CI/CD automation platform built directly into GitHub.  
It allows you to build, test, and deploy code automatically when events like pushes or pull requests occur.

---

## 🚀 Key Features

- **Native GitHub Integration** — trigger workflows directly from commits, PRs, or issues.  
- **Reusable Workflows** — define modular YAML-based pipelines that can be shared across repositories.  
- **Cross-Platform Support** — run jobs on Linux, Windows, or macOS runners.  
- **Secret Management** — securely handle credentials and tokens.  
- **Marketplace** — leverage thousands of prebuilt community actions.

---

## 🧩 Basic Workflow Example

Here’s a minimal workflow that runs tests whenever code is pushed to the `main` branch:

```yaml title=".github/workflows/test.yml"
name: Run Tests

on:
  push:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test
