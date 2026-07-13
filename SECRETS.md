# Gestion des secrets en GitOps

**Règle d'or : aucune vraie valeur secrète ne doit jamais être commitée dans
Git**, même en base64 (le base64 n'est PAS du chiffrement, c'est juste un
encodage — n'importe qui avec accès au dépôt peut le décoder en une commande).

Le fichier `apps/cnss/base/secret.yaml` de ce dépôt ne contient que des
placeholders (`CHANGE_ME_BASE64`). Pour un vrai déploiement, deux options :

## Option A — Sealed Secrets (recommandé, simple, pas de dépendance externe)

Sealed Secrets (Bitnami) chiffre vos secrets avec une clé publique ; seul le
contrôleur installé sur VOTRE cluster peut les déchiffrer. Le résultat chiffré
PEUT être commité sans risque dans Git.

```bash
# 1. Installer le contrôleur sur le cluster (une fois)
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.27.1/controller.yaml

# 2. Installer le client kubeseal sur votre poste
curl -fsSLo kubeseal https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.27.1/kubeseal-0.27.1-linux-amd64
chmod +x kubeseal && sudo mv kubeseal /usr/local/bin/

# 3. Créer votre Secret normalement (mais SANS le commiter)
kubectl create secret generic cnss-backend-secret -n cnss \
  --from-literal=SPRING_DATASOURCE_PASSWORD='votre_vrai_mdp' \
  --from-literal=SPRING_MAIL_PASSWORD='votre_vrai_mdp_app' \
  --from-literal=JWT_SECRET='votre_vrai_secret' \
  --from-literal=MINIO_SECRET_KEY='votre_vrai_secret' \
  --from-literal=GROQ_API_KEY='votre_vraie_cle' \
  --dry-run=client -o yaml > /tmp/secret-plain.yaml

# 4. Le chiffrer
kubeseal --format yaml < /tmp/secret-plain.yaml > apps/cnss/base/sealed-secret.yaml

# 5. Supprimer le brouillon en clair, remplacer secret.yaml par le résultat chiffré
rm /tmp/secret-plain.yaml
# Dans apps/cnss/base/kustomization.yaml, remplacer "secret.yaml" par "sealed-secret.yaml"

# 6. Commiter sealed-secret.yaml (chiffré, sans risque)
git add apps/cnss/base/sealed-secret.yaml apps/cnss/base/kustomization.yaml
git commit -m "Add sealed secrets"
git push
```

ArgoCD synchronise le `SealedSecret`, le contrôleur du cluster le déchiffre
automatiquement en un vrai `Secret` Kubernetes utilisable par vos pods.

## Option B — Ne pas versionner les secrets du tout

Appliquez les vrais secrets une seule fois manuellement en dehors de Git, et
excluez `secret.yaml` de la synchronisation ArgoCD :

```bash
kubectl create secret generic cnss-backend-secret -n cnss \
  --from-literal=SPRING_DATASOURCE_PASSWORD='...' \
  --from-literal=SPRING_MAIL_PASSWORD='...' \
  --from-literal=JWT_SECRET='...' \
  --from-literal=MINIO_SECRET_KEY='...' \
  --from-literal=GROQ_API_KEY='...'
```

Retirez alors `secret.yaml` de `apps/cnss/base/kustomization.yaml`. Simple,
mais moins "GitOps pur" (ce secret n'est plus traçable/reproductible depuis
Git) — utile en dépannage rapide, moins pour de la production long terme.

## Rappel : rotation

Les identifiants suivants ont circulé en clair dans une conversation de chat
et doivent être régénérés avant la mise en production :
- Mot de passe d'application Gmail
- Clé API Groq
- Secret JWT
