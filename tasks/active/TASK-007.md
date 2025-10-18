---
id: TASK-007
title: Re-encrypt Sealed Secrets with Current Controller Key
status: active
priority: critical
created: 2025-10-18
updated: 2025-10-18
assignee: brendan
labels: [human-action, infrastructure, security, sealed-secrets, re-encryption]
type: human-action
estimated_time: 2-3 hours
---

## Description

**HUMAN ACTION REQUIRED**: Re-encrypt all sealed secrets with the current sealed-secrets controller public key. The existing sealed secrets were encrypted with an October key that's no longer available after the cluster rebuild. This task provides step-by-step instructions for backing up the current controller key and re-encrypting all secrets.

## Context

Investigation from TASK-006 revealed:
- Sealed secrets created October 13-15, 2025 were encrypted with a key that's no longer available
- Current sealed-secrets controller has keys from October 16 (cluster rebuild)
- Neither available key can decrypt the existing sealed secrets
- Most secret values are available in `~/.secrets/` directories
- Some secrets (database passwords, Redis auth) need to be regenerated

**Resolution**: Re-encrypt all sealed secrets using current controller's public key.

## Prerequisites

- Access to Kubernetes cluster (minikube)
- `kubeseal` CLI tool installed
- Access to `~/.secrets/` directory
- Database access to generate new passwords for triager/midwestmtg databases

## Acceptance Criteria

### Phase 1: Backup Current Sealed-Secrets Controller Key
- [x] Run backup script: `~/.sealed-secrets-backup/backup-sealed-secrets-keys.sh`
- [x] Verify backup created with timestamp in `~/.sealed-secrets-backup/`
- [x] Confirm backup includes: public cert, private key, and complete secret YAML

### Phase 2: Fetch Current Controller Public Key
- [ ] Extract current public key: `kubeseal --fetch-cert > ~/current-sealed-secrets-cert.pem`
- [ ] Verify certificate is valid: `openssl x509 -in ~/current-sealed-secrets-cert.pem -noout -text`
- [ ] Note the certificate fingerprint for reference

### Phase 3: Re-encrypt MidwestMTG Sealed Secrets (5 secrets)
- [ ] **midwestmtg-app-secret**: Re-encrypt using `~/.secrets/midwest_mtg/production.env`
- [ ] **openai-api-key**: Re-encrypt using `~/.secrets/midwest_mtg/openai_apikey`
- [ ] **anthropic-api-key**: Re-encrypt using `~/.secrets/midwest_mtg/anthropic_apikey`
- [ ] **feature-mgmt-git-token**: Re-encrypt using `~/.secrets/.github_pat`
- [ ] **discord-bot-secrets**: Re-encrypt using `~/.secrets/midwest_mtg/discord_*` files

### Phase 4: Re-encrypt Triager Sealed Secrets (6 secrets)
- [ ] **triager-database-secret**: Generate new password, re-encrypt
- [ ] **triager-redis-secret**: Generate new password, re-encrypt
- [ ] **triager-openai-secret**: Re-encrypt using `~/.secrets/midwest_mtg/openai_apikey` (shared)
- [ ] **triager-anthropic-secret**: Re-encrypt using `~/.secrets/midwest_mtg/anthropic_apikey` (shared)
- [ ] **triager-git-token-secret**: Re-encrypt using `~/.secrets/.github_pat`
- [ ] **ccbot-oauth-secret**: Re-encrypt GitHub OAuth credentials (regenerate if needed)

### Phase 5: Re-encrypt Infrastructure Sealed Secrets (2 secrets)
- [ ] **chartmuseum-basic-auth** (infra namespace): Generate new username/password
- [ ] **minio-backup** (infra namespace): Generate new access key/secret key

### Phase 6: Update beckerkube Repository
- [ ] Replace all sealed secret YAML files in `apps/midwestmtg/`
- [ ] Replace all sealed secret YAML files in `apps/triager/`
- [ ] Replace sealed secret YAML files in `infra/`
- [ ] Commit changes to beckerkube repository
- [ ] Push to origin

### Phase 7: Reconcile and Verify
- [ ] Reconcile Flux: `flux reconcile kustomization clusters-minikube --with-source`
- [ ] Verify all sealed secrets are decrypted: `kubectl get sealedsecrets -A`
- [ ] Verify all regular secrets created: `kubectl get secrets -n midwestmtg`, `kubectl get secrets -n triager`
- [ ] Check sealed-secrets controller logs for no errors
- [ ] Verify all HelmReleases reconcile: `flux get helmreleases -A`
- [ ] Confirm all pods running: `kubectl get pods -A | grep -v Running`

## Detailed Instructions

### Step 1: Backup Current Key (CRITICAL)

```bash
# Run the backup script
~/.sealed-secrets-backup/backup-sealed-secrets-keys.sh

# Verify backup created
ls -lh ~/.sealed-secrets-backup/sealed-secrets-key-*.yaml

# This creates a timestamped backup of the current controller key
# so we can restore it if something goes wrong
```

### Step 2: Extract Current Public Key

```bash
# Fetch the current controller's public certificate
kubeseal --fetch-cert > ~/current-sealed-secrets-cert.pem

# Verify it's valid
openssl x509 -in ~/current-sealed-secrets-cert.pem -noout -text

# Get fingerprint for reference
openssl x509 -in ~/current-sealed-secrets-cert.pem -noout -fingerprint -sha256
```

### Step 3: Re-encrypt MidwestMTG Secrets

#### 3.1: midwestmtg-app-secret

```bash
# Source values from production.env
source ~/.secrets/midwest_mtg/production.env

# Create temporary unsealed secret
cat <<EOF > /tmp/midwestmtg-app-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: midwestmtg-app-secret
  namespace: midwestmtg
type: Opaque
stringData:
  secret-key: "$SECRET_KEY"
  service-account-token: "$(cat ~/.secrets/midwest_mtg/SERVICE_ACCOUNT_TOKEN 2>/dev/null || echo 'PLACEHOLDER')"
EOF

# Seal the secret
kubeseal --cert ~/current-sealed-secrets-cert.pem \
  --format yaml < /tmp/midwestmtg-app-secret.yaml \
  > ~/projects/beckerkube/apps/midwestmtg/midwestmtg-app-sealed-secret.yaml

# Clean up temp file
rm /tmp/midwestmtg-app-secret.yaml
```

#### 3.2: openai-api-key

```bash
cat <<EOF > /tmp/openai-api-key.yaml
apiVersion: v1
kind: Secret
metadata:
  name: openai-api-key
  namespace: midwestmtg
type: Opaque
stringData:
  OPENAI_API_KEY: "$(cat ~/.secrets/midwest_mtg/openai_apikey)"
EOF

kubeseal --cert ~/current-sealed-secrets-cert.pem \
  --format yaml < /tmp/openai-api-key.yaml \
  > ~/projects/beckerkube/apps/midwestmtg/openai-sealed-secret.yaml

rm /tmp/openai-api-key.yaml
```

#### 3.3: anthropic-api-key

```bash
cat <<EOF > /tmp/anthropic-api-key.yaml
apiVersion: v1
kind: Secret
metadata:
  name: anthropic-api-key
  namespace: midwestmtg
type: Opaque
stringData:
  ANTHROPIC_API_KEY: "$(cat ~/.secrets/midwest_mtg/anthropic_apikey)"
EOF

kubeseal --cert ~/current-sealed-secrets-cert.pem \
  --format yaml < /tmp/anthropic-api-key.yaml \
  > ~/projects/beckerkube/apps/midwestmtg/anthropic-sealed-secret.yaml

rm /tmp/anthropic-api-key.yaml
```

#### 3.4: feature-mgmt-git-token

```bash
cat <<EOF > /tmp/feature-mgmt-git-token.yaml
apiVersion: v1
kind: Secret
metadata:
  name: feature-mgmt-git-token
  namespace: midwestmtg
type: Opaque
stringData:
  FEATURE_MGMT_GIT_TOKEN: "$(cat ~/.secrets/.github_pat)"
EOF

kubeseal --cert ~/current-sealed-secrets-cert.pem \
  --format yaml < /tmp/feature-mgmt-git-token.yaml \
  > ~/projects/beckerkube/apps/midwestmtg/feature-mgmt-git-token-sealed-secret.yaml

rm /tmp/feature-mgmt-git-token.yaml
```

#### 3.5: discord-bot-secrets

```bash
cat <<EOF > /tmp/discord-bot-secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: discord-bot-secrets
  namespace: midwestmtg
type: Opaque
stringData:
  DISCORD_TOKEN: "$(cat ~/.secrets/midwest_mtg/discord_bot_token)"
  DISCORD_CLIENT_ID: "$(cat ~/.secrets/midwest_mtg/discord_client_id)"
  DISCORD_CLIENT_SECRET: "$(cat ~/.secrets/midwest_mtg/discord_client_secret)"
EOF

kubeseal --cert ~/current-sealed-secrets-cert.pem \
  --format yaml < /tmp/discord-bot-secrets.yaml \
  > ~/projects/beckerkube/apps/midwestmtg/discord-bot-sealed-secret.yaml

rm /tmp/discord-bot-secrets.yaml
```

### Step 4: Re-encrypt Triager Secrets

#### 4.1: triager-database-secret

```bash
# Generate new secure password
TRIAGER_DB_PASSWORD=$(openssl rand -base64 32)

# Store password for later use
echo "$TRIAGER_DB_PASSWORD" > ~/.secrets/triager_postgres_password
chmod 600 ~/.secrets/triager_postgres_password

# Create database URL
DATABASE_URL="postgresql://triager_user:${TRIAGER_DB_PASSWORD}@postgresql-pgvector.infra.svc.cluster.local:5432/triager"

cat <<EOF > /tmp/triager-database-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: triager-database-secret
  namespace: triager
type: Opaque
stringData:
  password: "$TRIAGER_DB_PASSWORD"
  database_url: "$DATABASE_URL"
EOF

kubeseal --cert ~/current-sealed-secrets-cert.pem \
  --format yaml < /tmp/triager-database-secret.yaml \
  > ~/projects/beckerkube/apps/triager/database-sealed-secret.yaml

rm /tmp/triager-database-secret.yaml

# UPDATE POSTGRES: You'll need to update the triager_user password in PostgreSQL
echo "⚠️  IMPORTANT: Update PostgreSQL password for triager_user:"
echo "kubectl exec -n infra postgresql-pgvector-0 -- psql -U postgres -c \"ALTER USER triager_user WITH PASSWORD '$TRIAGER_DB_PASSWORD';\""
```

#### 4.2: triager-redis-secret

```bash
# Generate new Redis password
TRIAGER_REDIS_PASSWORD=$(openssl rand -base64 32)
echo "$TRIAGER_REDIS_PASSWORD" > ~/.secrets/triager_redis_password
chmod 600 ~/.secrets/triager_redis_password

REDIS_URL="redis://:${TRIAGER_REDIS_PASSWORD}@triager-redis.triager.svc.cluster.local:6379/0"

cat <<EOF > /tmp/triager-redis-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: triager-redis-secret
  namespace: triager
type: Opaque
stringData:
  password: "$TRIAGER_REDIS_PASSWORD"
  redis_url: "$REDIS_URL"
EOF

kubeseal --cert ~/current-sealed-secrets-cert.pem \
  --format yaml < /tmp/triager-redis-secret.yaml \
  > ~/projects/beckerkube/apps/triager/redis-sealed-secret.yaml

rm /tmp/triager-redis-secret.yaml

# Update Redis Helm values with new password
echo "⚠️  IMPORTANT: Update triager-redis HelmRelease with auth.password: '$TRIAGER_REDIS_PASSWORD'"
```

#### 4.3: triager-openai-secret

```bash
cat <<EOF > /tmp/triager-openai-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: triager-openai-secret
  namespace: triager
type: Opaque
stringData:
  OPENAI_API_KEY: "$(cat ~/.secrets/midwest_mtg/openai_apikey)"
EOF

kubeseal --cert ~/current-sealed-secrets-cert.pem \
  --format yaml < /tmp/triager-openai-secret.yaml \
  > ~/projects/beckerkube/apps/triager/openai-sealed-secret.yaml

rm /tmp/triager-openai-secret.yaml
```

#### 4.4: triager-anthropic-secret

```bash
cat <<EOF > /tmp/triager-anthropic-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: triager-anthropic-secret
  namespace: triager
type: Opaque
stringData:
  ANTHROPIC_API_KEY: "$(cat ~/.secrets/midwest_mtg/anthropic_apikey)"
EOF

kubeseal --cert ~/current-sealed-secrets-cert.pem \
  --format yaml < /tmp/triager-anthropic-secret.yaml \
  > ~/projects/beckerkube/apps/triager/anthropic-sealed-secret.yaml

rm /tmp/triager-anthropic-secret.yaml
```

#### 4.5: triager-git-token-secret

```bash
cat <<EOF > /tmp/triager-git-token-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: triager-git-token-secret
  namespace: triager
type: Opaque
stringData:
  FEATURE_MGMT_GIT_TOKEN: "$(cat ~/.secrets/.github_pat)"
EOF

kubeseal --cert ~/current-sealed-secrets-cert.pem \
  --format yaml < /tmp/triager-git-token-secret.yaml \
  > ~/projects/beckerkube/apps/triager/git-token-sealed-secret.yaml

rm /tmp/triager-git-token-secret.yaml
```

#### 4.6: ccbot-oauth-secret

```bash
# If you have ccbot OAuth credentials, use them. Otherwise regenerate at GitHub
# For now, using placeholder - UPDATE WITH ACTUAL VALUES
cat <<EOF > /tmp/ccbot-oauth-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: ccbot-oauth-secret
  namespace: triager
type: Opaque
stringData:
  .credentials.json: |
    {
      "client_id": "YOUR_GITHUB_OAUTH_CLIENT_ID",
      "client_secret": "YOUR_GITHUB_OAUTH_CLIENT_SECRET"
    }
EOF

kubeseal --cert ~/current-sealed-secrets-cert.pem \
  --format yaml < /tmp/ccbot-oauth-secret.yaml \
  > ~/projects/beckerkube/apps/triager/ccbot-oauth-sealed-secret.yaml

rm /tmp/ccbot-oauth-secret.yaml

echo "⚠️  IMPORTANT: Update ccbot-oauth-secret with actual GitHub OAuth credentials"
```

### Step 5: Re-encrypt Infrastructure Secrets

#### 5.1: chartmuseum-basic-auth

```bash
# Generate new credentials
CHARTMUSEUM_USER="admin"
CHARTMUSEUM_PASSWORD=$(openssl rand -base64 20)

cat <<EOF > /tmp/chartmuseum-basic-auth.yaml
apiVersion: v1
kind: Secret
metadata:
  name: chartmuseum-basic-auth
  namespace: infra
type: Opaque
stringData:
  username: "$CHARTMUSEUM_USER"
  password: "$CHARTMUSEUM_PASSWORD"
EOF

kubeseal --cert ~/current-sealed-secrets-cert.pem \
  --format yaml < /tmp/chartmuseum-basic-auth.yaml \
  > ~/projects/beckerkube/infra/chartmuseum/chartmuseum-sealed-secret.yaml

rm /tmp/chartmuseum-basic-auth.yaml

echo "ChartMuseum credentials: $CHARTMUSEUM_USER / $CHARTMUSEUM_PASSWORD" >> ~/.secrets/chartmuseum_creds
```

#### 5.2: minio-backup

```bash
# Generate new MinIO credentials
MINIO_ACCESS_KEY=$(openssl rand -hex 20)
MINIO_SECRET_KEY=$(openssl rand -base64 32)

cat <<EOF > /tmp/minio-backup.yaml
apiVersion: v1
kind: Secret
metadata:
  name: minio-backup
  namespace: infra
type: Opaque
stringData:
  MINIO_ACCESS_KEY: "$MINIO_ACCESS_KEY"
  MINIO_SECRET_KEY: "$MINIO_SECRET_KEY"
EOF

kubeseal --cert ~/current-sealed-secrets-cert.pem \
  --format yaml < /tmp/minio-backup.yaml \
  > ~/projects/beckerkube/infra/minio/minio-backup-sealed-secret.yaml

rm /tmp/minio-backup.yaml

echo "MinIO Access Key: $MINIO_ACCESS_KEY" >> ~/.secrets/minio_creds
echo "MinIO Secret Key: $MINIO_SECRET_KEY" >> ~/.secrets/minio_creds
```

### Step 6: Commit Changes to beckerkube

```bash
cd ~/projects/beckerkube

# Check what changed
git status

# Add all updated sealed secrets
git add apps/midwestmtg/*.yaml apps/triager/*.yaml infra/**/*.yaml

# Commit with descriptive message
git commit -m "fix(sealed-secrets): Re-encrypt all sealed secrets with current controller key

- Re-encrypted 5 midwestmtg secrets with current sealed-secrets controller key
- Re-encrypted 6 triager secrets (generated new database and Redis passwords)
- Re-encrypted 2 infra secrets (ChartMuseum, MinIO)
- All secrets now use controller key from October 16, 2025

Resolves TASK-007
Unblocks TASK-006"

# Push to origin
git push origin main
```

### Step 7: Reconcile Flux and Verify

```bash
# Reconcile Flux to apply new sealed secrets
flux reconcile kustomization clusters-minikube --with-source

# Wait 30 seconds for secrets to be created
sleep 30

# Verify all sealed secrets are synced (Status.Synced = True)
kubectl get sealedsecrets -A

# Verify regular secrets created
kubectl get secrets -n midwestmtg | grep -E "openai|anthropic|discord|midwestmtg-app"
kubectl get secrets -n triager | grep -E "database|redis|openai|anthropic|git-token|ccbot"

# Check sealed-secrets controller logs (should be no errors)
kubectl logs -n kube-system deployment/sealed-secrets-controller --tail=50 | grep -i error

# Verify HelmReleases reconcile
flux get helmreleases -A

# Check for any pods still in error state
kubectl get pods -A | grep -E "Error|CreateContainerConfigError|ImagePullBackOff"
```

## Post-Completion Actions

After all secrets are re-encrypted and verified:

1. **Update database passwords**:
   ```bash
   # Update triager_user password in PostgreSQL
   kubectl exec -n infra postgresql-pgvector-0 -- psql -U postgres -c "ALTER USER triager_user WITH PASSWORD '$(cat ~/.secrets/triager_postgres_password)';"
   ```

2. **Update Redis password in HelmRelease**:
   - Edit `apps/triager/triager-redis-helmrelease.yaml`
   - Update `auth.password` value reference to use new sealed secret

3. **Update ccbot OAuth credentials**:
   - Go to GitHub Settings > Developer Settings > OAuth Apps
   - Get client ID and secret for ccbot
   - Re-seal ccbot-oauth-secret with actual values

4. **Verify all services start successfully**:
   ```bash
   kubectl get pods -n midwestmtg
   kubectl get pods -n triager
   ```

## Success Criteria

Task is complete when:
- ✅ Current sealed-secrets controller key backed up to `~/.sealed-secrets-backup/`
- ✅ All 13 sealed secrets re-encrypted with current controller public key
- ✅ All sealed secret YAML files updated in beckerkube repository
- ✅ Changes committed and pushed to beckerkube
- ✅ Flux reconciliation completes without errors
- ✅ All sealed secrets show Status.Synced = True
- ✅ All 13 regular secrets created in their namespaces
- ✅ No decryption errors in sealed-secrets controller logs
- ✅ All 11 blocked HelmReleases reconcile successfully (READY=True)
- ✅ All application pods running without CreateContainerConfigError
- ✅ Database and Redis passwords updated in their respective services

## Estimated Time

- Phase 1-2: 15 minutes (backup and fetch cert)
- Phase 3: 30 minutes (5 midwestmtg secrets)
- Phase 4: 45 minutes (6 triager secrets + password updates)
- Phase 5: 20 minutes (2 infra secrets)
- Phase 6: 10 minutes (commit and push)
- Phase 7: 20 minutes (reconcile and verify)
- Post-completion: 20 minutes (database updates, OAuth)

**Total: 2-3 hours**

## Dependencies

**Blocked By:**
- None (can start immediately)

**Blocks:**
- TASK-006: Resolve Sealed Secrets Decryption Failures

**Related:**
- TASK-006: Investigation task that identified this as the resolution path

## Notes

- **Security**: All temporary unsealed secret files are deleted after sealing
- **Backup**: Current controller key is backed up before making changes
- **Passwords**: New passwords generated for triager database and Redis (old passwords unknown)
- **Shared Secrets**: OpenAI and Anthropic API keys shared between midwestmtg and triager
- **OAuth**: ccbot OAuth credentials may need to be regenerated at GitHub if not available

## Progress Log

- 2025-10-18 01:00: Task created based on TASK-006 investigation results
- 2025-10-18 01:00: Confirmed most secret values available in `~/.secrets/` directories
- 2025-10-18 01:00: Task ready for human action - awaiting Brendan to execute steps
