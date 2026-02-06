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

## Scope — MD Files Only

This pipeline syncs **only `.md` files**. The following wiki artifacts are explicitly excluded:

| Artifact | Excluded? | Reason |
|----------|-----------|--------|
| `*.md` files | No — these are synced | Documentation content for the search index |
| `.order` files | Yes | Wiki sidebar ordering only; not searchable content |
| `.attachments/` folder | Yes | Images/binaries; not indexable as text by Azure AI Search |
| `README.md` | Yes | Git convention file; not a wiki page |

---

## How Create / Update / Delete Are Handled

Every wiki edit is a Git commit. The pipeline detects the type of change and maps it to the correct blob storage operation:

| Wiki Action | Git Effect | Pipeline Detection | Blob Storage Operation | Search Index Effect |
|-------------|-----------|-------------------|----------------------|-------------------|
| **Create** a new page | New `.md` file added | File in diff + exists on disk | `az storage blob upload` (new blob) | Indexer adds new document on next run |
| **Update** an existing page | `.md` file modified | File in diff + exists on disk | `az storage blob upload --overwrite` | Indexer re-processes document (change feed detects modified blob) |
| **Delete** a page | `.md` file removed | File in diff + **not** on disk | `az storage blob delete` | Indexer removes document (soft-delete detection policy) |
| **Rename** a page | Old file deleted + new file added | Two entries in diff (one exists, one doesn't) | Delete old blob + upload new blob | Old doc removed, new doc added |
| **Move** page to subfolder | Path changes in git | Same as rename — old path deleted, new path added | Delete old blob + upload new blob | Category field updates (derived from folder path) |

**Full sync** handles all of these by comparing the wiki snapshot to what's in blob storage and reconciling (upload missing, delete orphaned). **Incremental sync** handles them by inspecting `git diff` between the last two commits.

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
                                              ┌───────┴───────┐
                                              │               │
                                          Upload new/     Delete orphaned
                                          modified .md    blobs from
                                          files           storage
                                              │               │
                                              └───────┬───────┘
                                                      │
                                          ┌───────────▼──────────┐
                                          │                      │
                                          │  Blob Storage        │
                                          │  (markdown-docs)     │
                                          │  .md files only      │
                                          │                      │
                                          └───────────┬──────────┘
                                                      │
                                                      │ Indexer detects
                                                      │ changes via
                                                      │ change feed
                                                      ▼
                                          ┌──────────────────────┐
                                          │  Azure AI Search     │
                                          │  (Index + Vectors)   │
                                          └──────────────────────┘
```

### Two Sync Strategies

| Strategy | Trigger | Creates | Updates | Deletes | Pros | Cons |
|----------|---------|---------|---------|---------|------|------|
| **Full sync** | Scheduled (weekly) or manual | Yes | Yes | Yes (orphan cleanup) | Guarantees consistency; catches any missed incremental runs | Slower; re-uploads unchanged files |
| **Incremental sync** | Wiki push trigger | Yes | Yes | Yes (via git diff) | Fast; only touches changed files | Requires `fetchDepth: 2`; may miss multi-commit pushes |

**Recommendation:** Use incremental sync for every wiki push, with a weekly full sync as a safety net to catch any drift.

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

This pipeline clones the wiki repo, filters to only `.md` files, uploads them to blob storage, and **deletes orphaned blobs** that no longer have a matching wiki page. This guarantees blob storage is an exact mirror of the wiki's `.md` files.

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
              MANIFEST="$(Build.ArtifactStagingDirectory)/wiki-manifest.txt"
              mkdir -p "$STAGING_DIR"

              # Copy .md files preserving folder structure
              cd "$WIKI_DIR"
              find . -name "*.md" \
                -not -path "./.attachments/*" \
                -not -name "README.md" \
                | while read -r file; do
                    # Strip leading ./
                    clean_path="${file#./}"
                    dest_dir="$STAGING_DIR/$(dirname "$clean_path")"
                    mkdir -p "$dest_dir"
                    cp "$file" "$dest_dir/"
                    # Record in manifest for orphan detection
                    echo "$clean_path" >> "$MANIFEST"
                  done

              # Show what will be uploaded
              echo "=== Files staged for upload ==="
              sort "$MANIFEST"
              echo "Total: $(wc -l < "$MANIFEST") files"
            displayName: "Filter and stage MD files"

          # 3. Upload to blob storage (handles creates + updates)
          - task: AzureCLI@2
            displayName: "Upload MD files to Blob Storage"
            inputs:
              azureSubscription: $(azureSubscription)
              scriptType: "bash"
              scriptLocation: "inlineScript"
              inlineScript: |
                STAGING_DIR="$(Build.ArtifactStagingDirectory)/wiki-md"

                echo "=== Uploading new and updated MD files ==="
                az storage blob upload-batch \
                  --account-name "$(storageAccountName)" \
                  --destination "$(containerName)" \
                  --source "$STAGING_DIR" \
                  --overwrite true \
                  --content-type "text/markdown" \
                  --auth-mode login \
                  --pattern "*.md"

          # 4. Delete orphaned blobs (handles deletes)
          #    Compare blobs in storage against the wiki manifest.
          #    Any blob that exists in storage but NOT in the wiki is orphaned.
          - task: AzureCLI@2
            displayName: "Delete orphaned blobs (wiki pages that were removed)"
            inputs:
              azureSubscription: $(azureSubscription)
              scriptType: "bash"
              scriptLocation: "inlineScript"
              inlineScript: |
                MANIFEST="$(Build.ArtifactStagingDirectory)/wiki-manifest.txt"

                echo "=== Checking for orphaned blobs ==="

                # List all .md blobs currently in the container
                az storage blob list \
                  --account-name "$(storageAccountName)" \
                  --container-name "$(containerName)" \
                  --auth-mode login \
                  --query "[?ends_with(name, '.md')].name" \
                  --output tsv > /tmp/blob-list.txt

                DELETED_COUNT=0

                # Compare: delete any blob not present in the wiki manifest
                while read -r blob_name; do
                  if ! grep -qxF "$blob_name" "$MANIFEST"; then
                    echo "  [DELETE] $blob_name (no longer in wiki)"
                    az storage blob delete \
                      --account-name "$(storageAccountName)" \
                      --container-name "$(containerName)" \
                      --name "$blob_name" \
                      --auth-mode login
                    DELETED_COUNT=$((DELETED_COUNT + 1))
                  fi
                done < /tmp/blob-list.txt

                echo "=== Orphan cleanup complete: $DELETED_COUNT blob(s) deleted ==="

          # 5. Summary
          - task: AzureCLI@2
            displayName: "Print final blob inventory"
            inputs:
              azureSubscription: $(azureSubscription)
              scriptType: "bash"
              scriptLocation: "inlineScript"
              inlineScript: |
                echo "=== Current blob storage contents ==="
                az storage blob list \
                  --account-name "$(storageAccountName)" \
                  --container-name "$(containerName)" \
                  --auth-mode login \
                  --output table \
                  --query "[].{Name:name, Size:properties.contentLength, Modified:properties.lastModified}"
```

---

## Azure DevOps Pipeline — Incremental Sync

This pipeline triggers on every wiki push and only processes the `.md` files that changed. It uses `git diff --diff-filter` to reliably separate creates, updates, renames, and deletes.

### How `git diff --diff-filter` Maps to Operations

| Git Status | Filter Flag | Wiki Action | Pipeline Action |
|-----------|-------------|-------------|-----------------|
| `A` (Added) | `--diff-filter=A` | New page created | Upload new blob |
| `M` (Modified) | `--diff-filter=M` | Page content edited | Upload blob with `--overwrite` |
| `R` (Renamed) | `--diff-filter=R` | Page renamed or moved to another folder | Delete old blob path + upload new blob path |
| `D` (Deleted) | `--diff-filter=D` | Page deleted | Delete blob |

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

          # 1. Detect added, modified, renamed, and deleted .md files
          - script: |
              echo "=== Detecting changed .md files ==="
              WIKI_DIR="$(Build.SourcesDirectory)"
              STAGING_DIR="$(Build.ArtifactStagingDirectory)/wiki-md"
              DELETED_FILE="$(Build.ArtifactStagingDirectory)/deleted-blobs.txt"
              mkdir -p "$STAGING_DIR"
              touch "$DELETED_FILE"

              cd "$WIKI_DIR"

              # Check if we have a previous commit to diff against
              if ! git rev-parse HEAD~1 >/dev/null 2>&1; then
                echo "First commit — falling back to full sync"
                find . -name "*.md" \
                  -not -path "./.attachments/*" \
                  -not -name "README.md" \
                  | while read -r file; do
                      clean_path="${file#./}"
                      dest_dir="$STAGING_DIR/$(dirname "$clean_path")"
                      mkdir -p "$dest_dir"
                      cp "$file" "$dest_dir/"
                      echo "  [UPLOAD] $clean_path"
                    done
              else
                # --- ADDED + MODIFIED: stage for upload ---
                git diff --name-only --diff-filter=AM HEAD~1 HEAD \
                  -- '*.md' ':!.attachments/' ':!README.md' \
                  | while read -r file; do
                      echo "  [UPLOAD] $file"
                      dest_dir="$STAGING_DIR/$(dirname "$file")"
                      mkdir -p "$dest_dir"
                      cp "$file" "$dest_dir/"
                    done

                # --- RENAMED: upload new path, delete old path ---
                # git diff -M shows renames as: old_path -> new_path
                git diff --name-status --diff-filter=R -M HEAD~1 HEAD \
                  -- '*.md' ':!.attachments/' ':!README.md' \
                  | while read -r status old_path new_path; do
                      echo "  [RENAME] $old_path -> $new_path"
                      # Stage new file for upload
                      dest_dir="$STAGING_DIR/$(dirname "$new_path")"
                      mkdir -p "$dest_dir"
                      cp "$new_path" "$dest_dir/"
                      # Mark old path for deletion
                      echo "$old_path" >> "$DELETED_FILE"
                    done

                # --- DELETED: mark for blob deletion ---
                git diff --name-only --diff-filter=D HEAD~1 HEAD \
                  -- '*.md' ':!.attachments/' ':!README.md' \
                  | while read -r file; do
                      echo "  [DELETE] $file"
                      echo "$file" >> "$DELETED_FILE"
                    done
              fi

              UPLOAD_COUNT=$(find "$STAGING_DIR" -name "*.md" 2>/dev/null | wc -l)
              DELETE_COUNT=$(wc -l < "$DELETED_FILE" 2>/dev/null || echo 0)
              echo ""
              echo "=== Summary: $UPLOAD_COUNT to upload, $DELETE_COUNT to delete ==="

              # Set pipeline variables for conditional steps
              echo "##vso[task.setvariable variable=hasUploads]$([ "$UPLOAD_COUNT" -gt 0 ] && echo true || echo false)"
              echo "##vso[task.setvariable variable=hasDeletes]$([ "$DELETE_COUNT" -gt 0 ] && echo true || echo false)"
            displayName: "Detect changed MD files"

          # 2. Upload new and modified .md files (creates + updates)
          - task: AzureCLI@2
            displayName: "Upload new/modified MD files"
            condition: eq(variables['hasUploads'], 'true')
            inputs:
              azureSubscription: $(azureSubscription)
              scriptType: "bash"
              scriptLocation: "inlineScript"
              inlineScript: |
                STAGING_DIR="$(Build.ArtifactStagingDirectory)/wiki-md"
                echo "=== Uploading changed MD files ==="
                az storage blob upload-batch \
                  --account-name "$(storageAccountName)" \
                  --destination "$(containerName)" \
                  --source "$STAGING_DIR" \
                  --overwrite true \
                  --content-type "text/markdown" \
                  --auth-mode login \
                  --pattern "*.md"

          # 3. Delete blobs for removed/renamed wiki pages (deletes)
          - task: AzureCLI@2
            displayName: "Delete removed MD files from blob"
            condition: eq(variables['hasDeletes'], 'true')
            inputs:
              azureSubscription: $(azureSubscription)
              scriptType: "bash"
              scriptLocation: "inlineScript"
              inlineScript: |
                DELETED_FILE="$(Build.ArtifactStagingDirectory)/deleted-blobs.txt"
                echo "=== Deleting removed blobs ==="
                while read -r blob_name; do
                  [ -z "$blob_name" ] && continue
                  echo "  Deleting: $blob_name"
                  az storage blob delete \
                    --account-name "$(storageAccountName)" \
                    --container-name "$(containerName)" \
                    --name "$blob_name" \
                    --auth-mode login \
                    --delete-snapshots include \
                    || echo "  Warning: $blob_name may not exist in blob storage"
                done < "$DELETED_FILE"
```

---

## Alternative: AzCopy for Large Wiki Repos

For wikis with hundreds of `.md` pages, `az storage blob upload-batch` can be slow. **AzCopy** is faster and natively handles creates, updates, **and deletes** in a single command via `--delete-destination=true`.

```yaml
# Can replace both the upload-batch and orphan-delete steps in either pipeline
- script: |
    STAGING_DIR="$(Build.ArtifactStagingDirectory)/wiki-md"

    # azcopy sync handles all three operations:
    #   - Creates:  uploads .md files not yet in blob storage
    #   - Updates:  re-uploads .md files where MD5 hash changed
    #   - Deletes:  removes blobs that have no matching local .md file
    azcopy sync \
      "$STAGING_DIR" \
      "https://$(storageAccountName).blob.core.windows.net/$(containerName)" \
      --delete-destination=true \
      --include-pattern="*.md" \
      --put-md5 \
      --log-level=WARNING
  displayName: "AzCopy sync MD files to blob storage (create + update + delete)"
  env:
    AZCOPY_AUTO_LOGIN_TYPE: "SPN"
    AZCOPY_SPA_CLIENT_SECRET: $(servicePrincipalKey)
    AZCOPY_SPA_APPLICATION_ID: $(servicePrincipalId)
    AZCOPY_TENANT_ID: $(tenantId)
```

**AzCopy `sync` vs `upload-batch` + manual delete:**

| Feature | `az storage blob upload-batch` + manual delete | `azcopy sync --delete-destination` |
|---------|------------------------------------------------|-------------------------------------|
| Creates (new .md) | Yes | Yes |
| Updates (modified .md) | Yes (`--overwrite`) | Yes (MD5 hash comparison) |
| Deletes (removed .md) | Manual (must list blobs + compare) | Automatic (`--delete-destination=true`) |
| Differential upload | No (re-uploads everything) | Yes (skips unchanged files) |
| Parallelism | Limited | High (auto-tuned) |
| Speed (500+ .md files) | Slow | Fast |
| Availability on agents | Built-in (Azure CLI) | Built-in on hosted agents |

---

## End-to-End Flow

```
 Developer creates / edits / deletes / renames a wiki page
                    │
                    ▼
 Git commit pushed to <project>.wiki repo (wikiMain branch)
                    │
                    ▼
 Pipeline triggered (resource trigger on wiki repo)
                    │
                    ▼
 Agent clones wiki repo (shallow, fetchDepth: 2)
                    │
                    ▼
 git diff detects which .md files were added, modified, renamed, or deleted
                    │
                    ├── Added / Modified .md  ──>  Upload blob (--overwrite)
                    ├── Renamed .md           ──>  Delete old blob + upload new blob
                    └── Deleted .md           ──>  Delete blob
                    │
                    ▼
 Blob Storage now mirrors the wiki's .md files exactly
                    │
                    ▼
 Search indexer detects changes via blob change feed (next scheduled run)
   ├── New/modified blobs  ──>  Re-index document (skillset: split, keyphrase, embed)
   └── Soft-deleted blobs  ──>  Remove document from search index
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
- [ ] Create the incremental sync pipeline YAML in your main repo
- [ ] Configure the wiki repo as a pipeline resource with trigger on `wikiMain`
- [ ] Run a full sync to seed the blob container with all existing `.md` files
- [ ] Verify the search indexer picks up the uploaded `.md` files
- [ ] Test update: edit a wiki page, confirm blob is overwritten and re-indexed
- [ ] Test delete: remove a wiki page, confirm blob is deleted and removed from index
- [ ] Test rename: rename a wiki page, confirm old blob deleted + new blob created
- [ ] Set up the weekly scheduled full sync as a safety net for orphan cleanup
- [ ] (Optional) Add secret scanning step before upload
