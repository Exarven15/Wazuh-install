## **TUTORIEL : Installation de Wazuh 4.12.0 sur une VM avec K3s**

---

### Étape 1 — Pré-requis et installation de K3s

**Dépendances à installer sur la VM (debian 12 ou +)**

```bash
sudo apt update && sudo apt install -y curl git docker.io
```

**Installer K3s (Kubernetes allégé)**

```bash
curl -sfL https://get.k3s.io | sh -
```

**Donner accès à `kubectl`**

```bash
sudo chmod 644 /etc/rancher/k3s/k3s.yaml
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
```

**Vérifier que Kubernetes est prêt**

```bash
kubectl get nodes
```

**Attendre que le status du nœud soit "Ready".**

---

### Étape 2 — Cloner le dépôt Wazuh

```bash
git clone https://github.com/wazuh/wazuh-kubernetes.git
cd wazuh-kubernetes
```

**Changer la version Wazuh (par défaut c'est `5.0.0`)**

```bash
find . -type f -name "*.yaml" -exec sed -i 's/5\.0\.0/4.12.0/g' {} \;
```

**Tu peux vérifier manuellement dans les fichiers sous :**

- **`wazuh/indexer_stack/wazuh-dashboard/dashboard-deploy.yaml`**
- **`wazuh/indexer_stack/wazuh-indexer/cluster/indexer-sts.yaml`**
- **`wazuh/wazuh_managers/wazuh-master-sts.yaml`**
- **`wazuh/wazuh_managers/wazuh-worker-sts.yaml`**

**que la version `4.12.0` a bien été appliquée.**

---

### Étape 3 — Génération des certificats

Avant tout déploiement, on génère les certificats nécessaires :

```bash
bash wazuh/certs/indexer_cluster/generate_certs.sh
bash wazuh/certs/dashboard_http/generate_certs.sh
```

---

### Étape 4 — Modifier les valeurs de stockage si besoin

Tous les fichiers à modifier sont dans :

```bash
envs/local-env/
ou
wazuh/indexer_stack/wazuh-dashboard/dashboard-deploy.yaml
wazuh/indexer_stack/wazuh-indexer/cluster/indexer-sts.yaml
wazuh/wazuh_managers/wazuh-master-sts.yaml
wazuh/wazuh_managers/wazuh-worker-sts.yaml

rm -f ./wazuh/base/storage-class.yaml
rm -f ./envs/local-env/storage-class.yaml

grep -rl "storage-class.yaml" . | while read file; do
  sed -i '/storage-class.yaml/d' "$file"
  echo "🧹 Nettoyé : $file"
done

grep -rl "storageClassName: wazuh-storage" . | while read file; do
  sed -i 's/storageClassName: wazuh-storage/storageClassName: rook-ceph-block/g' "$file"
  echo "✅ Modifié : $file"
done

find . -name "storage-class.yaml"
```

Par défaut, les volumes sont à `50Gi`. On peut le réduire dans les fichiers juste au-dessus.

**Modifier la section :**

```bash
volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      resources:
        requests:
          storage: 10Gi  # ← à adapter
```

---

### Étape 5 — Déploiement de Wazuh

Lance le déploiement :

```bash
kubectl apply -k envs/local-env/
```

Vérifie ensuite que les pods sont tous "Running" :

```bash
kubectl get pods -n wazuh -w
```

Tous les pods doivent passer à **Running** après quelques minutes.

---

### Étape 6 — Accès au Dashboard

Pour voir l’adresse IP du dashboard :

```bash
kubectl get svc -n wazuh
```

Tu dois avoir une ligne comme :

```bash
wazuh-dashboard   ClusterIP   10.43.XXX.XXX   <none>     443/TCP
```

Connecte-toi ensuite via navigateur :

```bash
https://<IP_VM>:<PORT>
```

Ignore l’alerte de certificat auto-signé.

**Identifiants par défaut :**

- **Utilisateur :** `admin`
- **Mot de passe :** `SecretPassword`

# Tutoriel – Installation d’un Agent Wazuh sur une VM distante

---

## 1ère façon de faire:

suivre la doc: 
https://documentation.wazuh.com/current/user-manual/agent/index.html

ou alors passer directement par l’interface graphique de wazuh.
Sinon la 2ème méthode est complètement en commande.

## CÔTÉ SERVEUR WAZUH – Création du mot de passe d’authentification

1. Entrer dans le pod manager :

```bash
kubectl exec -n wazuh -it wazuh-manager-master-0 -- bash
```

1. Créer le fichier de mot de passe :

```bash
echo "<mot de passe>" > /var/ossec/etc/authd.pass
```

Pas besoin de redémarrer quoi que ce soit : Wazuh lit automatiquement ce fichier à intervalle régulier.

---

## CÔTÉ AGENT — Étape 1 : Installer les dépendances

```bash
sudo apt update
sudo apt install curl gnupg apt-transport-https -y
```

---

## CÔTÉ AGENT — Étape 2 : Ajouter le dépôt Wazuh

```bash
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo gpg --dearmor -o /usr/share/keyrings/wazuh-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/wazuh-archive-keyring.gpg] https://packages.wazuh.com/4.x/apt stable main" | sudo tee /etc/apt/sources.list.d/wazuh.list
```

---

## CÔTÉ AGENT — Étape 3 : Installer l’agent

```bash
sudo apt update
sudo apt install wazuh-agent -y
```

---

## CÔTÉ AGENT — Étape 4 : Configurer le fichier ossec.conf

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Modifier ou ajouter la section suivante :

```bash
<server>
  <address><@ IP (external IP)></address> <!-- Remplace par l'IP du serveur -->
  <port>1514</port>
  <protocol>tcp</protocol>
</server>
```

---

## CÔTÉ AGENT — Étape 5 : Authentifier l’agent

```bash
sudo /var/ossec/bin/agent-auth -m <@ IP> -P <MDP>
```

Si l’authentification réussit, le message affiché est :

```
INFO: Valid key received
```

---

## CÔTÉ AGENT — Étape 6 : Démarrer le service agent

```bash
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

---

## CÔTÉ SERVEUR — Vérification sur le dashboard

- Ouvrir le dashboard via l’IP de la VM
- Aller dans l’onglet "Agents"
- L’agent distant doit apparaître avec le statut "Active"
- si il appaît pas il faut attendre 5min et si il appaît toujours pas alors redémarrer l’agent et le serveur.

---

# Commandes utiles — Débug

## Sur l’AGENT

- Lire les logs de l’agent :

```bash
cat /var/ossec/logs/ossec.log
```

- Re-authentifier l’agent :

```bash
/var/ossec/bin/agent-auth -m 192.168.122.237 -P password
```

- Vérifier le statut du service :

```bash
systemctl status wazuh-agent
```

- Redémarrer le service :

```bash
systemctl restart wazuh-agent
```

## Sur le SERVEUR WAZUH (dans le pod)

- Entrer dans le pod :

```bash
kubectl exec -n wazuh -it wazuh-manager-master-0 -- bash
```

- Lire les logs relatifs aux connexions des agents :

```bash
cat /var/ossec/logs/ossec.log | grep authd
```

- Vérifier si un agent a été accepté :

```bash
grep 'Agent key generated' /var/ossec/logs/ossec.log
```

- Vérifier les ports d'écoute

```bash
netstat -tulnp | grep 151
```

- Voir les agents enregistrés :

```
cat /var/ossec/etc/client.keys
```

# Ressources

https://documentation.wazuh.com/current/deployment-options/deploying-with-kubernetes/kubernetes-conf.html
https://documentation.wazuh.com/current/user-manual/agent/index.html
