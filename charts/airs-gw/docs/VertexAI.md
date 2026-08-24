# Google Vertex AI Workload Identity Configuration

This document describes how to configure AIRS Gateway to call Google Vertex AI
using GKE Workload Identity instead of a static service account key.

## Overview

To use Vertex AI with AIRS Gateway you can either supply a service account JSON
key on the integration, or let the gateway obtain an OAuth 2 access token from
the GCP metadata server using the identity bound to its pod. Workload Identity
is the more secure and recommended option, since no long-lived credential is
stored in the control plane or in the cluster.

Workload Identity requires **two** settings, one on each side. Configuring only
one of them is the most common cause of Vertex authentication failures:

| Where | Setting | Value |
|-------|---------|-------|
| Integration / Virtual Key (control plane) | Vertex auth type | `workload` |
| Gateway deployment (`values.yaml`) | `environment.data.GCP_AUTH_MODE` | `workload` |

The integration setting selects the workload-identity code path; the
`GCP_AUTH_MODE` environment variable is what permits that path to actually
request a token. If `GCP_AUTH_MODE` is unset, the gateway sends the request with
no bearer token and Vertex rejects it.

## Step 1: Create the Google Service Account

Create a GSA in the project that hosts your Vertex AI models and grant it the
Vertex AI user role:

```bash
gcloud iam service-accounts create <GSA> --project <MODEL_PROJECT_ID>

gcloud projects add-iam-policy-binding <MODEL_PROJECT_ID> \
  --member "serviceAccount:<GSA>@<GSA_PROJECT_ID>.iam.gserviceaccount.com" \
  --role roles/aiplatform.user
```

The project that owns the models does not have to be the project the cluster
runs in — grant `roles/aiplatform.user` on whichever project serves the models.

## Step 2: Bind the Kubernetes Service Account

Allow the gateway's KSA to impersonate the GSA:

```bash
gcloud iam service-accounts add-iam-policy-binding \
  <GSA>@<GSA_PROJECT_ID>.iam.gserviceaccount.com \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:<CLUSTER_PROJECT_ID>.svc.id.goog[<NAMESPACE>/<KSA>]"
```

## Step 3: Annotate the Gateway Service Account

```yaml
serviceAccount:
  create: true
  automount: true
  name: <KSA>
  annotations:
    iam.gke.io/gcp-service-account: <GSA>@<GSA_PROJECT_ID>.iam.gserviceaccount.com
```

## Step 4: Enable Workload Auth on the Gateway

```yaml
environment:
  data:
    GCP_AUTH_MODE: workload
```

With `GCP_AUTH_MODE: workload` the gateway requests an access token from the GKE
metadata server and caches it for the lifetime of the token.

If you are running outside GKE and authenticating through Workload Identity
Federation, set `GCP_WIF_AUDIENCE` to your federated audience as well; when it
is set the gateway exchanges a federated token for that audience instead of
reading the metadata server.

## Step 5: Configure the Integration

On the Vertex AI integration (or Virtual Key) in the control plane, set the
auth type to `workload` and provide the Vertex project ID and region. Leave the
service account JSON field empty — if a JSON key is present it takes precedence
and the workload-identity path is skipped.

## Step 6: Verify

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

## Important Notes

- Both the integration auth type **and** `GCP_AUTH_MODE` must be set to
  `workload`. Setting only one produces an authentication error.
- `GCP_AUTH_MODE` is shared with the GCS log store. If you already set it for
  [Log Store](./LogStore.md) GCS access, Vertex uses the same value.
- Anthropic models on Vertex must be addressed with the publisher prefix, for
  example `anthropic.claude-sonnet-4-5@20250929`. Without the prefix the gateway
  resolves the request against `publishers/google` and Vertex returns a 404.
- Changing `environment.data` rolls the gateway pods, since the values are
  rendered into the deployment.

## Troubleshooting

**`Request had invalid authentication credentials. Expected OAuth 2 access token, login cookie or other valid authentication credential.`**

The gateway sent no usable token. Check, in order:

1. `GCP_AUTH_MODE` is present in the running pod —
   `kubectl exec <pod> -- env | grep GCP_AUTH_MODE`. This is the most common
   cause: the integration is set to `workload`, but the variable was never added
   to `values.yaml`.
2. The integration's auth type is `workload` and its service account JSON field
   is empty.
3. Workload Identity itself resolves from inside the pod:

   ```bash
   kubectl exec <pod> -- wget -qO- \
     --header "Metadata-Flavor: Google" \
     "http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/email"
   ```

   This should print the GSA, not the node's default compute service account.
   If it prints the wrong identity, the KSA annotation or the
   `roles/iam.workloadIdentityUser` binding is wrong.

**`Publisher model ... was not found or your project does not have access to it`**

Authentication succeeded and the request reached Vertex. Either the model name
is missing its publisher prefix (see Important Notes above), the model is not
enabled in the target project, or it is not served in the configured region.

**`Permission 'aiplatform.endpoints.predict' denied`**

Workload Identity resolved correctly but the GSA is missing
`roles/aiplatform.user` on the project that serves the model.
