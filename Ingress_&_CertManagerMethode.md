# Ingress & CertManager methode

👉 **La bonne méthode (propre et recommandée)** est :

> **Issuer → Certificate → Secret → Ingress**

Mais il existe aussi un **mode simplifié via annotations**. Je vais t’expliquer les **2 approches clairement**.

---

#  Méthode 1 (RECOMMANDÉE) : Flux complet

## Étapes

### 1. Créer un Issuer / ClusterIssuer

C’est lui qui parle à Let's Encrypt ou HashiCorp Vault

```yaml
kind: ClusterIssuer
```

---

### 2. Créer un Certificate

👉 C’est **l’étape clé que tu mentionnes**

```yaml
kind: Certificate
```

Dans ce fichier tu dis :

* quel domaine
* quel issuer
* quel nom de secret

👉 Exemple :

```yaml
spec:
  secretName: mon-secret
  dnsNames:
    - monapp.com
  issuerRef:
    name: letsencrypt-prod
```

---

### 3. cert-manager fait le travail 

cert-manager va :

* générer la clé privée
* contacter l’issuer
* valider le domaine
* créer automatiquement :

👉 **un Secret Kubernetes (`mon-secret`)**

---

### 4. Utiliser le secret dans l’Ingress

```yaml
tls:
- hosts:
  - monapp.com
  secretName: mon-secret
```

---

## Résumé logique

```
Issuer → Certificate → Secret → Ingress
```

👉 C’est propre, contrôlé, et très utilisé en production

---

#  Méthode 2 : Via annotation (rapide mais implicite)

Ici tu **NE crées PAS de Certificate**

Tu mets juste une annotation dans ton Ingress :

```yaml
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
```

et :

```yaml
tls:
- hosts:
  - monapp.com
  secretName: mon-secret
```

---

## Ce que fait cert-manager

Quand il voit ça :

👉 il crée automatiquement un objet **Certificate en interne**

Donc en réalité :

```
Ingress → (auto) Certificate → Secret
```

---

## ⚠️ Important

👉 Le secret (`mon-secret`) :

* **n’existe pas au début**
* mais tu DOIS quand même mettre son nom dans l’Ingress

👉 cert-manager va le créer après

---

#  Donc ta question exacte :

> "on injecte un secret qu'on a pas encore ?"

👉 ✅ **OUI, exactement !**

C’est normal 😄

Kubernetes accepte ça, et :

* cert-manager va créer ce secret plus tard
* l’Ingress va commencer à fonctionner dès qu’il existe

---

# 🔥 Comparaison claire

| Méthode     | Certificate YAML | Contrôle | Usage           |
| ----------- | ---------------- | -------- | --------------- |
| Annotation  | ❌ non visible    | faible   | simple / rapide |
| Certificate | ✅ explicite      | fort     | prod / propre   |

---

# 🧠 Quand utiliser quoi ?

### 🟢 Utilise Certificate (recommandé)

* production
* Vault
* multi-domaines
* besoin de contrôle

---

### 🟡 Utilise annotation

* tests
* démos
* setup rapide

---

# Bonus (important pour toi)

Dans ton cas avec HashiCorp Vault :

👉 ❌ annotation = pas idéal
👉 ✅ **Certificate obligatoire**

---

#  Résumé final ultra simple

👉 Option 1 (propre) :

```
Issuer
↓
Certificate
↓
Secret (auto)
↓
Ingress
```

👉 Option 2 (rapide) :

```
Ingress (+ annotation)
↓
cert-manager crée Certificate
↓
Secret
```
