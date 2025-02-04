# External Secrets with Google Secret Manager and Kubernetes

This guide explains how to store credentials in Google Secret Manager (GSM), configure the External Secrets Operator (ESO) to pull them into Kubernetes, and then consume the resulting secret in your pods.

> **Note:** This guide uses GSM to store a JSON file with multiple key-value pairs for database credentials essentially. We use a namespaced SecretStore or a ClusterSecretStore for global access.

## Prerequisites

- [gcloud CLI installed](https://cloud.google.com/sdk/docs/install)
- Access to a GKE cluster with proper network connectivity to external endpoints.
- The Google Secret Manager API enabled.
- Helm (if installing ESO via Helm).
- Grant cluster service account permission to Google Secret Manager with minimum role: secretAccessor

## Step 1: Enable the Secret Manager API

Enable the GSM API in your GCP project:

```bash
gcloud services enable secretmanager.googleapis.com --project=techplain-hub
```

## Step 2: Create Secrets in Google Secret Manager

### Create a Secret for Database Credentials

Create a JSON file named ex: `db-creds.json` with your database credentials:

```json
{
  "CONFIG_DB_ODIBETS_PASSWORD": "xxxx",
  "CONFIG_DB_ODIBETS_USERNAME": "xxxx",
  "CONFIG_DB_WEB_HOST": "dev-mysql-host"
}
```

Create the secret in GSM:

```bash
gcloud secrets create db_creds --replication-policy="automatic" --project=techplain-hub
```

Then add the JSON file as a secret version:

```bash
gcloud secrets versions add db_creds --data-file=db-creds.json --project=techplain-hub
```

## Step 3: Grant Kubernetes Service Account Access to GSM

List your GKE service accounts:

```bash
gcloud iam service-accounts list --project=techplain-hub
```

Grant the service account access to Secret Manager:

```bash
gcloud projects add-iam-policy-binding techplain-hub \
    --member="serviceAccount:kareco-test-cluster-gke-svc@techplain-hub.iam.gserviceaccount.com" \
    --role="roles/secretmanager.secretAccessor"
```

## Step 4: Deploy the External Secrets Operator (ESO)

### Using Helm

Add the Helm repo and install:

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets -n external-secrets --create-namespace
```

Verify that ESO is running:

```bash
kubectl get pods -n external-secrets
```

## Step 5: Configure External Secrets to Use Google Secret Manager

### Create a Kubernetes Secret for the Service Account Key

```bash
kubectl create secret generic gsm-key --from-file=sa_SecretManager.key=key.json -n external-secrets
```

### Create a SecretStore to Authenticate with GSM

Create `cluster-secret-store.yaml`:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: google-secret-store
  namespace: odi-dev
spec:
  provider:
    gcpsm:
      projectID: techplain-hub
      auth:
        secretRef:
          secretAccessKeySecretRef:
            name: gsm-key
            key: sa_SecretManager.key
            namespace: external-secrets
```

Apply it:

```bash
kubectl apply -f cluster-secret-store.yaml
```

### Create an ExternalSecret for Database Credentials

Create `external-secret.yaml`:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: external-pull-db-creds
  namespace: odi-dev
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: google-secret-store
    kind: ClusterSecretStore
  target:
    name: database-creds
  data:
    - secretKey: CONFIG_DB_ODIBETS_PASSWORD
      remoteRef:
        key: db_creds
        property: CONFIG_DB_ODIBETS_PASSWORD
    - secretKey: CONFIG_DB_ODIBETS_USERNAME
      remoteRef:
        key: db_creds
        property: CONFIG_DB_ODIBETS_USERNAME
    - secretKey: CONFIG_DB_WEB_HOST
      remoteRef:
        key: db_creds
        property: CONFIG_DB_WEB_HOST
```

Apply it:

```bash
kubectl apply -f external-secret.yaml
```

Verify the secret:

```bash
kubectl get secret database-creds -n odi-dev -o yaml
```

## Step 6: Use the Secret in Your Kubernetes Deployment

Create `my-app.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: odi-dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-container
        image: alpine:latest
        command: ["/bin/sh", "-c", "echo DB User: $CONFIG_DB_ODIBETS_USERNAME; echo DB Password: $CONFIG_DB_ODIBETS_PASSWORD; echo DB Host: $CONFIG_DB_WEB_HOST; sleep 3600"]
        env:
          - name: CONFIG_DB_ODIBETS_PASSWORD
            valueFrom:
              secretKeyRef:
                name: database-creds
                key: CONFIG_DB_ODIBETS_PASSWORD
          - name: CONFIG_DB_ODIBETS_USERNAME
            valueFrom:
              secretKeyRef:
                name: database-creds
                key: CONFIG_DB_ODIBETS_USERNAME
          - name: CONFIG_DB_WEB_HOST
            valueFrom:
              secretKeyRef:
                name: database-creds
                key: CONFIG_DB_WEB_HOST
```

Apply it:

```bash
kubectl apply -f my-app.yaml
```

## Step 7: Test If It Works

Inspect the pod’s environment variables:

```bash
kubectl exec -it <pod-name> -n odi-dev -- printenv | grep CONFIG_DB_
```

## Additional Notes

- **SecretStore vs. ClusterSecretStore:** Use a `ClusterSecretStore` for multi-namespace access.
- **Secret Key Matching:** Ensure `secretKey` in `ExternalSecret` matches your application’s expected environment variable names.
- **Namespace Considerations:** Kubernetes secrets are namespace-scoped. The ExternalSecret must be in the same namespace as the consuming pod.


