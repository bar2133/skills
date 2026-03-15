---
name: azure-devops-cli
description: >
  Install and configure Azure CLI with the Azure DevOps extension on macOS using Homebrew.
  Handles login, default organization/project configuration, and sanity checks.
  Use when the user wants to set up Azure DevOps CLI tools, install az cli,
  configure az devops defaults, or troubleshoot their Azure DevOps CLI setup on macOS.
---

# Azure DevOps CLI Setup (macOS)

Install Azure CLI via Homebrew, add the Azure DevOps extension, log in, configure
defaults, and verify the full setup with a sanity check. macOS only.

**Announce at start:** "I'm using the azure-devops-cli skill to set up Azure DevOps CLI tools."

## Prerequisites — macOS Guard

Before anything else, confirm the OS is macOS:

```bash
if [ "$(uname -s)" != "Darwin" ]; then
  echo "ERROR: This skill is macOS-only. Detected OS: $(uname -s)" >&2
  exit 1
fi
```

If not macOS, stop and inform the user this skill only supports macOS with Homebrew.

---

## Step 1 — Homebrew

Check if Homebrew is installed:

```bash
command -v brew
```

If missing, install it:

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

After install, ensure `brew` is on PATH by evaluating the shellenv:

```bash
eval "$(/opt/homebrew/bin/brew shellenv)"
```

---

## Step 2 — Install Azure CLI

Check if `az` is already installed:

```bash
az --version 2>/dev/null
```

If not installed, run:

```bash
brew update && brew install azure-cli
```

If already installed, skip and move on.

---

## Step 3 — Install Azure DevOps Extension

Check if the extension is present:

```bash
az extension show --name azure-devops 2>/dev/null
```

If not present, install it:

```bash
az extension add --name azure-devops
```

---

## Step 4 — Login

Check if the user is already logged in:

```bash
az account show 2>/dev/null
```

If not logged in (command fails), run interactive login:

```bash
az login
```

This opens a browser window for authentication. Wait for the user to complete the flow.

After login, display the active subscription:

```bash
az account show --query "{Subscription:name, TenantId:tenantId}" -o table
```

---

## Step 5 — Configure Defaults

Ask the user for their Azure DevOps organization URL and project name:

```
What is your Azure DevOps organization URL?
(e.g. https://dev.azure.com/my-org)
```

```
What is your default project name?
(e.g. my-project)
```

Then configure:

```bash
az devops configure --defaults organization=<ORG_URL> project=<PROJECT_NAME>
```

Verify the defaults were saved:

```bash
az devops configure --list
```

---

## Step 6 — Sanity Check

Run all checks and collect pass/fail results. Present as a summary table.

```bash
echo "=== Azure DevOps CLI Sanity Check ==="

# 1. CLI installed
echo -n "az CLI installed ........... "
az --version >/dev/null 2>&1 && echo "PASS" || echo "FAIL"

# 2. DevOps extension
echo -n "azure-devops extension ..... "
az extension show --name azure-devops >/dev/null 2>&1 && echo "PASS" || echo "FAIL"

# 3. Logged in
echo -n "Logged in .................. "
az account show >/dev/null 2>&1 && echo "PASS" || echo "FAIL"

# 4. Defaults configured
echo -n "Defaults configured ........ "
az devops configure --list 2>/dev/null | grep -q "organization" && echo "PASS" || echo "FAIL"

# 5. End-to-end connectivity
echo -n "DevOps connectivity ........ "
az devops project list --top 1 >/dev/null 2>&1 && echo "PASS" || echo "FAIL"

echo "======================================"
```

### Troubleshooting Failures

| Check | Fix |
|-------|-----|
| az CLI installed | Run `brew install azure-cli` |
| azure-devops extension | Run `az extension add --name azure-devops` |
| Logged in | Run `az login` |
| Defaults configured | Run `az devops configure --defaults organization=<URL> project=<NAME>` |
| DevOps connectivity | Verify org URL is correct and you have network access |

If any check fails, re-run the corresponding step above, then re-run the sanity check.

---

## Quick Reference

| Task | Command |
|------|---------|
| List projects | `az devops project list` |
| List repos | `az repos list` |
| Create a repo | `az repos create --name <name>` |
| List pipelines | `az pipelines list` |
| Trigger a pipeline run | `az pipelines run --name <name>` |
| List work items (query) | `az boards query --wiql "SELECT ... FROM WorkItems"` |
| Create a work item | `az boards work-item create --type Task --title "<title>"` |
| List PRs | `az repos pr list` |
| Create a PR | `az repos pr create --title "<title>" --source-branch <branch>` |
| Show current defaults | `az devops configure --list` |
| Switch organization | `az devops configure --defaults organization=<URL>` |
| Switch project | `az devops configure --defaults project=<NAME>` |
