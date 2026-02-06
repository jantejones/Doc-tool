# Azure AI Search Integration Plan

## Overview

This document outlines the plan for integrating Azure AI Search with MCP (Model Context Protocol) tools to enable document indexing and semantic search capabilities for markdown documentation. It includes Terraform infrastructure-as-code, indexer skillset configuration, and best practices for production deployment.

---

## Prerequisites

### Terraform Provider Setup

All infrastructure is managed via the AzureRM Terraform provider. Pin the version to avoid breaking changes.

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.85.0"
    }
  }

  backend "azurerm" {
    resource_group_name  = "rg-terraform-state"
    storage_account_name = "stterraformstate"
    container_name       = "tfstate"
    key                  = "azure-search-mcp.tfstate"
  }
}

provider "azurerm" {
  features {}
}
```

### Shared Variables

Define variables used across all modules:

```hcl
variable "project_name" {
  description = "Project prefix for all resource names"
  type        = string
  default     = "docsearch"
}

variable "location" {
  description = "Azure region for all resources"
  type        = string
  default     = "eastus2"
}

variable "environment" {
  description = "Deployment environment (dev, staging, prod)"
  type        = string
  default     = "dev"
}

locals {
  resource_prefix = "${var.project_name}-${var.environment}"
  common_tags = {
    Project     = var.project_name
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}
```

### Resource Group

```hcl
resource "azurerm_resource_group" "main" {
  name     = "rg-${local.resource_prefix}"
  location = var.location
  tags     = local.common_tags
}
```

---

## Components

### 1. Azure AI Search

Azure AI Search (formerly Azure Cognitive Search) provides full-text search, semantic ranking, and vector search over indexed content. For markdown documentation, it handles parsing, chunking, enrichment, and query execution.

#### SKU Selection Guide

| SKU    | Indexes | Indexers | Skillsets | Storage  | Semantic Ranker | Use Case                  |
|--------|---------|----------|-----------|----------|-----------------|---------------------------|
| Free   | 3       | 3        | 3         | 50 MB    | No              | Prototyping only          |
| Basic  | 15      | 15       | 15        | 2 GB     | No              | Small doc sets (<2 GB)    |
| S1     | 50      | 50       | 50        | 25 GB    | Yes             | Production workloads      |
| S2     | 200     | 200      | 200       | 100 GB   | Yes             | Large-scale production    |

**Recommendation:** Use `standard` (S1) for production. It supports semantic ranker (required for semantic search), has enough capacity for most documentation sets, and balances cost with capability.

#### Terraform Configuration

```hcl
resource "azurerm_search_service" "main" {
  name                = "search-${local.resource_prefix}"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  sku                 = "standard"  # S1 — supports semantic ranker

  replica_count   = 1  # Increase for high availability (2+ for SLA)
  partition_count = 1  # Increase for more storage/throughput

  semantic_search_sku = "standard"  # Enables semantic ranking capabilities

  public_network_access_enabled = true  # Set false for private endpoint only

  identity {
    type = "SystemAssigned"  # Used for managed identity auth to Blob Storage
  }

  tags = local.common_tags
}
```

#### Key Configuration Notes

- **Semantic search SKU** must be set to `"standard"` or `"free"` to enable the semantic ranker. Without this, the `semantic_search` MCP tool cannot perform ranked retrieval.
- **System-assigned managed identity** allows the search service to authenticate to Blob Storage without storing keys. This is the recommended approach.
- For production, set `replica_count >= 2` to get the 99.9% read SLA.

---

### 2. Blob Storage

Azure Blob Storage serves as the data source for the indexer. Markdown files are uploaded to a dedicated container and the search indexer reads from it directly.

#### Terraform Configuration

```hcl
resource "azurerm_storage_account" "docs" {
  name                     = "st${replace(local.resource_prefix, "-", "")}"
  resource_group_name      = azurerm_resource_group.main.name
  location                 = azurerm_resource_group.main.location
  account_tier             = "Standard"
  account_replication_type = "LRS"   # Use GRS for production geo-redundancy
  account_kind             = "StorageV2"

  # Recommended: disable public blob access, use managed identity
  allow_nested_items_to_be_public = false

  blob_properties {
    # Enable soft delete so accidentally removed docs can be recovered
    delete_retention_policy {
      days = 7
    }
    container_delete_retention_policy {
      days = 7
    }
    # Enable change feed for incremental indexing detection
    change_feed_enabled = true
  }

  tags = local.common_tags
}

resource "azurerm_storage_container" "markdown_docs" {
  name                  = "markdown-docs"
  storage_account_id    = azurerm_storage_account.docs.id
  container_access_type = "private"
}
```

#### Role Assignment: Search Service -> Blob Storage

Grant the search service's managed identity read access to the blob container:

```hcl
resource "azurerm_role_assignment" "search_blob_reader" {
  scope                = azurerm_storage_account.docs.id
  role_definition_name = "Storage Blob Data Reader"
  principal_id         = azurerm_search_service.main.identity[0].principal_id
}
```

#### Blob Naming Convention for MD Files

Use a consistent path structure so the indexer can extract metadata from the path:

```
markdown-docs/
  ├── product-guides/
  │   ├── getting-started.md
  │   ├── installation.md
  │   └── configuration.md
  ├── api-reference/
  │   ├── endpoints.md
  │   └── authentication.md
  └── runbooks/
      ├── incident-response.md
      └── deployment.md
```

The folder structure becomes searchable metadata (category, subcategory) once the indexer field mappings are configured.

---

### 3. Indexer, Index, Data Source & Skillset

This is the core of the search pipeline. It consists of four tightly coupled resources:

1. **Data Source** — tells the indexer where to read from
2. **Index** — defines the schema of searchable fields
3. **Skillset** — defines AI enrichment steps applied during indexing
4. **Indexer** — orchestrates the pipeline: reads data, applies skillset, writes to index

#### 3a. Data Source Connection

The data source connects Azure AI Search to the Blob Storage container using managed identity authentication.

```json
{
  "name": "ds-markdown-docs",
  "type": "azureblob",
  "credentials": {
    "connectionString": "ResourceId=/subscriptions/<sub-id>/resourceGroups/rg-docsearch-dev/providers/Microsoft.Storage/storageAccounts/stdocsearchdev;"
  },
  "container": {
    "name": "markdown-docs",
    "query": null
  },
  "dataDeletionDetectionPolicy": {
    "@odata.type": "#Microsoft.Azure.Search.NativeBlobSoftDeleteDeletionDetectionPolicy"
  }
}
```

**Best Practice:** Use the `ResourceId=` connection string format for managed identity auth instead of account keys. This pairs with the role assignment defined above.

#### Terraform — Data Source via REST API

Azure AI Search sub-resources (data sources, indexes, skillsets, indexers) are not native Terraform resources. Use `azurerm_resource_deployment` or the `azapi_resource` provider, or provision them via a `null_resource` with the REST API:

```hcl
resource "null_resource" "search_datasource" {
  depends_on = [
    azurerm_search_service.main,
    azurerm_role_assignment.search_blob_reader
  ]

  provisioner "local-exec" {
    command = <<-EOT
      az rest --method PUT \
        --uri "https://${azurerm_search_service.main.name}.search.windows.net/datasources/ds-markdown-docs?api-version=2024-07-01" \
        --headers "Content-Type=application/json" "api-key=$(az search admin-key show \
          --resource-group ${azurerm_resource_group.main.name} \
          --service-name ${azurerm_search_service.main.name} \
          --query primaryKey -o tsv)" \
        --body '@${path.module}/search-configs/datasource.json'
    EOT
  }
}
```

#### 3b. Index Schema

The index defines the fields stored and how they are searchable. For markdown files, a good schema captures content, metadata, and vector embeddings.

```json
{
  "name": "idx-markdown-docs",
  "fields": [
    {
      "name": "id",
      "type": "Edm.String",
      "key": true,
      "filterable": true
    },
    {
      "name": "content",
      "type": "Edm.String",
      "searchable": true,
      "analyzer": "en.microsoft"
    },
    {
      "name": "title",
      "type": "Edm.String",
      "searchable": true,
      "filterable": true,
      "sortable": true,
      "analyzer": "en.microsoft"
    },
    {
      "name": "category",
      "type": "Edm.String",
      "searchable": true,
      "filterable": true,
      "facetable": true
    },
    {
      "name": "metadata_storage_path",
      "type": "Edm.String",
      "filterable": true,
      "sortable": true
    },
    {
      "name": "metadata_storage_last_modified",
      "type": "Edm.DateTimeOffset",
      "filterable": true,
      "sortable": true
    },
    {
      "name": "metadata_storage_name",
      "type": "Edm.String",
      "filterable": true
    },
    {
      "name": "keyphrases",
      "type": "Collection(Edm.String)",
      "searchable": true,
      "filterable": true,
      "facetable": true
    },
    {
      "name": "content_vector",
      "type": "Collection(Edm.Single)",
      "searchable": true,
      "dimensions": 1536,
      "vectorSearchProfile": "vector-profile-hnsw"
    }
  ],
  "vectorSearch": {
    "algorithms": [
      {
        "name": "hnsw-algorithm",
        "kind": "hnsw",
        "hnswParameters": {
          "metric": "cosine",
          "m": 4,
          "efConstruction": 400,
          "efSearch": 500
        }
      }
    ],
    "profiles": [
      {
        "name": "vector-profile-hnsw",
        "algorithm": "hnsw-algorithm",
        "vectorizer": "vectorizer-openai"
      }
    ],
    "vectorizers": [
      {
        "name": "vectorizer-openai",
        "kind": "azureOpenAI",
        "azureOpenAIParameters": {
          "resourceUri": "https://<your-openai-resource>.openai.azure.com",
          "deploymentId": "text-embedding-ada-002",
          "modelName": "text-embedding-ada-002",
          "apiKey": "<your-api-key>"
        }
      }
    ]
  },
  "semantic": {
    "configurations": [
      {
        "name": "semantic-config",
        "prioritizedFields": {
          "titleField": { "fieldName": "title" },
          "contentFields": [
            { "fieldName": "content" }
          ],
          "keywordsFields": [
            { "fieldName": "keyphrases" }
          ]
        }
      }
    ]
  }
}
```

#### Index Field Design — Best Practices for MD Files

| Field | Purpose | Why |
|-------|---------|-----|
| `content` | Full markdown body text | Primary search target for full-text and semantic queries |
| `title` | Extracted from first `# heading` | Enables title-based filtering and boosting in relevance |
| `category` | Derived from blob folder path | Enables faceted navigation (e.g., "show only runbooks") |
| `keyphrases` | AI-extracted key phrases | Improves recall and enables tag-based filtering |
| `content_vector` | 1536-dim embedding (ada-002) | Powers vector/hybrid search for semantic similarity |
| `metadata_storage_path` | Blob URI | Unique document identifier; used by `get_document` tool |
| `metadata_storage_last_modified` | Last modified timestamp | Enables date-range filtering in search queries |

#### 3c. Skillset Configuration

The skillset defines AI enrichment steps that run during indexing. For markdown files, the recommended skillset extracts key phrases, detects language, and generates vector embeddings.

```json
{
  "name": "ss-markdown-enrichment",
  "description": "Skillset for enriching markdown documentation during indexing",
  "skills": [
    {
      "@odata.type": "#Microsoft.Skills.Text.SplitSkill",
      "name": "split-skill",
      "description": "Split content into chunks for processing",
      "context": "/document",
      "inputs": [
        { "name": "text", "source": "/document/content" }
      ],
      "outputs": [
        { "name": "textItems", "targetName": "chunks" }
      ],
      "textSplitMode": "pages",
      "maximumPageLength": 2000,
      "pageOverlapLength": 200
    },
    {
      "@odata.type": "#Microsoft.Skills.Text.KeyPhraseExtractionSkill",
      "name": "keyphrase-skill",
      "description": "Extract key phrases from markdown content",
      "context": "/document",
      "inputs": [
        { "name": "text", "source": "/document/content" }
      ],
      "outputs": [
        { "name": "keyPhrases", "targetName": "keyphrases" }
      ],
      "defaultLanguageCode": "en",
      "maxKeyPhraseCount": 15
    },
    {
      "@odata.type": "#Microsoft.Skills.Text.LanguageDetectionSkill",
      "name": "language-detection-skill",
      "description": "Detect document language",
      "context": "/document",
      "inputs": [
        { "name": "text", "source": "/document/content" }
      ],
      "outputs": [
        { "name": "languageCode", "targetName": "language" }
      ]
    },
    {
      "@odata.type": "#Microsoft.Skills.Custom.WebApiSkill",
      "name": "markdown-title-extractor",
      "description": "Extract title from first H1 heading in markdown",
      "context": "/document",
      "uri": "https://func-${local.resource_prefix}.azurewebsites.net/api/extract-title",
      "httpMethod": "POST",
      "batchSize": 10,
      "inputs": [
        { "name": "text", "source": "/document/content" }
      ],
      "outputs": [
        { "name": "title", "targetName": "extracted_title" }
      ]
    },
    {
      "@odata.type": "#Microsoft.Skills.Text.AzureOpenAIEmbeddingSkill",
      "name": "embedding-skill",
      "description": "Generate vector embeddings for semantic search",
      "context": "/document",
      "resourceUri": "https://<your-openai-resource>.openai.azure.com",
      "deploymentId": "text-embedding-ada-002",
      "modelName": "text-embedding-ada-002",
      "inputs": [
        { "name": "text", "source": "/document/content" }
      ],
      "outputs": [
        { "name": "embedding", "targetName": "content_vector" }
      ]
    }
  ],
  "cognitiveServices": {
    "@odata.type": "#Microsoft.Azure.Search.CognitiveServicesByKey",
    "key": "<cognitive-services-key>"
  }
}
```

#### Skillset Selection Guide — Which Skills to Use for MD Files

| Skill | Required? | Purpose | Notes |
|-------|-----------|---------|-------|
| **SplitSkill** | Yes | Chunk large MD files into pages | Set `maximumPageLength` to 2000 chars with 200 overlap for context continuity |
| **KeyPhraseExtractionSkill** | Yes | Auto-tag documents with key terms | Enables faceted search; set `maxKeyPhraseCount` to 10-15 |
| **LanguageDetectionSkill** | Optional | Detect doc language | Useful if docs span multiple languages; drives analyzer selection |
| **AzureOpenAIEmbeddingSkill** | Yes | Generate vector embeddings | Required for semantic/vector search; use `text-embedding-ada-002` (1536 dims) |
| **Custom WebApiSkill** | Recommended | Extract title from `# heading` | Azure Search doesn't natively parse markdown headings; a small Azure Function handles this |
| **EntityRecognitionSkill** | Optional | Extract people, orgs, locations | Useful for knowledge base docs with named entities |
| **MergeSkill** | Conditional | Recombine chunked + enriched text | Needed if you split content and want to rejoin enriched chunks |

#### 3d. Indexer Configuration

The indexer ties everything together — it reads from the data source, applies the skillset, and writes to the index.

```json
{
  "name": "ixr-markdown-docs",
  "dataSourceName": "ds-markdown-docs",
  "targetIndexName": "idx-markdown-docs",
  "skillsetName": "ss-markdown-enrichment",
  "schedule": {
    "interval": "PT2H"
  },
  "parameters": {
    "batchSize": 10,
    "maxFailedItems": 5,
    "maxFailedItemsPerBatch": 2,
    "configuration": {
      "dataToExtract": "contentAndMetadata",
      "parsingMode": "text",
      "indexStorageMetadataOnlyForOversizedDocuments": true
    }
  },
  "fieldMappings": [
    {
      "sourceFieldName": "metadata_storage_path",
      "targetFieldName": "id",
      "mappingFunction": {
        "name": "base64Encode"
      }
    },
    {
      "sourceFieldName": "metadata_storage_path",
      "targetFieldName": "metadata_storage_path"
    },
    {
      "sourceFieldName": "metadata_storage_name",
      "targetFieldName": "metadata_storage_name"
    }
  ],
  "outputFieldMappings": [
    {
      "sourceFieldName": "/document/keyphrases",
      "targetFieldName": "keyphrases"
    },
    {
      "sourceFieldName": "/document/extracted_title",
      "targetFieldName": "title"
    },
    {
      "sourceFieldName": "/document/content_vector",
      "targetFieldName": "content_vector"
    }
  ]
}
```

#### Indexer Best Practices for MD Files

- **Parsing mode:** Use `"text"` for markdown files. The `"default"` mode tries to detect format, but `"text"` ensures the raw markdown content is ingested as-is, preserving headings and structure.
- **Schedule:** `PT2H` (every 2 hours) is a good default. For frequently updated docs, use `PT30M`. For static docs, `PT24H` or manual triggers.
- **Batch size:** 10 is safe. Increase to 50 if files are small (<10 KB each). Decrease to 1 if files are large (>1 MB) or if skillset processing is heavy.
- **Field mappings:** `base64Encode` on `metadata_storage_path` is required because blob paths contain `/` characters which are invalid in Azure Search document keys.
- **Change detection:** Enabled automatically with the blob change feed. Only modified blobs are re-indexed on subsequent runs.
- **Deletion detection:** The `NativeBlobSoftDeleteDeletionDetectionPolicy` on the data source ensures deleted blobs are removed from the index.

#### Terraform — Deploy All Search Sub-Resources

```hcl
# Azure Cognitive Services account (required for built-in skillset skills)
resource "azurerm_cognitive_account" "enrichment" {
  name                = "cog-${local.resource_prefix}"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  kind                = "CognitiveServices"
  sku_name            = "S0"
  tags                = local.common_tags
}

# Azure OpenAI for embeddings
resource "azurerm_cognitive_account" "openai" {
  name                = "oai-${local.resource_prefix}"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  kind                = "OpenAI"
  sku_name            = "S0"
  tags                = local.common_tags
}

resource "azurerm_cognitive_deployment" "embedding" {
  name                 = "text-embedding-ada-002"
  cognitive_account_id = azurerm_cognitive_account.openai.id

  model {
    format  = "OpenAI"
    name    = "text-embedding-ada-002"
    version = "2"
  }

  sku {
    name     = "Standard"
    capacity = 120  # Tokens per minute (in thousands)
  }
}

# Deploy search index, datasource, skillset, and indexer via REST API
resource "null_resource" "search_resources" {
  depends_on = [
    azurerm_search_service.main,
    azurerm_role_assignment.search_blob_reader,
    azurerm_cognitive_account.enrichment,
    azurerm_cognitive_deployment.embedding
  ]

  # Deploy in order: datasource -> index -> skillset -> indexer
  provisioner "local-exec" {
    command = <<-EOT
      SEARCH_URL="https://${azurerm_search_service.main.name}.search.windows.net"
      API_KEY=$(az search admin-key show \
        --resource-group ${azurerm_resource_group.main.name} \
        --service-name ${azurerm_search_service.main.name} \
        --query primaryKey -o tsv)
      API_VERSION="2024-07-01"

      # 1. Create Data Source
      az rest --method PUT \
        --uri "$SEARCH_URL/datasources/ds-markdown-docs?api-version=$API_VERSION" \
        --headers "Content-Type=application/json" "api-key=$API_KEY" \
        --body '@${path.module}/search-configs/datasource.json'

      # 2. Create Index
      az rest --method PUT \
        --uri "$SEARCH_URL/indexes/idx-markdown-docs?api-version=$API_VERSION" \
        --headers "Content-Type=application/json" "api-key=$API_KEY" \
        --body '@${path.module}/search-configs/index.json'

      # 3. Create Skillset
      az rest --method PUT \
        --uri "$SEARCH_URL/skillsets/ss-markdown-enrichment?api-version=$API_VERSION" \
        --headers "Content-Type=application/json" "api-key=$API_KEY" \
        --body '@${path.module}/search-configs/skillset.json'

      # 4. Create Indexer (must be last)
      az rest --method PUT \
        --uri "$SEARCH_URL/indexers/ixr-markdown-docs?api-version=$API_VERSION" \
        --headers "Content-Type=application/json" "api-key=$API_KEY" \
        --body '@${path.module}/search-configs/indexer.json'
    EOT
  }
}
```

---

### 4. MCP Logic App

The Azure Logic App acts as the MCP server, exposing the search functionality as MCP-compatible tool endpoints. It receives MCP tool calls, translates them to Azure AI Search REST API requests, and returns formatted results.

#### Terraform Configuration

```hcl
resource "azurerm_service_plan" "logic_app" {
  name                = "asp-${local.resource_prefix}"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  os_type             = "Windows"
  sku_name            = "WS1"  # Logic App Standard (Workflow Standard 1)
  tags                = local.common_tags
}

resource "azurerm_logic_app_standard" "mcp" {
  name                       = "logic-${local.resource_prefix}"
  resource_group_name        = azurerm_resource_group.main.name
  location                   = azurerm_resource_group.main.location
  app_service_plan_id        = azurerm_service_plan.logic_app.id
  storage_account_name       = azurerm_storage_account.docs.name
  storage_account_access_key = azurerm_storage_account.docs.primary_access_key

  identity {
    type = "SystemAssigned"
  }

  app_settings = {
    "SEARCH_SERVICE_NAME"   = azurerm_search_service.main.name
    "SEARCH_INDEX_NAME"     = "idx-markdown-docs"
    "SEARCH_API_VERSION"    = "2024-07-01"
    "SEMANTIC_CONFIG_NAME"  = "semantic-config"
  }

  site_config {
    dotnet_framework_version = "v6.0"
  }

  tags = local.common_tags
}

# Grant Logic App access to search service
resource "azurerm_role_assignment" "logic_app_search" {
  scope                = azurerm_search_service.main.id
  role_definition_name = "Search Index Data Reader"
  principal_id         = azurerm_logic_app_standard.mcp.identity[0].principal_id
}
```

#### Logic App Workflow — `get_document`

This workflow retrieves a specific document by its path or ID from the search index.

```json
{
  "definition": {
    "$schema": "https://schema.management.azure.com/providers/Microsoft.Logic/schemas/2016-06-01/workflowdefinition.json#",
    "contentVersion": "1.0.0.0",
    "triggers": {
      "manual": {
        "type": "Request",
        "kind": "Http",
        "inputs": {
          "schema": {
            "type": "object",
            "properties": {
              "document_id": { "type": "string" },
              "document_path": { "type": "string" }
            }
          }
        }
      }
    },
    "actions": {
      "Search_Document": {
        "type": "Http",
        "inputs": {
          "method": "POST",
          "uri": "https://@{parameters('searchServiceName')}.search.windows.net/indexes/@{parameters('indexName')}/docs/search?api-version=2024-07-01",
          "headers": {
            "Content-Type": "application/json",
            "api-key": "@{parameters('searchApiKey')}"
          },
          "body": {
            "search": "*",
            "filter": "id eq '@{triggerBody()?['document_id']}' or metadata_storage_path eq '@{triggerBody()?['document_path']}'",
            "select": "id,title,content,category,metadata_storage_path,metadata_storage_last_modified,keyphrases",
            "top": 1
          }
        }
      },
      "Return_Result": {
        "type": "Response",
        "inputs": {
          "statusCode": 200,
          "headers": { "Content-Type": "application/json" },
          "body": {
            "document": "@{body('Search_Document')?['value']?[0]}",
            "found": "@{greater(length(body('Search_Document')?['value']), 0)}"
          }
        },
        "runAfter": { "Search_Document": ["Succeeded"] }
      }
    }
  }
}
```

#### Logic App Workflow — `semantic_search`

This workflow performs semantic search with optional filters and returns ranked results.

```json
{
  "definition": {
    "$schema": "https://schema.management.azure.com/providers/Microsoft.Logic/schemas/2016-06-01/workflowdefinition.json#",
    "contentVersion": "1.0.0.0",
    "triggers": {
      "manual": {
        "type": "Request",
        "kind": "Http",
        "inputs": {
          "schema": {
            "type": "object",
            "properties": {
              "query": { "type": "string" },
              "top": { "type": "integer", "default": 5 },
              "filter_category": { "type": "string" },
              "filter_date_from": { "type": "string" },
              "filter_date_to": { "type": "string" }
            },
            "required": ["query"]
          }
        }
      }
    },
    "actions": {
      "Build_Filter": {
        "type": "Compose",
        "inputs": "@{if(not(empty(triggerBody()?['filter_category'])), concat('category eq ''', triggerBody()?['filter_category'], ''''), '')}@{if(and(not(empty(triggerBody()?['filter_category'])), not(empty(triggerBody()?['filter_date_from']))), ' and ', '')}@{if(not(empty(triggerBody()?['filter_date_from'])), concat('metadata_storage_last_modified ge ', triggerBody()?['filter_date_from']), '')}"
      },
      "Semantic_Search": {
        "type": "Http",
        "inputs": {
          "method": "POST",
          "uri": "https://@{parameters('searchServiceName')}.search.windows.net/indexes/@{parameters('indexName')}/docs/search?api-version=2024-07-01",
          "headers": {
            "Content-Type": "application/json",
            "api-key": "@{parameters('searchApiKey')}"
          },
          "body": {
            "search": "@{triggerBody()?['query']}",
            "queryType": "semantic",
            "semanticConfiguration": "semantic-config",
            "captions": "extractive",
            "answers": "extractive",
            "top": "@{coalesce(triggerBody()?['top'], 5)}",
            "filter": "@{outputs('Build_Filter')}",
            "select": "id,title,content,category,metadata_storage_path,keyphrases,metadata_storage_last_modified"
          }
        },
        "runAfter": { "Build_Filter": ["Succeeded"] }
      },
      "Format_Results": {
        "type": "Select",
        "inputs": {
          "from": "@body('Semantic_Search')?['value']",
          "select": {
            "title": "@item()?['title']",
            "content_snippet": "@take(item()?['content'], 500)",
            "category": "@item()?['category']",
            "path": "@item()?['metadata_storage_path']",
            "last_modified": "@item()?['metadata_storage_last_modified']",
            "keyphrases": "@item()?['keyphrases']",
            "score": "@item()?['@search.score']",
            "reranker_score": "@item()?['@search.rerankerScore']"
          }
        },
        "runAfter": { "Semantic_Search": ["Succeeded"] }
      },
      "Return_Results": {
        "type": "Response",
        "inputs": {
          "statusCode": 200,
          "headers": { "Content-Type": "application/json" },
          "body": {
            "results": "@{body('Format_Results')}",
            "count": "@{length(body('Semantic_Search')?['value'])}",
            "query": "@{triggerBody()?['query']}"
          }
        },
        "runAfter": { "Format_Results": ["Succeeded"] }
      }
    }
  }
}
```

---

### 5. MCP Tools

The two MCP tools are exposed via the Logic App HTTP triggers and follow the MCP tool specification.

#### Tool 1: `get_document`

Retrieves a specific document from the indexed collection by ID or blob path.

**MCP Tool Definition:**

```json
{
  "name": "get_document",
  "description": "Retrieve a specific markdown document from the indexed documentation collection. Use this when you need the full content of a known document.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "document_id": {
        "type": "string",
        "description": "The base64-encoded document ID (derived from blob storage path)"
      },
      "document_path": {
        "type": "string",
        "description": "The blob storage path of the document (e.g., 'product-guides/getting-started.md')"
      }
    },
    "oneOf": [
      { "required": ["document_id"] },
      { "required": ["document_path"] }
    ]
  }
}
```

**Example Request:**

```json
{
  "document_path": "product-guides/getting-started.md"
}
```

**Example Response:**

```json
{
  "document": {
    "id": "aHR0cHM6Ly9zdGRvY3NlYXJjaC5ibG9iLmNvcmUud2luZG93cy5uZXQvbWFya2Rvd24tZG9jcy9nZXR0aW5nLXN0YXJ0ZWQubWQ1",
    "title": "Getting Started",
    "content": "# Getting Started\n\nWelcome to the product guide...",
    "category": "product-guides",
    "metadata_storage_path": "https://stdocsearch.blob.core.windows.net/markdown-docs/product-guides/getting-started.md",
    "metadata_storage_last_modified": "2026-01-15T10:30:00Z",
    "keyphrases": ["getting started", "installation", "quickstart", "setup"]
  },
  "found": true
}
```

#### Tool 2: `semantic_search`

Performs semantic search across all indexed documents with optional filtering.

**MCP Tool Definition:**

```json
{
  "name": "semantic_search",
  "description": "Search across indexed markdown documentation using semantic understanding. Returns ranked results based on meaning, not just keyword matching. Use this to find relevant documents for a topic or question.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "query": {
        "type": "string",
        "description": "Natural language search query (e.g., 'how to configure authentication')"
      },
      "top": {
        "type": "integer",
        "description": "Number of results to return (1-50, default 5)",
        "default": 5,
        "minimum": 1,
        "maximum": 50
      },
      "filter_category": {
        "type": "string",
        "description": "Filter results to a specific document category (e.g., 'api-reference', 'runbooks')"
      },
      "filter_date_from": {
        "type": "string",
        "format": "date-time",
        "description": "Only include documents modified after this date (ISO 8601 format)"
      },
      "filter_date_to": {
        "type": "string",
        "format": "date-time",
        "description": "Only include documents modified before this date (ISO 8601 format)"
      }
    },
    "required": ["query"]
  }
}
```

**Example Request:**

```json
{
  "query": "how to handle incident response",
  "top": 3,
  "filter_category": "runbooks"
}
```

**Example Response:**

```json
{
  "results": [
    {
      "title": "Incident Response",
      "content_snippet": "# Incident Response\n\n## Overview\nThis runbook covers the standard incident response procedure...",
      "category": "runbooks",
      "path": "https://stdocsearch.blob.core.windows.net/markdown-docs/runbooks/incident-response.md",
      "last_modified": "2026-01-20T14:00:00Z",
      "keyphrases": ["incident response", "escalation", "severity levels", "on-call"],
      "score": 12.5,
      "reranker_score": 3.42
    },
    {
      "title": "Deployment",
      "content_snippet": "# Deployment Runbook\n\n## Rollback Procedures\nIf a deployment causes incidents...",
      "category": "runbooks",
      "path": "https://stdocsearch.blob.core.windows.net/markdown-docs/runbooks/deployment.md",
      "last_modified": "2026-01-18T09:00:00Z",
      "keyphrases": ["deployment", "rollback", "blue-green", "canary"],
      "score": 8.3,
      "reranker_score": 2.15
    }
  ],
  "count": 2,
  "query": "how to handle incident response"
}
```

---

## Architecture Flow

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────────┐
│                 │     │                  │     │                     │
│  Blob Storage   │────>│    Indexer        │────>│  Azure AI Search    │
│  (MD files)     │     │  + Skillset      │     │  (Index + Vectors)  │
│                 │     │                  │     │                     │
└─────────────────┘     └──────────────────┘     └──────────┬──────────┘
                                                            │
                                                            │ REST API
                                                            │
                                                 ┌──────────▼──────────┐
                                                 │                     │
                                                 │  MCP Logic App      │
                                                 │  (Tool Endpoints)   │
                                                 │                     │
                                                 └──────────┬──────────┘
                                                            │
                                                   ┌────────┴────────┐
                                                   │                 │
                                              get_document    semantic_search
```

### Step-by-Step Flow

1. **Upload:** Markdown files are uploaded to the `markdown-docs` container in Azure Blob Storage (manually, via CI/CD, or via a sync pipeline).
2. **Detection:** The indexer runs on schedule (`PT2H`) and detects new/modified blobs via the change feed.
3. **Enrichment:** The skillset processes each document:
   - Splits content into manageable chunks
   - Extracts key phrases for tagging
   - Detects language
   - Extracts the title from the first `#` heading
   - Generates a 1536-dimension vector embedding via Azure OpenAI
4. **Indexing:** Enriched fields are mapped to the index schema and stored in Azure AI Search.
5. **Querying:** The MCP Logic App receives tool calls and translates them to Search REST API requests:
   - `get_document` → point lookup by ID or path filter
   - `semantic_search` → semantic query with optional OData filters
6. **Response:** Results are formatted and returned to the MCP client.

---

## Deployment Order

Resources must be deployed in this order due to dependencies:

1. **Resource Group**
2. **Storage Account + Container** — no dependencies
3. **Azure AI Search Service** — no dependencies (parallel with #2)
4. **Cognitive Services Account** — no dependencies (parallel with #2, #3)
5. **Azure OpenAI + Embedding Deployment** — depends on #4
6. **Role Assignment (Search -> Blob)** — depends on #2, #3
7. **Search Data Source** — depends on #3, #6
8. **Search Index** — depends on #3
9. **Search Skillset** — depends on #3, #4, #5
10. **Search Indexer** — depends on #7, #8, #9 (must be last)
11. **Logic App** — depends on #3 (can run parallel with #7-#10)

---

## Monitoring & Observability

### Indexer Status Monitoring

```hcl
resource "azurerm_monitor_diagnostic_setting" "search" {
  name                       = "diag-search-${local.resource_prefix}"
  target_resource_id         = azurerm_search_service.main.id
  log_analytics_workspace_id = azurerm_log_analytics_workspace.main.id

  enabled_log {
    category = "OperationLogs"
  }

  metric {
    category = "AllMetrics"
  }
}

resource "azurerm_log_analytics_workspace" "main" {
  name                = "law-${local.resource_prefix}"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  sku                 = "PerGB2018"
  retention_in_days   = 30
  tags                = local.common_tags
}
```

### Alert on Indexer Failures

```hcl
resource "azurerm_monitor_metric_alert" "indexer_failures" {
  name                = "alert-indexer-failures-${local.resource_prefix}"
  resource_group_name = azurerm_resource_group.main.name
  scopes              = [azurerm_search_service.main.id]
  description         = "Alert when the search indexer has failed documents"
  severity            = 2
  frequency           = "PT15M"
  window_size         = "PT1H"

  criteria {
    metric_namespace = "Microsoft.Search/searchServices"
    metric_name      = "DocumentsProcessedCount"
    aggregation      = "Total"
    operator         = "LessThan"
    threshold        = 1
  }
}
```

---

## Security Best Practices

| Area | Recommendation | Implementation |
|------|---------------|----------------|
| **Authentication** | Use managed identity everywhere | System-assigned identity on Search + Logic App |
| **Network** | Restrict public access | Use private endpoints for Search and Storage in production |
| **Keys** | Avoid hardcoded API keys | Store in Azure Key Vault; reference via Logic App parameters |
| **RBAC** | Least privilege | `Storage Blob Data Reader` for search, `Search Index Data Reader` for Logic App |
| **Encryption** | Enable at rest and in transit | Enabled by default on all Azure services; optionally add CMK |
| **Logging** | Enable diagnostic settings | Send to Log Analytics for audit and troubleshooting |

### Key Vault Integration for API Keys

```hcl
resource "azurerm_key_vault" "main" {
  name                = "kv-${local.resource_prefix}"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  sku_name            = "standard"
  tenant_id           = data.azurerm_client_config.current.tenant_id

  tags = local.common_tags
}

resource "azurerm_key_vault_secret" "search_api_key" {
  name         = "search-api-key"
  value        = azurerm_search_service.main.primary_key
  key_vault_id = azurerm_key_vault.main.id
}

data "azurerm_client_config" "current" {}
```

---

## Cost Estimation (Monthly)

| Resource | SKU | Estimated Cost |
|----------|-----|---------------|
| Azure AI Search | S1 (1 replica, 1 partition) | ~$250/mo |
| Storage Account | Standard LRS | ~$5/mo (for small doc sets) |
| Logic App Standard | WS1 | ~$150/mo |
| Cognitive Services | S0 | ~$1/1K transactions |
| Azure OpenAI (ada-002) | Standard | ~$0.0001/1K tokens |
| Log Analytics | PerGB2018 | ~$2.76/GB ingested |
| **Total (dev)** | | **~$410/mo** |

Use the `Free` search SKU and `Consumption` Logic App plan for development/testing to reduce costs to under $50/mo (note: Free SKU does not support semantic ranker).

---

## Benefits

- **Efficient document retrieval and search** — sub-second query response with pre-built index
- **Semantic understanding of queries** — goes beyond keyword matching using vector embeddings and the semantic ranker
- **Scalable storage and indexing** — blob storage handles any volume; indexer scales with partitions
- **Integration with MCP ecosystem** — standard MCP tool interface for any MCP-compatible client
- **AI-powered search capabilities** — key phrase extraction, vector search, and extractive answers
- **Infrastructure as Code** — fully reproducible with Terraform; no manual portal clicks
- **Security by default** — managed identity, RBAC, Key Vault, and private endpoints
