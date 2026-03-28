# radshow-apim — APIOps Configuration

Azure API Management configuration for the RAD Showcase POC, managed via the [Azure APIOps Toolkit](https://github.com/Azure/apiops).

## Structure

```
radshow-apim/
├── configuration.dev.yaml           # DEV environment overrides
├── configuration.stg.yaml           # STG environment overrides (dual-region)
├── configuration.prd.yaml           # PRD environment overrides (dual-region)
├── apimartifacts/
│   ├── apis/
│   │   ├── radshow-product-api/     # Product CRUD API
│   │   │   ├── apiInformation.json
│   │   │   ├── specification.yaml   # OpenAPI 3.0
│   │   │   └── policy.xml
│   │   └── radshow-status-api/      # Status + Failover API
│   │       ├── apiInformation.json
│   │       ├── specification.yaml
│   │       └── policy.xml
│   ├── backends/                    # Backend service definitions
│   │   ├── functions-primary.json
│   │   ├── functions-secondary.json
│   │   ├── aca-primary.json
│   │   └── aca-secondary.json
│   ├── named-values/                # APIM named values (env-specific via config YAML)
│   │   └── named-values.json
│   ├── policy-fragments/            # Reusable policy fragments
│   │   ├── cors-policy.xml
│   │   └── auth-policy.xml
│   └── products/
│       └── unlimited.json
└── .github/workflows/
    ├── extractor.yml                # Extract config from running APIM instance
    └── publisher.yml                # Publish artifacts to DEV → STG → PRD
```

## APIs

| API | Path | Description |
|-----|------|-------------|
| **radshow-product-api** | `/products` | CRUD operations for product inventory |
| **radshow-status-api** | `/ops` | Regional health, healthz probe, and DR failover |

## Layered Approach

Infrastructure (APIM instance, networking, SKU, multi-region gateway) is managed by **Terraform** in `radshow-def` / `radshow-lic`.

API configuration (definitions, policies, products, named values) is managed here via **APIOps** — the recommended separation per Microsoft Well-Architected guidance.

## Pipelines

### Publisher (`publisher.yml`)
Triggers on push to `main` when `apimartifacts/**` or `configuration.*.yaml` change. Deploys sequentially: DEV → STG (requires approval) → PRD (requires approval).

### Extractor (`extractor.yml`)
Manual workflow dispatch. Extracts current configuration from a running APIM instance into `apimartifacts/` and opens a PR for review.

## GitHub Environments Setup

Create three GitHub environments (`dev`, `stg`, `prd`) with these secrets:

| Secret | Description |
|--------|-------------|
| `AZURE_CLIENT_ID` | Service principal client ID |
| `AZURE_CLIENT_SECRET` | Service principal client secret |
| `AZURE_TENANT_ID` | Azure AD tenant ID |
| `AZURE_SUBSCRIPTION_ID` | Azure subscription ID |

STG and PRD environments should have **required reviewers** configured for deployment approval gates.

## DR Behavior

During failover, no APIM configuration change is needed:
- APIM Premium multi-region gateway is deployed to both SCUS and NCUS
- Front Door routes traffic to the healthy regional gateway
- Each regional gateway routes to its local Functions/ACA backend via the `set-backend-service` policy using named values
