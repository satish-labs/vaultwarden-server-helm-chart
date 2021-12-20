# Bitwarden Server Helm Chart

## Introduction

## Examples

### Local deploy using helm commands

### Deploy Bitwarden Server using Argo CD

#### ApplicationSet config example: bitwarden-server.yaml
- Note: bitwarden secrets needs to be created in the same namespace as bitwarden server. Example is given below

```
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: DOMAIN-COM-bitwarden
  namespace: argocd
spec:
  generators:
  # Allow use of all clusters in argo CD
  - matrix:
      generators:
      - clusters: {}
      - list:
          elements:
          # List for all environment deployments
          - cluster: "in-cluster"
            url: "https://kubernetes.default.svc"
            values:
              prefix: "DOMAIN-COM"
              environment: "prod"
              targetRevision: "main"
              applicationName: "bitwarden"
              domain: "DOMAIN-COM"
              fqdnDomain: "bitwarden.DOMAIN-COM"
              storageClass: "DOMAIN-COM-prod-persistent"
              secretsName: "bitwarden-secrets"
              certIssuer: "issuer-dns01-DOMAIN-COM"
              adminEmail: "admin@DOMAIN-COM"
              smtpFromName: "Vault Alert"
  template:
    metadata:
      name: '{{values.prefix}}-{{values.environment}}-{{values.applicationName}}'
    spec:
      project: '{{values.prefix}}-{{values.environment}}'
      source:
        repoURL: git@github.com:satish-labs/bitwarden-server.git
        targetRevision: "{{values.targetRevision}}"
        path: "./"
        helm:
          version: v3
          releaseName: '{{values.prefix}}-{{values.environment}}-{{values.applicationName}}'
          parameters:
          - name: "bitwarden.domain"
            value: "https://{{values.fqdnDomain}}"
          - name: "secrets.name"
            value: "{{values.secretsName}}"
          - name: "storage.storageClassName"
            value: "{{values.storageClass}}"
          - name: "backup.persistence.storageClassName"
            value: "{{values.storageClass}}"
          - name: "backup.secrets.name"
            value: "{{values.secretsName}}"
          - name: "bitwarden.server_admin_email"
            value: "{{values.adminEmail}}"
          values: |
            deployment:
              healthCheck:
                enabled: true
            bitwarden:
              smtp_from_name: "{{values.smtpFromName}}"
              signups_verify: false
              websocket_enabled: true
              signups_allowed: false
            ingress:
              enabled: true
              annotations:
                kubernetes.io/ingress.class: "nginx"
                nginx.ingress.kubernetes.io/backend-protocol: "HTTP"
                nginx.ingress.kubernetes.io/ssl-redirect: "true"
                nginx.ingress.kubernetes.io/server-snippet: |
                  proxy_ssl_verify off;
                kubernetes.io/ingress.allow-http: "false"
                kubernetes.io/tls-acme: "true"
                cert-manager.io/acme-challenge-type: dns01
                cert-manager.io/acme-dns01-provider: route53
                cert-manager.io/cluster-issuer: {{values.certIssuer}}
              host: {{values.fqdnDomain}}
              tls:
                secretName: tls-vault-{{values.environment}}-{{values.domain}}
            resources:
              enabled: true
              specs: |-
                apiVersion: v1
                kind: Namespace
                metadata:
                  name: {{values.prefix}}-{{values.environment}}-{{values.applicationName}}
      destination:
        server: '{{url}}'
        namespace: '{{values.prefix}}-{{values.environment}}-{{values.applicationName}}'
      syncPolicy:
        automated:
          selfHeal: true
          prune: true
        syncOptions:
        - CreateNamespace=true
```

#### Bitwarden Secrets Example
```
apiVersion: v1
kind: Secret
metadata:
  name: bitwarden-secrets
type: Opaque
data:
  ADMIN_TOKEN: $(64-char-secret | base64)
  SMTP_AUTH_MECHANISM: $(Login,Plain | base64)
  SMTP_EXPLICIT_TLS: $(false | base64)
  SMTP_FROM: $(alerts@domain.com | base64)
  SMTP_FROM_NAME: $(Bitwarden Vault | base64)
  SMTP_HOST: $(email-smtp.us-west-2.amazonaws.com | base64)
  SMTP_PASSWORD: $(aws-secret-key | base64)
  SMTP_PORT: $(587 | base64)
  SMTP_SSL: $(true | base64)
  SMTP_TIMEOUT: $(15 | base64)
  SMTP_USERNAME: $(aws-access-key | base64)
```

## Credits
- Vaultwarden Server: https://github.com/dani-garcia/vaultwarden
