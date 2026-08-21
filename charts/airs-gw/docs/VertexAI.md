# Google Vertex AI Workload Identity Configuration

This document provides a guide for configuring AIRS Gateway to call Google Vertex AI (including Anthropic Claude on Vertex) using Google Workload Identity instead of a static service account key.

## Overview

To use Vertex AI with AIRS Gateway, the gateway needs a Google OAuth 2 access token. You can supply a service account JSON key on the integration, but Workload Identity is the more secure and recommended approach — no long-lived credential is stored in the control plane or in the cluster.

Workload Identity requires **two** settings, one on each side. Configuring only one of them is the most common cause of Vertex authentication failures:

| Where | Setting | Value |
|-------|---------|-------|
| Integration / Virtual Key (control plane) | Vertex auth type | `workload` |
| Gateway deployment (`values.yaml`) | `environment.data.GCP_AUTH_MODE` | `workload` |

The gateway obtains the token in one of two ways depending on where it runs:

- **On GKE** — from the GKE metadata server, using the identity bound to its pod (`GCP_WIF_AUDIENCE` unset).
- **Off GKE** (EKS, or any non-GCP host) — via Workload Identity Federation, exchanging the platform's native identity for a Google token (`GCP_WIF_AUDIENCE` set). No metadata server, no static key.

## Step 1: Create the Vertex AI Service Account

Create a service account and grant it the Vertex AI user role on the project that serves your models:

```bash
gcloud iam service-accounts create <GSA> --project <GSA_PROJECT_ID>

gcloud projects add-iam-policy-binding <MODEL_PROJECT_ID> \
  --member "serviceAccount:<GSA>@<GSA_PROJECT_ID>.iam.gserviceaccount.com" \
  --role roles/aiplatform.user
```

The project that owns the models need not be the project the cluster runs in — grant `roles/aiplatform.user` on whichever project serves the models. (For direct resource access, off GKE, you can skip the service account and grant the federated principal instead — see Step 2, Option B.)

## Step 2: Bind the Workload Identity

Choose the option matching where the gateway runs.

### Option A: GKE (metadata server)

Allow the gateway's Kubernetes service account to impersonate the GSA, then annotate it:

```bash
gcloud iam service-accounts add-iam-policy-binding \
  <GSA>@<GSA_PROJECT_ID>.iam.gserviceaccount.com \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:<CLUSTER_PROJECT_ID>.svc.id.goog[<NAMESPACE>/<KSA>]"
```

```yaml
serviceAccount:
  create: true
  automount: true
  name: <KSA>
  annotations:
    iam.gke.io/gcp-service-account: <GSA>@<GSA_PROJECT_ID>.iam.gserviceaccount.com
```

Leave `GCP_WIF_AUDIENCE` unset — the gateway reads the GKE metadata server.

### Option B: Off GKE — Workload Identity Federation

Federate the platform's native identity (e.g. an AWS IAM role on EKS/EC2) into Google. Create a workload identity pool and provider that trusts your platform; the AWS example:

```bash
gcloud iam workload-identity-pools create <POOL_ID> \
  --project <PROJECT_ID> --location global

gcloud iam workload-identity-pools providers create-aws <PROVIDER_ID> \
  --project <PROJECT_ID> --location global \
  --workload-identity-pool <POOL_ID> \
  --account-id <AWS_ACCOUNT_ID> \
  --attribute-mapping \
'google.subject=assertion.arn,attribute.aws_account=assertion.account,attribute.aws_role=assertion.arn.extract('"'"'assumed-role/{role}/'"'"')'
```

> **Note:** the `attribute.aws_role` mapping has **no leading slash** before `assumed-role`. GetCallerIdentity ARNs are `arn:aws:sts::<acct>:assumed-role/<role>/<session>` — a colon precedes `assumed-role`, so a `/assumed-role/{role}/` pattern matches nothing and every `attribute.aws_role` binding fails (see Troubleshooting).

Then grant the federated identity access to Vertex, using **one** of the following.

**Direct resource access** — grant the model role to the federated principal on each model project:

```bash
gcloud projects add-iam-policy-binding <MODEL_PROJECT_ID> \
  --role roles/aiplatform.user \
  --member "principalSet://iam.googleapis.com/projects/<PROJECT_NUMBER>/locations/global/workloadIdentityPools/<POOL_ID>/attribute.aws_role/<AWS_ROLE_NAME>"
```

**Service account impersonation** (recommended) — let the federated principal impersonate the GSA from Step 1:

```bash
gcloud iam service-accounts add-iam-policy-binding \
  <GSA>@<GSA_PROJECT_ID>.iam.gserviceaccount.com \
  --role roles/iam.serviceAccountTokenCreator \
  --member "principalSet://iam.googleapis.com/projects/<PROJECT_NUMBER>/locations/global/workloadIdentityPools/<POOL_ID>/attribute.aws_role/<AWS_ROLE_NAME>"
```

Impersonation is usually preferable for enterprises: a named, single-purpose, revocable identity whose Vertex grant lives on the service account (grantable across many model projects), rather than binding an external principal into each project's IAM. Scope the member to a specific role (`attribute.aws_role/<AWS_ROLE_NAME>`), not the whole account.

## Step 3: Enable Workload Auth on the Gateway

```yaml
environment:
  data:
    GCP_AUTH_MODE: workload
    # Off GKE only — the federation audience (pool/provider):
    GCP_WIF_AUDIENCE: "//iam.googleapis.com/projects/<PROJECT_NUMBER>/locations/global/workloadIdentityPools/<POOL_ID>/providers/<PROVIDER_ID>"
    # Optional — set to impersonate the GSA; omit for direct resource access:
    GCP_WIF_SERVICE_ACCOUNT_EMAIL: "<GSA>@<GSA_PROJECT_ID>.iam.gserviceaccount.com"
```

| Variable | Effect |
|----------|--------|
| `GCP_AUTH_MODE=workload` | permits the workload-identity token path (required) |
| `GCP_WIF_AUDIENCE` | off-GKE federation audience; when unset the GKE metadata server is used |
| `GCP_WIF_SERVICE_ACCOUNT_EMAIL` | when set, the gateway impersonates this SA after federating; when empty it accesses Vertex directly. Gateway-wide (not per-request). |

The gateway builds its own credential — it does **not** read `GOOGLE_APPLICATION_CREDENTIALS`. Changing `environment.data` rolls the gateway pods.

## Step 4: Configure the Integration

On the Vertex AI integration (or Virtual Key) in the control plane, set the auth type to `workload` and provide the Vertex project ID and region. Leave the service account JSON field empty — if a JSON key is present it takes precedence and the workload-identity path is skipped.

## Step 5: Verify

Send a request through the gateway routed at your Vertex integration:

```bash
curl https://<GATEWAY_HOST>/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "x-portkey-api-key: <API_KEY>" \
  -H "x-portkey-provider: @<INTEGRATION_SLUG>" \
  -d '{
    "model": "anthropic.claude-sonnet-4-5@20250929",
    "max_tokens": 64,
    "messages": [{"role": "user", "content": "hello"}]
  }'
```

## Additional configurations

### Multiple Vertex projects from one gateway

The Vertex project and region are per integration, so a single gateway can route to several projects — one integration per project, each its own billing/quota boundary. The gateway has a single federated identity (and, if used, a single impersonation service account), so grant that identity `roles/aiplatform.user` on each target project. Per-project *identity* isolation requires a separate gateway.

### Attributing usage per user

Enabling **"map metadata to vertex labels"** on the integration stamps request metadata onto the Vertex request as labels, surfaced in Cloud Monitoring (and, SKU dependent, the billing export). Label values must be lowercase, `[a-z0-9_-]`, ≤63 characters.

## Important Notes

- Both the integration auth type **and** `GCP_AUTH_MODE` must be set to `workload`. Setting only one produces an authentication error.
- `GCP_AUTH_MODE` is shared with the GCS log store. If you already set it for [Log Store](./LogStore.md) GCS access, Vertex uses the same value.
- Anthropic models on Vertex must be addressed with the publisher prefix, for example `anthropic.claude-sonnet-4-5@20250929`. Without the prefix the gateway resolves the request against `publishers/google` and Vertex returns a 404.
- `GCP_WIF_SERVICE_ACCOUNT_EMAIL` is read from the environment (gateway-wide); it cannot be varied per request.
- Changing `environment.data` rolls the gateway pods, since the values are rendered into the deployment.

## Troubleshooting

**`Request had invalid authentication credentials. Expected OAuth 2 access token, login cookie or other valid authentication credential.`**

The gateway sent no usable token. Check, in order:

1. `GCP_AUTH_MODE` is present in the running pod —
   `kubectl exec <pod> -- env | grep GCP_AUTH_MODE`. This is the most common cause: the integration is set to `workload`, but the variable was never added to `values.yaml`.
2. The integration's auth type is `workload` and its service account JSON field is empty.
3. On GKE, Workload Identity resolves from inside the pod:
   ```bash
   kubectl exec <pod> -- wget -qO- \
     --header "Metadata-Flavor: Google" \
     "http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/email"
   ```
   This should print the GSA, not the node's default compute service account.

**`Publisher model ... was not found or your project does not have access to it`**

Authentication succeeded and the request reached Vertex. Either the model name is missing its publisher prefix (see Important Notes), the model is not enabled in the target project, or it is not served in the configured region.

**`Permission 'aiplatform.endpoints.predict' denied` (direct) or `Permission 'iam.serviceAccounts.getAccessToken' denied` (impersonation) — persists past IAM propagation, while federation itself succeeds**

For federated (off-GKE) setups scoped to `attribute.aws_role`, the usual cause is the `attribute.aws_role` mapping having a **leading slash** (`/assumed-role/{role}/`), which resolves to an empty string so no role-scoped binding matches. Use `assertion.arn.extract('assumed-role/{role}/')` (no leading slash) and re-create/update the provider. (`attribute.aws_account` bindings are unaffected, which is why account-scoped grants can appear to work while role-scoped ones fail.) Otherwise confirm the direct `roles/aiplatform.user` grant (direct mode), or the `roles/iam.serviceAccountTokenCreator` grant on the GSA plus the GSA's own `roles/aiplatform.user` (impersonation mode).
