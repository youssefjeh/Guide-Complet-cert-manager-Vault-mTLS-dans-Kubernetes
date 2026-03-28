
# 1. AVANT cert-manager (gestion manuelle des certificats)

##  Objectif

Déployer une app sécurisée en HTTPS dans Kubernetes

---

##  Étapes côté DevOps

### 1. Générer un certificat SSL

* Soit auto-signé :

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt
```

* Soit via une autorité comme Let's Encrypt :

  * Installer Certbot
  * Lancer :

```bash
certbot certonly --standalone -d monapp.com
```

---

### 2. Stocker les certificats dans Kubernetes

Créer un secret :

```bash
kubectl create secret tls mon-secret \
  --cert=tls.crt \
  --key=tls.key
```

---

### 3. Configurer l’Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mon-app
spec:
  tls:
  - hosts:
    - monapp.com
    secretName: mon-secret
```

---

### 4. Gérer le renouvellement (LE GROS PROBLÈME 🚨)

* Les certificats expirent (ex: 90 jours avec Let's Encrypt)
* Donc il faut :

  * surveiller l’expiration
  * relancer certbot
  * recréer le secret
  * re-déployer

👉 souvent via :

* cron job
* scripts bash
* pipelines CI/CD

---

##  Problèmes rencontrés

### ❌ 1. Manuel et répétitif

* Beaucoup d’étapes humaines

### ❌ 2. Risque d’expiration

* oubli = site down HTTPS

### ❌ 3. Pas scalable

* 10 services → 10 certificats à gérer

### ❌ 4. Sécurité fragile

* manipulation de clés privées

### ❌ 5. Complexité multi-environnements

* dev / staging / prod = duplication

---

#  2. APRÈS cert-manager (automatisation complète)

---

## Objectif

Automatiser **création + renouvellement + stockage**

---

##  Étapes côté DevOps

### 1. Installer cert-manager

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
```

---

### 2. Définir un Issuer

Exemple avec Let's Encrypt :

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: devops@monapp.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: account-key
    solvers:
    - http01:
        ingress:
          class: nginx
```

---

### 3. Demander un certificat

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: mon-cert
spec:
  secretName: mon-secret
  dnsNames:
  - monapp.com
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
```

---

### 4. Utiliser dans l’Ingress (presque pareil)

```yaml
tls:
- hosts:
  - monapp.com
  secretName: mon-secret
```

---

##  Ce que cert-manager fait automatiquement

### ✅ Génération du certificat

→ via ACME (Let's Encrypt ou autre)

### ✅ Validation domaine

→ HTTP01 / DNS01

### ✅ Création du Secret Kubernetes

→ automatique

### ✅ Renouvellement

→ avant expiration (ex: 30 jours avant)

### ✅ Monitoring

→ status dans Kubernetes

---

#  Comparaison directe

| Étape           | Avant     | Après cert-manager |
| --------------- | --------- | ------------------ |
| Génération cert | manuel    | automatique        |
| Stockage        | manuel    | automatique        |
| Renewal         | scripts   | automatique        |
| Scalabilité     | difficile | native             |
| Sécurité        | risquée   | centralisée        |
| Effort DevOps   | élevé     | faible             |

---

#  Le vrai problème que cert-manager résout

👉 **Automatisation du cycle de vie des certificats**

Plus précisément :

### 🔑 1. Lifecycle management

* création
* renouvellement
* révocation

###  2. Intégration native Kubernetes

* CRDs (Certificate, Issuer)
* declarative (YAML)

###  3. Réduction du toil (travail répétitif)

###  4. Sécurisation des opérations

---

#  Bonus (niveau avancé)

cert-manager fonctionne avec :

* HashiCorp Vault
* CA interne entreprise
* DNS providers (Cloudflare, Route53…)

👉 donc pas limité à Let's Encrypt

---

#  Résumé simple

👉 **Avant cert-manager**

> "Je gère mes certificats moi-même 😓"

👉 **Après cert-manager**

> "Je décris ce que je veux, Kubernetes s’occupe du reste 😎"
