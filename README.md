# Busbar GitHub Workflow Parser

A reusable GitHub Action that serves a dual purpose for securing Salesforce deployments:

1. **Integrity Mode (Default)**: Parses the executing workflow into a Component Integrity Matrix and transmits it to a Salesforce org via OIDC for zero-trust authorization.
2. **Scanner Mode**: A static analysis engine that evaluates workflows against security rules (e.g., injection vectors, risky `pull_request_target` usage) and emits SARIF/JSON vulnerability reports.

---

## 🛡️ Scanner Mode (Static Analysis)

When `report: 'true'` is provided, the parser bypasses Salesforce synchronization and acts as a security scanner.

**Behavior:**
- If `workflow` is explicitly provided, it scans that specific file.
- If `workflow` is **omitted** (the default), it automatically discovers and scans **ALL** workflows under `.github/workflows/`.

### Example: Scan all workflows and upload SARIF to GitHub Code Scanning
```yaml
jobs:
  security-scan:
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@v4
      
      - id: scan
        uses: busbar-actions/gh-workflow-parser@main
        with:
          report: 'true'
          format: 'both' # Generates both JSON and SARIF
          fail-on: 'high' # Fail the build if High/Critical findings are discovered

      - name: Upload SARIF to GitHub Code Scanning
        if: always() && steps.scan.outputs.sarif-path != ''
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: ${{ steps.scan.outputs.sarif-path }}
```

---

## 🔗 Integrity Mode (OIDC Token Exchange)

When `report` is `false` (default), the action parses the specific workflow running the current deployment and POSTs the payload to your Salesforce org.

**Behavior:**
- The `workflow` input is **required** in Integrity Mode. You must explicitly declare the path of the executing workflow.
- Requires `id-token: write` permissions for OIDC authentication.

### Example: Sync Integrity Matrix to Salesforce
```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
    steps:
      - uses: actions/checkout@v4
      
      - uses: busbar-actions/gh-workflow-parser@main
        with:
          target-instance: 'https://my-domain.my.salesforce.com'
          workflow: '.github/workflows/deploy.yml' # REQUIRED in Integrity Mode
```

---

## Inputs

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `report` | No | `'false'` | Set to `'true'` to enable Scanner Mode (skips OIDC sync). |
| `format` | No | `'json'` | Report output format in Scanner Mode: `json`, `sarif`, or `both`. |
| `fail-on` | No | `'none'` | Fail the action (exit 1) if any finding is at or above this severity: `critical`, `high`, `medium`, `hygiene`, `none`. |
| `target-instance` | Yes* | `''` | **Required for Integrity Mode.** The URL of the Salesforce org to authenticate against. |
| `workflow` | Yes* | `''` | **Required for Integrity Mode.** Path to the workflow file. Optional in Scanner Mode (scans all if omitted). |
| `sha` | No | `github.sha` | Git Commit SHA of the executing workflow context. |
| `rest-endpoint` | No | `/services/apexrest/busbar/GitHubWorkflowSync` | The Salesforce REST endpoint for OIDC sync. |
| `upload-artifact`| No | `'true'` | Automatically upload the JSON report as a GitHub Actions workflow artifact. |
| `artifact-name` | No | `workflow-integrity-report` | The name of the uploaded artifact. |
| `sarif-out` | No | `''` | Custom output path for the SARIF file. |
| `report-out` | No | `''` | Custom output path for the JSON file. |

## Outputs

| Name | Description |
|------|-------------|
| `workflow-hash` | The computed SHA-256 hash of the workflow file. |
| `report-path` | Path to the generated JSON report. |
| `sarif-path` | Path to the generated SARIF report (if `format` was `sarif` or `both`). |
