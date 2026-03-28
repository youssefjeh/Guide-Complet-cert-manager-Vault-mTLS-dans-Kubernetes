# Guide Complet: Cert-Manager & Vault & mTLS dans Kubernetes


## Objectif

Mettre en place un système sécurisé dans Kubernetes où :

* cert-manager automatise les certificats
* Vault agit comme autorité de certification (CA)
* Les Pods utilisent ces certificats (mTLS)

## Architecture globale

```
cert-manager → Vault → Certificat → Secret → Pod
```




## 1. Installation de cert-manager

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
```




## 2. Configuration de Vault (PKI)

### Activer PKI

```bash
vault secrets enable pki
```

### Générer une CA

```bash
vault write pki/root/generate/internal \
  common_name="mine.com" \
  ttl=87600h
```

### Créer un rôle

```bash
vault write pki/roles/my-role \
  allowed_domains="mine.com" \
  allow_subdomains=true \
  max_ttl="72h"
```




## 3. Authentification Kubernetes vers Vault

### Activer auth Kubernetes

```bash
vault auth enable kubernetes
```

### Configurer l'accès

```bash
vault write auth/kubernetes/config \
  token_reviewer_jwt="..." \
  kubernetes_host="https://<K8S-API>"
```

### Créer un rôle Vault

```bash
vault write auth/kubernetes/role/cert-manager \
  bound_service_account_names=cert-manager \
  bound_service_account_namespaces=cert-manager \
  policies=default \
  ttl=1h
```

## 4. Créer un ClusterIssuer (Vault)

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: vault-issuer
spec:
  vault:
    server: https://vault-test.mine.com
    path: pki/sign/my-role
    auth:
      kubernetes:
        mountPath: /v1/auth/kubernetes
        role: cert-manager
        serviceAccountRef:
          name: cert-manager
```



## 5. Demander un certificat

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: backend-cert
spec:
  secretName: backend-tls
  issuerRef:
    name: vault-issuer
    kind: ClusterIssuer
  commonName: backend.mine.com
  dnsNames:
  - backend.mine.com
```



## 6. Fonctionnement de cert-manager

1. Détecte la ressource Certificate
2. Contacte Vault
3. S’authentifie via Kubernetes
4. Demande un certificat
5. Vault signe
6. Création d’un Secret



## 7. Utilisation dans un Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  template:
    spec:
      containers:
      - name: app
        image: my-backend
        volumeMounts:
        - name: tls
          mountPath: /tls
          readOnly: true
      volumes:
      - name: tls
        secret:
          secretName: backend-tls
```



## 8. mTLS entre services

### Principe

* Chaque service a un certificat
* Chaque service vérifie l’autre

```
frontend 🔐 ⇄ 🔐 backend
```



## 9. Renouvellement automatique

cert-manager :

* surveille expiration
* renouvelle certificat
* met à jour le Secret

⚠️ Les Pods doivent être redémarrés ou recharger les certificats



## Résumé

* cert-manager génère les certificats
* Vault signe les certificats
* Kubernetes stocke dans des Secrets
* Les Pods utilisent les certificats
* mTLS sécurise la communication



## Bonnes pratiques

* Utiliser des TTL courts
* Sécuriser Vault (TLS + auth)
* Monitorer les certificats
* Automatiser le reload des Pods



## Conclusion

Tu as maintenant un système complet :

* 🔐 sécurisé
* 🤖 automatisé
* ☸️ intégré Kubernetes


