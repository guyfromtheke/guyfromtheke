---
title: "ArgoCD Integration Patterns"
date: 2025-04-27
draft: false
description: "Comprehensive guide for integrating ArgoCD with CI/CD pipelines, external tools, and enterprise systems"
tags: ["kubernetes", "argocd", "gitops", "devops", "integration"]
categories: ["Infrastructure", "Kubernetes"]
series: ["GitOps Implementation"]
weight: 4
---

# ArgoCD Integration Patterns

## CI/CD Pipeline Integration

### GitHub Actions Integration
```yaml
name: Deploy to ArgoCD
on:
  push:
    branches: [ main ]
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    - name: Install ArgoCD CLI
      run: |
        curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
        sudo install -m 555 argocd /usr/local/bin/argocd
    - name: Deploy to ArgoCD
      run: |
        argocd app sync my-application --auth-token ${{ secrets.ARGOCD_TOKEN }}
```

### Jenkins Pipeline
```groovy
pipeline {
    agent any
    environment {
        ARGOCD_SERVER = 'argocd.example.com'
        ARGOCD_TOKEN = credentials('argocd-token')
    }
    stages {
        stage('Deploy') {
            steps {
                sh '''
                    argocd login $ARGOCD_SERVER --auth-token $ARGOCD_TOKEN --insecure
                    argocd app sync my-application
                '''
            }
        }
    }
}
```

### GitLab CI Integration
```yaml
deploy:
  stage: deploy
  script:
    - argocd login $ARGOCD_SERVER --auth-token $ARGOCD_TOKEN
    - argocd app sync my-application
  only:
    - main
```

## External Tools Connectivity

### Prometheus Integration
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: argocd-metrics
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: argocd-metrics
  endpoints:
  - port: metrics
    interval: 30s
```

### Grafana Dashboard
```yaml
apiVersion: integreatly.org/v1alpha1
kind: GrafanaDashboard
metadata:
  name: argocd-dashboard
spec:
  json: |
    {
      "dashboard": {
        "title": "ArgoCD Metrics",
        "panels": [
          {
            "title": "Sync Status",
            "type": "graph"
          }
        ]
      }
    }
```

### Slack Notifications
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
data:
  service.slack: |
    token: $slack-token
    username: ArgoCD
  trigger.on-sync-status-changed: |
    - send: [slack]
```

## Authentication Systems

### OIDC Configuration
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  url: https://argocd.example.com
  dex.config: |
    connectors:
      - type: oidc
        id: google
        name: Google
        config:
          issuer: https://accounts.google.com
          clientID: your-client-id
          clientSecret: $oidc-secret
```

### LDAP Integration
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
data:
  url: https://argocd.example.com
  dex.config: |
    connectors:
      - type: ldap
        name: ActiveDirectory
        id: ad
        config:
          host: ldap.example.com:389
          bindDN: cn=admin,dc=example,dc=com
          bindPW: $ldap-password
```

## Git Provider Integration

### GitHub Integration
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: repo-secret
  namespace: argocd
stringData:
  type: git
  url: https://github.com/organization/repo
  password: github-token
  username: git
```

### GitLab Integration
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: gitlab-secret
  namespace: argocd
stringData:
  type: git
  url: https://gitlab.com/organization/repo
  password: gitlab-token
  username: git
```

## Custom Tool Integration

### Webhook Configuration
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
data:
  webhooks: |
    - type: GitHub
      endpoint: /api/webhook
      url: https://github.com/your-org/repo
      secret: $webhook-secret
```

### Custom Health Checks
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
data:
  resource.customizations: |
    custom.group/Kind:
      health.lua: |
        hs = {}
        if obj.status.health == "healthy" then
          hs.status = "Healthy"
        else
          hs.status = "Progressing"
        end
        return hs
```

## Monitoring Solutions

### Datadog Integration
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: datadog-secret
  namespace: argocd
stringData:
  api-key: your-datadog-api-key
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
data:
  datadog.enabled: "true"
  datadog.address: "https://api.datadoghq.com"
```

### New Relic Integration
```yaml
apiVersion: monitoring.newrelic.com/v1alpha1
kind: NRQLAlert
metadata:
  name: argocd-sync-alert
spec:
  query: "SELECT count(*) FROM ArgoCD WHERE status = 'Failed'"
  threshold: 1
```

## Security Integration

### Vault Integration
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
data:
  vault.enabled: "true"
  vault.addr: "https://vault.example.com"
  vault.role: "argocd"
```

### Certificate Management
```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: argocd-cert
spec:
  secretName: argocd-secret
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - argocd.example.com
```

## Troubleshooting Integrations

### Common Integration Issues
1. Authentication failures
2. Webhook misconfiguration
3. Permission problems
4. Network connectivity

### Debug Procedures
```bash
# Check integration status
argocd admin settings validate

# Test webhook
curl -X POST https://argocd/api/webhook -d @webhook-payload.json

# Verify connections
argocd admin app validate
```

## Best Practices

### Security Considerations
- Use service accounts
- Implement least privilege
- Regular secret rotation
- Audit logging

### Performance Optimization
- Rate limiting
- Cache configuration
- Connection pooling
- Resource allocation

## Homelab Integration Examples

### Local Git Server Integration (Gitea)
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: gitea-repo-secret
  namespace: argocd
stringData:
  type: git
  url: http://gitea.local/your-org/repo
  username: git
  password: your-token
```

### Local Registry Integration
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: local-registry-secret
  namespace: argocd
type: kubernetes.io/dockerconfigjson
stringData:
  .dockerconfigjson: |
    {
      "auths": {
        "registry.local:5000": {
          "auth": "base64-encoded-auth"
        }
      }
    }
```

### Tailscale VPN Integration
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  timeout.reconciliation: 180s
  kustomize.buildOptions: --network
```

## HomeLab Monitoring Integration

### Prometheus Stack
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: prometheus-stack
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/prometheus-community/helm-charts.git
    targetRevision: HEAD
    path: charts/kube-prometheus-stack
    helm:
      values: |
        grafana:
          adminPassword: your-admin-password
        prometheus:
          prometheusSpec:
            retention: 15d
            storageSpec:
              volumeClaimTemplate:
                spec:
                  storageClassName: local-path
                  resources:
                    requests:
                      storage: 50Gi
```

### Grafana Dashboard Configuration
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-grafana-dashboard
  namespace: monitoring
  labels:
    grafana_dashboard: "true"
data:
  argocd-dashboard.json: |
    {
      "title": "ArgoCD Overview",
      "panels": [
        {
          "title": "Sync Status",
          "type": "gauge",
          "datasource": "Prometheus"
        }
      ]
    }
```

## Local Authentication Integration

### LDAP with FreeIPA
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  url: https://argocd.local
  dex.config: |
    connectors:
      - type: ldap
        name: FreeIPA
        id: freeipa
        config:
          host: freeipa.local:389
          insecureNoSSL: false
          bindDN: uid=service-account,cn=users,cn=accounts,dc=local
          bindPW: $LDAP_PASSWORD
          userSearch:
            baseDN: cn=users,cn=accounts,dc=local
            filter: (objectClass=person)
            username: uid
            idAttr: uid
            emailAttr: mail
            nameAttr: displayName
          groupSearch:
            baseDN: cn=groups,cn=accounts,dc=local
            filter: (objectClass=groupOfNames)
            userAttr: DN
            groupAttr: member
            nameAttr: cn
```

## Advanced Integration Examples

### HashiCorp Vault Integration
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: argocd-vault-plugin-credentials
  namespace: argocd
stringData:
  VAULT_ADDR: "http://vault.local:8200"
  VAULT_TOKEN: "your-vault-token"
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  configManagementPlugins: |
    - name: argocd-vault-plugin
      generate:
        command: ["argocd-vault-plugin"]
        args: ["generate", "./"]
```

### Custom Metrics Integration
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: argocd-metrics
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: argocd-metrics
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
    metricRelabelings:
    - sourceLabels: [__name__]
      regex: 'argocd_.*'
      action: keep
```

### Backup Integration with Minio
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: minio-backup-credentials
  namespace: argocd
stringData:
  AWS_ACCESS_KEY_ID: your-access-key
  AWS_SECRET_ACCESS_KEY: your-secret-key
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: argocd-backup
  namespace: argocd
spec:
  schedule: "0 1 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: bitnami/kubectl
            command:
            - /bin/sh
            - -c
            - |
              argocd admin export > /backup/argocd-$(date +%Y%m%d).yaml
              mc cp /backup/* minio/argocd-backup/
```

## Next Steps

Return to the [main guide]({{< ref "/posts/argocd-guide/_index" >}}) or review [Real-World Scenarios]({{< ref "/posts/argocd-guide/argocd-real-world-scenarios" >}}).

