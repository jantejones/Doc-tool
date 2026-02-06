# Azure DevOps Wiki to Blob Storage Sync

## Overview

This document details how to automatically sync markdown files from an Azure DevOps Wiki into the Azure Blob Storage container that feeds the Azure AI Search indexer (see [azure-search-integration-plan.md](./azure-search-integration-plan.md)). Because Azure DevOps Wiki is backed by a Git repository, we can treat it as a standard Git source and build a pipeline that clones, filters, and uploads content to blob storage on every wiki change.

---

## How Azure DevOps Wiki Storage Works

### Yes — It's a Git Repository

Azure DevOps Wiki is fully backed by a Git repository. There are two flavors:

| Type | Git Repo Name | Default Branch | Created By |
|------|--------------|----------------|------------|
| **Provisioned Wiki** (Project Wiki) | `<ProjectName>.wiki` | `wikiMain` | Azure DevOps automatically when you create the first wiki page |
| **Published Wiki** (Code Wiki) | Any existing repo | Any branch/folder you choose | You — by publishing an existing repo folder as a wiki |

Both store every page as a **markdown (.md) file** in the Git repo, version-controlled with full commit history.

### Wiki Repository Structure

```
<ProjectName>.wiki/
├── .order                          # Page ordering for the wiki root
├── Home.md                         # Root/home page
├── Getting-Started.md              # Page: "Getting Started"
├── Architecture/
│   ├── .order                      # Page ordering within this section
│   ├── Overview.md
│   └── Components.md
├── Runbooks/
│   ├── .order
│   ├── Incident-Response.md
│   └── Deployment.md
├── .attachments/                   # Hidden folder for images/files
│   ├── diagram.png
│   └── architecture/
│       └── flow.svg
└── README.md                       # Optional, not rendered as wiki page
```

**Key conventions:**
- Page titles use **hyphens** instead of spaces in filenames (`Getting-Started.md` → title "Getting Started")
- Each folder can contain a **`.order`** file that defines the display sequence of pages in the wiki sidebar
- **`.attachments/`** is a hidden folder at the root that stores all images and file attachments (max 19 MB per file)
- The repo is **hidden by default** in the Azure DevOps Repos UI — it won't show up unless you query with `includeHidden=True`

### Cloning the Wiki Repo

```bash
# Provisioned Wiki
git clone https://dev.azure.com/<org>/<project>/_git/<project>.wiki

# Published Wiki (just the regular repo)
git clone https://dev.azure.com/<org>/<project>/_git/<repo-name>
```

Authentication uses a **Personal Access Token (PAT)** with `Code > Read` scope, or a system-managed token in pipelines.

---

## Sync Architecture

```
┌──────────────────────┐    git clone     ┌──────────────────────┐
│                      │ ───────────────> │                      │
│  Azure DevOps Wiki   │                  │  Azure DevOps        │
│  (Git Repository)    │                  │  Pipeline Agent      │
│                      │                  │                      │
└──────────────────────┘                  └───────────┬──────────┘
                                                      │
                                                      │ az storage blob
                                                      │ upload-batch / azcopy
                                                      │
                                          ┌───────────▼──────────┐
                                          │                      │
                                          │  Blob Storage        │
                                          │  (markdown-docs)     │
                                          │                      │
                                          └───────────┬──────────┘
                                                      │
                                                      │ Indexer triggers
                                                      ▼
                                          ┌──────────────────────┐
                                          │  Azure AI Search     │
                                          │  (Index + Vectors)   │
                                          └──────────────────────┘
```

### Two Sync Strategies

| Strategy | Trigger | Pros | Cons |
|----------|---------|------|------|
| **Full sync** | Scheduled (e.g., nightly) or manual | Simple; guarantees consistency | Slower; re-uploads unchanged files |
| **Incremental sync** | Wiki push trigger via pipeline | Fast; only uploads changed files | Slightly more complex pipeline logic |

**Recommendation:** Use incremental sync with a weekly full sync as a safety net.

---

## Terraform — Pipeline Infrastructure

### Service Connection for Blob Storage

The pipeline needs access to the storage account. Use a service principal or managed identity.

```hcl
# Service principal for the pipeline to authenticate to blob storage
resource "azuread_application" "pipeline_sync" {
  display_name = "sp-wiki-sync-${local.resource_prefix}"
}

resource "azuread_service_principal" "pipeline_sync" {
  client_id = azuread_application.pipeline_sync.client_id
}

resource "azuread_service_principal_password" "pipeline_sync" {
  service_principal_id = azuread_service_principal.pipeline_sync.id
  end_date_relative    = "8760h" # 1 year
}

# Grant the service principal write access to the blob container
resource "azurerm_role_assignment" "pipeline_blob_contributor" {
  scope                = azurerm_storage_account.docs.id
  role_definition_name = "Storage Blob Data Contributor"
  principal_id         = azuread_service_principal.pipeline_sync.object_id
}
```

### Storage Account (Reference from Main Plan)

The same storage account and container from the main integration plan is used:

```hcl
# Already defined in azure-search-integration-plan.md
# azurerm_storage_account.docs
# azurerm_storage_container.markdown_docs (name: "markdown-docs")
```

---

## Azure DevOps Pipeline — Full Sync

This pipeline clones the wiki repo, filters to only `.md` files, and uploads them to blob storage.

### `azure-pipelines-wiki-sync.yml`

```yaml
trigger: none  # Triggered by wiki repo, not this repo

resources:
  repositories:
    - repository: wiki
      type: git
      name: <project>/<project>.wiki  # The wiki Git repo
      ref: wikiMain                    # Default branch for provisioned wikis

schedules:
  - cron: "0 2 * * 0"   # Weekly full sync: Sunday at 2 AM UTC
    displayName: "Weekly full sync"
    branches:
      include:
        - main
    always: true

pool:
  vmImage: "ubuntu-latest"

variables:
  - name: storageAccountName
    value: "stdocsearchdev"              # Must match Terraform output
  - name: containerName
    value: "markdown-docs"
  - name: azureSubscription
    value: "azure-service-connection"    # Azure DevOps service connection name

stages:
  - stage: SyncWikiToBlob
    displayName: "Sync Wiki MD Files to Blob Storage"
    jobs:
      - job: Sync
        displayName: "Clone Wiki and Upload to Blob"
        steps:
          # 1. Checkout the wiki repository
          - checkout: wiki
            clean: true
            fetchDepth: 1   # Shallow clone — we only need latest content

          # 2. Prepare: filter to only .md files, exclude .order and .attachments
          - script: |
              echo "=== Preparing MD files for upload ==="
              WIKI_DIR="$(Build.SourcesDirectory)"
              STAGING_DIR="$(Build.ArtifactStagingDirectory)/wiki-md"
              mkdir -p "$STAGING_DIR"

              # Copy .md files preserving folder structure
              cd "$WIKI_DIR"
              find . -name "*.md" \
                -not -path "./.attachments/*" \
                -not -name "README.md" \
                | while read -r file; do
                    dest_dir="$STAGING_DIR/$(dirname "$file")"
                    mkdir -p "$dest_dir"
                    cp "$file" "$dest_dir/"
                  done

              # Show what will be uploaded
              echo "=== Files staged for upload ==="
              find "$STAGING_DIR" -name "*.md" | sort
              echo "Total: $(find "$STAGING_DIR" -name "*.md" | wc -l) files"
            displayName: "Filter and stage MD files"

          # 3. Upload to blob storage
          - task: AzureCLI@2
            displayName: "Upload MD files to Blob Storage"
            inputs:
              azureSubscription: $(azureSubscription)
              scriptType: "bash"
              scriptLocation: "inlineScript"
              inlineScript: |
                STAGING_DIR="$(Build.ArtifactStagingDirectory)/wiki-md"

                echo "=== Syncing to blob storage ==="
                az storage blob upload-batch \
                  --account-name "$(storageAccountName)" \
                  --destination "$(containerName)" \
                  --source "$STAGING_DIR" \
                  --overwrite true \
                  --content-type "text/markdown" \
                  --auth-mode login \
                  --pattern "*.md"

                echo "=== Upload complete ==="
                az storage blob list \
                  --account-name "$(storageAccountName)" \
                  --container-name "$(containerName)" \
                  --auth-mode login \
                  --output table \
                  --query "[].{Name:name, Size:properties.contentLength, Modified:properties.lastModified}"
```

---

## Azure DevOps Pipeline — Incremental Sync

This pipeline triggers on every wiki push and only uploads the files that changed.

### `azure-pipelines-wiki-sync-incremental.yml`

```yaml
trigger: none

resources:
  repositories:
    - repository: wiki
      type: git
      name: <project>/<project>.wiki
      ref: wikiMain
      trigger:
        branches:
          include:
            - wikiMain        # Trigger on every wiki edit/push

pool:
  vmImage: "ubuntu-latest"

variables:
  - name: storageAccountName
    value: "stdocsearchdev"
  - name: containerName
    value: "markdown-docs"
  - name: azureSubscription
    value: "azure-service-connection"

stages:
  - stage: IncrementalSync
    displayName: "Incremental Wiki Sync"
    jobs:
      - job: Sync
        displayName: "Detect changes and sync"
        steps:
          - checkout: wiki
            clean: true
            fetchDepth: 2   # Need 2 commits to diff

          - script: |
              echo "=== Detecting changed files ==="
              WIKI_DIR="$(Build.SourcesDirectory)"
              STAGING_DIR="$(Build.ArtifactStagingDirectory)/wiki-md"
              DELETED_FILE="$(Build.ArtifactStagingDirectory)/deleted-files.txt"
              mkdir -p "$STAGING_DIR"

              cd "$WIKI_DIR"

              # Get list of changed files between last 2 commits
              CHANGED_FILES=$(git diff --name-only HEAD~1 HEAD 2>/dev/null || echo "")

              if [ -z "$CHANGED_FILES" ]; then
                echo "No changes detected, performing full sync"
                find . -name "*.md" \
                  -not -path "./.attachments/*" \
                  -not -name "README.md" \
                  | while read -r file; do
                      dest_dir="$STAGING_DIR/$(dirname "$file")"
                      mkdir -p "$dest_dir"
                      cp "$file" "$dest_dir/"
                    done
              else
                echo "Changed files:"
                echo "$CHANGED_FILES"

                # Separate added/modified from deleted
                echo "$CHANGED_FILES" | while read -r file; do
                  if [[ "$file" == *.md ]] && \
                     [[ "$file" != .attachments/* ]] && \
                     [[ "$file" != README.md ]]; then
                    if [ -f "$file" ]; then
                      # File exists — added or modified
                      dest_dir="$STAGING_DIR/$(dirname "$file")"
                      mkdir -p "$dest_dir"
                      cp "$file" "$dest_dir/"
                      echo "  [UPLOAD] $file"
                    else
                      # File no longer exists — deleted
                      echo "$file" >> "$DELETED_FILE"
                      echo "  [DELETE] $file"
                    fi
                  fi
                done
              fi

              echo "##vso[task.setvariable variable=hasDeleted]$([ -f "$DELETED_FILE" ] && echo true || echo false)"
              echo "##vso[task.setvariable variable=hasUploads]$([ "$(find "$STAGING_DIR" -name "*.md" | wc -l)" -gt 0 ] && echo true || echo false)"
            displayName: "Detect changed MD files"

          # Upload changed/new files
          - task: AzureCLI@2
            displayName: "Upload changed MD files"
            condition: eq(variables['hasUploads'], 'true')
            inputs:
              azureSubscription: $(azureSubscription)
              scriptType: "bash"
              scriptLocation: "inlineScript"
              inlineScript: |
                STAGING_DIR="$(Build.ArtifactStagingDirectory)/wiki-md"
                az storage blob upload-batch \
                  --account-name "$(storageAccountName)" \
                  --destination "$(containerName)" \
                  --source "$STAGING_DIR" \
                  --overwrite true \
                  --content-type "text/markdown" \
                  --auth-mode login \
                  --pattern "*.md"

          # Delete removed files from blob
          - task: AzureCLI@2
            displayName: "Delete removed files from blob"
            condition: eq(variables['hasDeleted'], 'true')
            inputs:
              azureSubscription: $(azureSubscription)
              scriptType: "bash"
              scriptLocation: "inlineScript"
              inlineScript: |
                DELETED_FILE="$(Build.ArtifactStagingDirectory)/deleted-files.txt"
                if [ -f "$DELETED_FILE" ]; then
                  while read -r blob_name; do
                    # Strip leading ./ if present
                    blob_name="${blob_name#./}"
                    echo "Deleting blob: $blob_name"
                    az storage blob delete \
                      --account-name "$(storageAccountName)" \
                      --container-name "$(containerName)" \
                      --name "$blob_name" \
                      --auth-mode login \
                      || echo "  Warning: could not delete $blob_name (may not exist)"
                  done < "$DELETED_FILE"
                fi
```

---

## Handling Wiki-Specific Artifacts

### `.order` Files — Exclude from Blob Storage

`.order` files control page ordering in the wiki sidebar. They are not documentation content and should **not** be uploaded to blob storage.

The pipelines above already exclude them (they only copy `*.md` files).

### `.attachments/` Folder — Optional Sync

Wiki images and file attachments live in `.attachments/`. If your search index or MCP tools need to serve images:

```yaml
# Add this step after the MD upload step
- task: AzureCLI@2
  displayName: "Upload wiki attachments (optional)"
  inputs:
    azureSubscription: $(azureSubscription)
    scriptType: "bash"
    scriptLocation: "inlineScript"
    inlineScript: |
      WIKI_DIR="$(Build.SourcesDirectory)"
      if [ -d "$WIKI_DIR/.attachments" ]; then
        az storage blob upload-batch \
          --account-name "$(storageAccountName)" \
          --destination "$(containerName)" \
          --destination-path ".attachments" \
          --source "$WIKI_DIR/.attachments" \
          --overwrite true \
          --auth-mode login
      fi
```

### Filename Normalization

Wiki filenames use hyphens for spaces (`Getting-Started.md`). If you want the search index `title` field to display "Getting Started" instead of "Getting-Started", handle this in the indexer skillset.

The **Custom WebApiSkill** from the main plan already extracts the title from the first `# heading` in the markdown content, which is the best approach. As a fallback, add a normalization step in the pipeline:

```bash
# Optional: create a metadata sidecar file for each MD file
for file in $(find "$STAGING_DIR" -name "*.md"); do
  filename=$(basename "$file" .md)
  # Convert hyphens to spaces for a display-friendly title
  display_title=$(echo "$filename" | sed 's/-/ /g')
  # Write metadata sidecar (Azure Search can read these)
  echo "{\"title\": \"$display_title\"}" > "${file%.md}.metadata.json"
done
```

---

## Terraform — Pipeline Triggers via Webhook (Alternative)

If you prefer to trigger the sync from outside Azure DevOps (e.g., from a Logic App or Azure Function), you can set up a service hook:

```hcl
# Azure Function to receive webhook and trigger pipeline
resource "azurerm_linux_function_app" "wiki_trigger" {
  name                = "func-wiki-trigger-${local.resource_prefix}"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  service_plan_id     = azurerm_service_plan.logic_app.id

  storage_account_name       = azurerm_storage_account.docs.name
  storage_account_access_key = azurerm_storage_account.docs.primary_access_key

  site_config {
    application_stack {
      python_version = "3.11"
    }
  }

  app_settings = {
    "DEVOPS_ORG_URL"        = "https://dev.azure.com/<org>"
    "DEVOPS_PROJECT"        = "<project>"
    "DEVOPS_PIPELINE_ID"    = "<pipeline-id>"
    "DEVOPS_PAT"            = "@Microsoft.KeyVault(VaultName=kv-${local.resource_prefix};SecretName=devops-pat)"
  }

  tags = local.common_tags
}
```

---

## Alternative: AzCopy for Large Wiki Repos

For wikis with thousands of pages or large attachments, `az storage blob upload-batch` can be slow. Use **AzCopy** for better performance:

```yaml
- script: |
    # Install AzCopy (already on hosted agents, but ensuring latest)
    wget -q https://aka.ms/downloadazcopy-v10-linux -O azcopy.tar.gz
    tar xzf azcopy.tar.gz --strip-components=1
    chmod +x azcopy

    # Sync using AzCopy (only uploads changed files, deletes removed ones)
    ./azcopy sync \
      "$(Build.ArtifactStagingDirectory)/wiki-md" \
      "https://$(storageAccountName).blob.core.windows.net/$(containerName)" \
      --delete-destination=true \
      --include-pattern="*.md" \
      --log-level=WARNING
  displayName: "AzCopy sync to blob storage"
  env:
    AZCOPY_AUTO_LOGIN_TYPE: "SPN"
    AZCOPY_SPA_CLIENT_SECRET: $(servicePrincipalKey)
    AZCOPY_SPA_APPLICATION_ID: $(servicePrincipalId)
    AZCOPY_TENANT_ID: $(tenantId)
```

**AzCopy `sync` vs `upload-batch`:**

| Feature | `az storage blob upload-batch` | `azcopy sync` |
|---------|-------------------------------|----------------|
| Differential upload | No (re-uploads everything) | Yes (compares MD5 hashes) |
| Delete removed blobs | Manual | `--delete-destination=true` |
| Parallelism | Limited | High (auto-tuned) |
| Speed (1000+ files) | Slow | Fast |
| Availability on agents | Built-in (Azure CLI) | Built-in on hosted agents |

---

## End-to-End Flow

```
 Developer edits wiki page in Azure DevOps
                    │
                    ▼
 Git commit pushed to <project>.wiki repo (wikiMain branch)
                    │
                    ▼
 Pipeline triggered (resource trigger on wiki repo)
                    │
                    ▼
 Agent clones wiki repo (shallow, fetchDepth: 1 or 2)
                    │
                    ▼
 Filter: keep only *.md, exclude .attachments/, .order, README.md
                    │
                    ▼
 Upload changed MD files to Blob Storage (markdown-docs container)
                    │
                    ▼
 Delete removed blobs (if incremental sync)
                    │
                    ▼
 Search indexer detects changes via blob change feed (next scheduled run)
                    │
                    ▼
 Indexer re-processes changed docs (skillset: split, keyphrase, embed)
                    │
                    ▼
 Updated content available via MCP tools (get_document, semantic_search)
```

**End-to-end latency:**
- Wiki edit → blob upload: **1-3 minutes** (pipeline execution)
- Blob upload → searchable: **up to 2 hours** (default indexer schedule `PT2H`)
- **Total worst case:** ~2 hours 3 minutes
- **To reduce:** Set indexer schedule to `PT5M` for near-real-time (increases Cognitive Services cost)

---

## Security Considerations

| Area | Recommendation |
|------|---------------|
| **Wiki repo access** | Pipeline uses `System.AccessToken` (automatic) — no PAT needed for same-project wikis |
| **Cross-project wikis** | Use a PAT stored in Azure Key Vault, referenced as a pipeline secret variable |
| **Blob storage auth** | Use `--auth-mode login` (service principal) — never storage account keys in pipelines |
| **Service connection** | Use Workload Identity Federation (OIDC) instead of client secret when possible |
| **Sensitive content** | Add a pipeline step to scan for secrets/PII before upload (e.g., `detect-secrets` or `gitleaks`) |

---

## Checklist

- [ ] Identify wiki type (Provisioned or Published) and note the Git repo name
- [ ] Create Azure DevOps service connection to the Azure subscription
- [ ] Grant the service principal `Storage Blob Data Contributor` on the storage account
- [ ] Create the sync pipeline YAML in your main repo
- [ ] Configure the wiki repo as a pipeline resource with trigger on `wikiMain`
- [ ] Run a full sync to seed the blob container
- [ ] Verify the search indexer picks up the uploaded files
- [ ] Set up the weekly scheduled full sync as a safety net
- [ ] (Optional) Sync `.attachments/` if images are needed
- [ ] (Optional) Add secret scanning step before upload
