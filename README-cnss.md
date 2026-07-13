# Application CNSS — déploiement ArgoCD

## Structure

```
argocd-apps/cnss/          # Manifests Kubernetes de l'application CNSS
├── 00-namespace.yaml
├── 01-configmap.yaml
├── 02-secret.yaml          # GABARIT — voir SECRETS.md avant de commiter de vraies valeurs
├── 03-mysql.yaml           # PVC + Deployment + Service
├── 04-minio.yaml           # PVC + Deployment + Service
├── 05-backend.yaml         # Deployment (Spring Boot) + Service
├── 06-frontend.yaml        # Deployment (Angular) + Service
└── 07-ingress.yaml

global-app/
└── cnss-application.yaml   # Application ArgoCD qui déploie tout argocd-apps/cnss/
```

Les fichiers sont préfixés par un numéro uniquement pour la lisibilité humaine
en navigant sur GitHub — ArgoCD applique tout le contenu du dossier `argocd-apps/cnss/`,
peu importe l'ordre (Kubernetes gère les dépendances via les retries automatiques ;
c'est pour ça que le backend a un `initContainer` qui attend MySQL plutôt que de
compter sur l'ordre d'application).

## Déploiement

```bash
# 1. Adapter repoURL dans global-app/cnss-application.yaml avec l'URL de CE dépôt

# 2. Gérer les vrais secrets AVANT de commiter (voir SECRETS.md) :
#    remplacer 02-secret.yaml par un SealedSecret, ou l'exclure du repo et
#    l'appliquer manuellement une fois.

# 3. Commit + push
git add .
git commit -m "Add CNSS application"
git push

# 4. Enregistrer l'Application dans ArgoCD
kubectl apply -f global-app/cnss-application.yaml
```

ArgoCD crée le namespace `cnss`, déploie tous les composants, et les garde
synchronisés avec ce dépôt en continu (toute modification manuelle du cluster
est automatiquement annulée grâce à `selfHeal: true`).

## Mettre à jour une image après un nouveau build

Éditez la ligne `image:` dans `argocd-apps/cnss/05-backend.yaml` (ou
`06-frontend.yaml`), commit, push. ArgoCD synchronise automatiquement, sans
commande `kubectl` nécessaire.

## Rappel sécurité

Les identifiants suivants ont circulé en clair dans un chat de développement
et doivent être régénérés avant mise en production : mot de passe
d'application Gmail, clé API Groq, secret JWT.
