# 🔁 HA Nginx Proxy Manager (NPM1 / NPM2) – DMZ

> Mise en place d’un **Nginx Proxy Manager hautement disponible** en mode **actif / passif** dans la DMZ, avec **VIP partagée via Keepalived** et **synchro de la config par rsync**.

---

## ⚙️ Détails techniques

| Élément | Valeur |
|:--|:--|
| **OS** | Debian 13 (NPM1 & NPM2) |
| **Rôle** | Reverse-proxy / entrée HTTP(S) unique Internet |
| **NPM1** | `NTE-NPM-001` – `10.47.50.201/24` (DMZ) |
| **NPM2** | `NTE-NPM-002` – `10.47.50.202/24` (DMZ) |
| **VIP** | `10.47.50.200/24` (Keepalived, interface `ens192`) |
| **Ports exposés** | 80 / 443 / 81 (UI Admin) |

---

<details>
<summary>1️⃣ Étape 1 : Installation de Docker + Compose et stack NPM</summary>

### 🎯 Objectif

Installer Docker / Compose sur **NPM1** et **NPM2**, créer les **volumes persistants** et déployer **Nginx Proxy Manager** via un fichier `docker-compose.yml` identique sur les deux nœuds.

---

### 1.1 Installation de Docker & Compose (NPM1 & NPM2)

Sur **chaque** VM (`NTE-NPM-001` et `NTE-NPM-002`) :

```bash
# Màj de base
apt update && apt upgrade -y

# Paquets nécessaires
apt install -y \
  ca-certificates \
  curl \
  gnupg \
  lsb-release \
  docker.io \
  docker-compose-plugin

# Activation du service Docker
systemctl enable --now docker

# Vérification rapide
docker version
docker info | grep -i "Server"
```
> ℹ️ On utilise ici le paquet docker.io de Debian 12 + le plugin officiel docker-compose (docker compose ...).
  
### 1.2 Création de l’arborescence NPM & des volumes
  
Sur chaque NPM :
```bash
# Répertoire de travail de la stack NPM
mkdir -p /opt/npm
cd /opt/npm

# Volumes persistants pour Nginx Proxy Manager
docker volume create npm_npm_data
docker volume create npm_npm_letsencrypt

# Répertoire pour les certificats internes (clé / chaines / PFX…)
mkdir -p /etc/ssl/npm
chmod 750 /etc/ssl/npm
```
> ✅ Les volumes Docker npm_npm_data et npm_npm_letsencrypt seront communs à tous les conteneurs NPM de la VM (persistance de la DB, conf, certs Let’s Encrypt, etc.).

### 1.3 Fichier docker-compose.yml (NPM1 / NPM2)

```yaml  
version: "3.8"

services:
  app:
    container_name: npm-app-1
    image: jc21/nginx-proxy-manager:2.11.1
    restart: unless-stopped
    environment:
      TZ: Europe/Paris
    ports:
      - "80:80"    # HTTP
      - "81:81"    # UI admin
      - "443:443"  # HTTPS
    volumes:
      - npm_npm_data:/data
      - npm_npm_letsencrypt:/etc/letsencrypt
      - /etc/ssl/npm:/etc/ssl/npm:ro

volumes:
  npm_npm_data:
    external: true
  npm_npm_letsencrypt:
    external: true
```

> 🔎 Important : Les volumes sont déclarés en external: true → ils doivent exister avant.
Le bind-mount /etc/ssl/npm:/etc/ssl/npm:ro permet d’injecter tes certificats internes dans le conteneur en lecture seule.
  
### 1.4 Premier démarrage de NPM (avant la HA)
  
Sur chaque NPM, pour un premier test sans HA : 
  
```bash
cd /opt/npm

# Lancer la stack
docker compose up -d

# Vérification
docker ps
```  
  
Tu dois voir, sur chaque nœud :
  
```bash
jc21/nginx-proxy-manager:2.11.1   "/init"   Up ...   0.0.0.0:80->80/tcp, 0.0.0.0:81->81/tcp, 0.0.0.0:443->443/tcp   npm-app-1|2
``` 
  
> ✅ À ce stade, NPM tourne en standalone sur les deux VMs.
  
</details>

---

<details>
<summary>2️⃣ Étape 2 : Mise en place de l’utilisateur dédié, SSH sécurisé et base Rsync NPM1 → NPM2</summary>

### 🎯 Objectif

Mettre en place une **synchro de données NPM1 → NPM2** via `rsync` en s’appuyant sur :

- un **compte système dédié** : `npm-sync`  
- une **authentification par clé SSH** (sans mot de passe)  
- une **restriction d’usage de la clé** : uniquement **depuis NPM1 vers NPM2**  
- une base propre pour la synchro des volumes NPM (qui sera automatisée à l’étape 3)

---

### 2.1 Création de l’utilisateur système `npm-sync` (NPM1 & NPM2)

Sur **NPM1** **et** **NPM2** :

```bash
adduser --system --group --home /var/lib/npm-sync npm-sync
usermod -s /usr/sbin/nologin npm-sync
```
  
Vérification :
```bash
getent passwd npm-sync
ls -ld /var/lib/npm-sync
```
  
Tu dois voir :
- un compte système (UID > 999, shell /usr/sbin/nologin)
- un home /var/lib/npm-sync appartenant à npm-sync:npm-sync
> 🔐 Le compte npm-sync ne peut pas se connecter en interactif (shell /usr/sbin/nologin) → il ne sert que à la synchro.
  
### 2.2 Génération de la clé SSH sur NPM1 uniquement
  
Sur NPM1 (10.47.50.201) :
```bash
sudo -u npm-sync mkdir -p /var/lib/npm-sync/.ssh
sudo -u npm-sync ssh-keygen -t ed25519 -f /var/lib/npm-sync/.ssh/id_ed25519 -N ""
``` 

Tu obtiens :
- clé privée : /var/lib/npm-sync/.ssh/id_ed25519
- clé publique : /var/lib/npm-sync/.ssh/id_ed25519.pub  
  
Récupérer la clé publique dans une variable :
```bash
SSH_PUB_KEY=$(cat /var/lib/npm-sync/.ssh/id_ed25519.pub)
echo "${SSH_PUB_KEY}"
```
> ⚠️ La clé privée ne doit jamais quitter NPM1.
  
### 2.3 Provisionnement de la clé sur NPM2 (avec restriction IP)
  
Autoriser npm-sync@NPM1 à se connecter sur npm-sync@NPM2 sans mot de passe, mais uniquement depuis NPM1 (10.47.50.201).
  
Sur NPM1, exécuter :
  
```bash
# 1) Préparer le home et .ssh côté NPM2 (via root)
ssh root@10.47.50.202 "
  mkdir -p /var/lib/npm-sync/.ssh &&
  chown -R npm-sync:npm-sync /var/lib/npm-sync
"

# 2) Pousser la clé publique avec restriction "from=10.47.50.201"
echo "from=\"10.47.50.201\" ${SSH_PUB_KEY}" | ssh root@10.47.50.202 "
  cat >> /var/lib/npm-sync/.ssh/authorized_keys &&
  chown -R npm-sync:npm-sync /var/lib/npm-sync/.ssh &&
  chmod 700 /var/lib/npm-sync/.ssh &&
  chmod 600 /var/lib/npm-sync/.ssh/authorized_keys
"
```
  
Points importants :
- from="10.47.50.201" : seul NPM1 est autorisé à utiliser cette clé.
- La clé est installée dans /var/lib/npm-sync/.ssh/authorized_keys du compte npm-sync sur NPM2.
> ℹ️ Si SSH root est désactivé, remplacer ssh root@... par un compte admin + sudo.  
  
### 2.4 Test de la connexion SSH sans mot de passe (NPM1 → NPM2)
  
Sur NPM1 :
  
```bash
sudo -u npm-sync ssh -i /var/lib/npm-sync/.ssh/id_ed25519 \
  -o StrictHostKeyChecking=accept-new \
  npm-sync@10.47.50.202 'hostname'
```

Résultat attendu :
- Pas de demande de mot de passe
- Affichage du hostname de NPM2 (ex : NTE-NPM-002)
  
### 2.5 Préparation des répertoires de données NPM (pour rsync)

NPM1 est le nœud source (actif) pour les données, NPM2 le backup :

NPM1 → lit :
```bash
/var/lib/docker/volumes/npm_npm_data/_data/

/var/lib/docker/volumes/npm_npm_letsencrypt/_data/

/etc/ssl/npm/
```
NPM2 → doit pouvoir écrire dans ces répertoires.

> Les ACL d’accès pour npm-sync sur NPM2 seront posées en Étape 3 avec le script final, mais tu peux déjà vérifier l’existence des chemins.

Sur NPM1 & NPM2 :  
```bash
ls -ld /var/lib/docker/volumes/npm_npm_data
ls -ld /var/lib/docker/volumes/npm_npm_letsencrypt
ls -ld /etc/ssl/npm
```
  
> ✅ Les volumes doivent exister, L’écriture côté NPM2 sera gérée avec **setfacl** dans la partie script de synchro.  
  
### 2.6 Préparation de Rsync & Inotify
  
Sur NPM1 & NPM2 :
  
```bash
apt install -y rsync inotify-tools acl
```
  
- **rsync** : synchro de fichiers efficace
- **inotify-tools** : détection des modifications en temps réel
- **acl** : gestion fine des permissions sur les volumes Docker 
</details>
  
---

<details>
<summary>3️⃣ Étape 3 : Script de synchronisation Rsync (NPM1 → NPM2) + ACL + Inotify + Service systemd</summary>

### 🎯 Objectif

Mettre en place une **synchronisation automatique, fiable et continue** des données Nginx Proxy Manager depuis **NPM1 (MASTER)** vers **NPM2 (BACKUP)** :

- Synchro **complète** au démarrage  
- Synchro **temps réel** grâce à `inotifywait`  
- Permissions ACL pour permettre à `npm-sync` d’écrire dans les volumes Docker du NPM2  
- Service systemd pour lancer la synchro en tâche de fond  
- Journalisation claire  
- Aucune synchro inverse (sécurité + cohérence SQLite)

Cette étape finalise le **cluster de données** entre les deux NPM.

---

### 🗂️ 3.1 Répertoires concernés par la synchro

### Sur NPM1 (source)
Les données à synchroniser proviennent de :

| Élément | Chemin |
|--------|--------|
| Base SQLite + config NPM | `/var/lib/docker/volumes/npm_npm_data/_data/` |
| Certificats Let’s Encrypt | `/var/lib/docker/volumes/npm_npm_letsencrypt/_data/` |
| Certificats internes | `/etc/ssl/npm/` |

> ✔ Ces trois chemins doivent exister et contenir des données valides **avant** la synchro.

---

### 🔐 3.2 Configuration des permissions ACL sur NPM2 (destination)

Les volumes Docker appartiennent à `root:root` et ne peuvent PAS être modifiés par un autre utilisateur sans ACL.

### Sur **NPM2**, exécuter :

```bash
apt install -y acl
```
  
Donner accès d’écriture à npm-sync :

```bash
# Traversée des répertoires parents
setfacl -m u:npm-sync:x /var/lib
setfacl -m u:npm-sync:x /var/lib/docker
setfacl -m u:npm-sync:x /var/lib/docker/volumes
setfacl -m u:npm-sync:x /var/lib/docker/volumes/npm_npm_data
setfacl -m u:npm-sync:x /var/lib/docker/volumes/npm_npm_letsencrypt

# Permissions RWX sur les _data
setfacl -R -m u:npm-sync:rwx /var/lib/docker/volumes/npm_npm_data/_data
setfacl -R -m u:npm-sync:rwx /var/lib/docker/volumes/npm_npm_letsencrypt/_data

# Certificats internes
setfacl -R -m u:npm-sync:rwx /etc/ssl/npm
```
  
Vérification :

```bash
getfacl /var/lib/docker/volumes/npm_npm_data/_data | grep npm-sync
```
  
### 3.3 Script de synchronisation sync_npm_to_backup.sh (NPM1)
  
Créer le script :
  
```bash
nano /usr/local/bin/sync_npm_to_backup.sh
```
  
Contenu :
  
```bash
#!/bin/bash

LOGFILE="/var/log/npm-sync.log"

SRC_DATA="/var/lib/docker/volumes/npm_npm_data/_data/"
SRC_LE="/var/lib/docker/volumes/npm_npm_letsencrypt/_data/"
SRC_SSL="/etc/ssl/npm/"

DEST_HOST="npm-sync@10.47.50.202"
DEST_DATA="/var/lib/docker/volumes/npm_npm_data/_data/"
DEST_LE="/var/lib/docker/volumes/npm_npm_letsencrypt/_data/"
DEST_SSL="/etc/ssl/npm/"

SSH_KEY="/var/lib/npm-sync/.ssh/id_ed25519"

RSYNC_OPTS="-az --delete --no-perms --no-owner --no-group --no-times"

log () {
    echo "$(date '+%Y-%m-%d %H:%M:%S') [NPM-SYNC] $1" | tee -a "$LOGFILE"
}

log "Initial full sync starting..."

# Synchro complète
rsync $RSYNC_OPTS -e "ssh -i $SSH_KEY" "$SRC_DATA" "$DEST_HOST:$DEST_DATA"
rsync $RSYNC_OPTS -e "ssh -i $SSH_KEY" "$SRC_LE" "$DEST_HOST:$DEST_LE"
rsync $RSYNC_OPTS -e "ssh -i $SSH_KEY" "$SRC_SSL" "$DEST_HOST:$DEST_SSL"

log "Initial full sync done."

log "Starting inotify loop..."

inotifywait -mrq -e modify,create,delete,move "$SRC_DATA" "$SRC_LE" "$SRC_SSL" \
| while read path action file; do

    log "Change detected: ${action} on ${path}${file}"

    rsync $RSYNC_OPTS -e "ssh -i $SSH_KEY" "$SRC_DATA" "$DEST_HOST:$DEST_DATA"
    rsync $RSYNC_OPTS -e "ssh -i $SSH_KEY" "$SRC_LE" "$DEST_HOST:$DEST_LE"
    rsync $RSYNC_OPTS -e "ssh -i $SSH_KEY" "$SRC_SSL" "$DEST_HOST:$DEST_SSL"

done
```
  
Rendre le script exécutable :

```bash
chmod +x /usr/local/bin/sync_npm_to_backup.sh
touch /var/log/npm-sync.log
```
  
### 3.4 Service systemd pour la synchro (NPM1)
  
Créer le fichier :
  
```bash
nano /etc/systemd/system/npm-sync.service
```
  
Contenu :
  
```ini
[Unit]
Description=HA Sync Service for NPM (rsync + inotify)
After=network-online.target
Wants=network-online.target

[Service]
User=npm-sync
Group=npm-sync
ExecStart=/usr/local/bin/sync_npm_to_backup.sh
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```
  
Recharger + activer :
  
```bash
systemctl daemon-reload
systemctl enable npm-sync
systemctl start npm-sync
```
  
### 3.5 Vérification de la synchronisation

Voir les logs en direct :
  
```bash
journalctl -u npm-sync -f
```
  
Tu dois voir :
  
```sql 
Initial full sync starting...
Initial full sync done.
Starting inotify loop...
Change detected: MODIFY ...
```  

Vérifier côté NPM2 que les fichiers arrivent :
  
```bash
ls -l /var/lib/docker/volumes/npm_npm_data/_data
ls -l /var/lib/docker/volumes/npm_npm_letsencrypt/_data
ls -l /etc/ssl/npm
```
</details> 

---

<details>
<summary>4️⃣ Étape 4 : Cluster Keepalived (VIP), Scripts HA et Service systemd NPM-HA</summary>

### 🎯 Objectif

Déployer la **Haute Disponibilité active/passive** de Nginx Proxy Manager à l’aide de :

- **Keepalived (VRRP)** pour la gestion de la VIP `10.47.50.200`
- **Scripts notify** permettant de démarrer / arrêter proprement le service `npm-ha.service`
- **Service systemd NPM-HA** orchestrant `docker compose up/down`
- **Health-check applicatif** pour basculer automatiquement si NPM tombe  
- **Kill-switch propre** lors des transitions MASTER → BACKUP

---

### 🧱 4.1 Installation de Keepalived (NPM1 & NPM2)

Sur **chaque** nœud :

```bash
apt install -y keepalived
systemctl enable keepalived
```
  
Vérification :
  
```bash
keepalived --version
``` 
  
### 4.2 Scripts HA : npm_master.sh & npm_backup.sh
  
> Ces scripts sont appelés automatiquement par Keepalived lors du passage en :
MASTER → démarrage du service npm-ha.service
BACKUP / FAULT → arrêt du service  
  
### 4.2.1 Script MASTER → start NPM (NPM1 & NPM2)
  
Créer le fichier :
  
```bash
nano /usr/local/bin/npm_master.sh
```
  
Contenu :
  
```bash
#!/bin/bash
exec >> /var/log/keepalived-npm.log 2>&1

echo "[KEEPALIVED] $(date) -> MASTER: starting npm-ha.service"
systemctl start npm-ha.service
RC=$?
echo "[KEEPALIVED] $(date) -> systemctl start npm-ha.service exited with code ${RC}"
exit "$RC"
```
  
### 4.2.2 Script BACKUP/FAULT → stop NPM (NPM1 & NPM2)
  
Créer :
  
```bash
nano /usr/local/bin/npm_backup.sh
```
  
Contenu :

```bash
#!/bin/bash
exec >> /var/log/keepalived-npm.log 2>&1

echo "[KEEPALIVED] $(date) -> BACKUP/FAULT: stopping npm-ha.service"
systemctl stop npm-ha.service
RC=$?
echo "[KEEPALIVED] $(date) -> systemctl stop npm-ha.service exited with code ${RC}"
exit "$RC"
```
  
### 4.2.3 Permissions 
 
```bash
chmod +x /usr/local/bin/npm_master.sh
chmod +x /usr/local/bin/npm_backup.sh
touch /var/log/keepalived-npm.log
```
  
### 4.3 Health-check applicatif  
  
Ce check fait basculer la VIP si :
- le service systemd npm-ha est down
- le conteneur NPM ne tourne plus
- Docker crash
  
Créer le script :
  
```bash
nano /usr/local/bin/check_npm_ha.sh
```
  
Contenu :
  
```bash
#!/bin/bash

# 1) npm-ha doit être actif
systemctl is-active --quiet npm-ha.service
if [ $? -ne 0 ]; then
  exit 1
fi

# 2) Le conteneur NPM doit être en état "Up"
if ! docker ps --format '{{.Names}}' | grep -q '^npm-app-'; then
  exit 1
fi

exit 0
```
  
Permissions :
  
```bash
chmod +x /usr/local/bin/check_npm_ha.sh
```
  
### 4.4 Configuration Keepalived (VRRP) — /etc/keepalived/keepalived.conf
  
Sur NPM1 (MASTER par défaut) :
  
```bash
nano /etc/keepalived/keepalived.conf
```
  
Contenu :
  
```bash
global_defs {
    script_user root
    enable_script_security
}

vrrp_script chk_npm_ha {
    script "/usr/local/bin/check_npm_ha.sh"
    interval 5
    fall 2
    rise 3
}

vrrp_instance VI_NPM {
    state MASTER
    interface ens192
    virtual_router_id 60
    priority 200
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass BesafeHA2025
    }

    virtual_ipaddress {
        10.47.50.200/24 dev ens192
    }

    track_script {
        chk_npm_ha
    }

    notify_master "/usr/local/bin/npm_master.sh"
    notify_backup "/usr/local/bin/npm_backup.sh"
    notify_fault  "/usr/local/bin/npm_backup.sh"
}
``` 
  
Sur NPM2 (BACKUP par défaut) :
  
Même fichier, sauf deux lignes :

```bash
state BACKUP
priority 190
```
  
### 4.5 Service systemd NPM-HA (déjà créé en Étape 1)
> Rappel (pour cohérence de doc) :
 
```ini
[Unit]
Description=Nginx Proxy Manager (HA stack via docker compose)
After=network-online.target docker.service
Requires=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/opt/npm
ExecStart=/usr/bin/docker compose up -d
ExecStop=/usr/bin/docker compose down

[Install]
WantedBy=multi-user.target
```
  
> ⚠️ NE PAS activer (enable) ce service.
Il est contrôlé uniquement par Keepalived. 
  
### 4.6 Rechargement général

Sur NPM1 & NPM2 :  
  
```bash
systemctl daemon-reload
systemctl restart keepalived
```
  
Vérification :
  
```bash
systemctl status keepalived
tail -f /var/log/keepalived-npm.log
ip a | grep 10.47.50.200
```
  
</details> 

---

<details>
<summary>5️⃣ Étape 5 : Scénarios de tests HA (Normal, Failover, Failback)</summary>

### 🎯 Objectif

Valider de manière **systématique** que :

- ✅ La VIP `10.47.50.200` est **toujours présente sur un seul nœud à la fois**
- ✅ Le conteneur NPM ne tourne **que sur le nœud MASTER**
- ✅ La **bascule automatique** fonctionne :
  - en cas de **panne réelle** (reboot / crash NPM1)
  - en cas de **panne applicative** (`npm-ha` down)
- ✅ Le **retour à la normale (failback)** est propre

---

### 🧪 5.1 Test 0 – État normal (NPM1 MASTER, NPM2 BACKUP)

### ✔ Objectif

Vérifier que l’état nominal est :

- NPM1 = MASTER  
- NPM2 = BACKUP  
- VIP + conteneur NPM **uniquement** sur NPM1

---

### 🔹 Commandes à lancer

#### Sur **NPM1** :

```bash
ip a | grep 10.47.50.200
systemctl status keepalived | grep -E "Active|VI_NPM"
systemctl status npm-ha | grep Active
docker ps
```

Sur NPM2 :  
  
```bash
ip a | grep 10.47.50.200
systemctl status keepalived | grep -E "Active|VI_NPM"
systemctl status npm-ha | grep Active
docker ps
```
  
Résultats attendus :
 
|Élément|NPM1 (MASTER)|NPM2 (BACKUP)  
|:--|:--|:--|
|**VIP** 10.47.50.200|✅ Présente sur **ens192**|❌ Absente|
|**keepalived**|**state MASTER** dans les logs|**state BACKUP** dans les logs|
|**npm-ha**|**active** (exited)|**inactive**|
|**docker ps**|npm-app-1 (Up)|Aucun conteneur NPM| 
  
Vérification applicative :
  
```bash
curl -k https://10.47.50.200
```
  
### 5.2 Test A – Failover en cas de panne réelle NPM1 (reboot / crash) 
  
Simuler une vraie panne de NPM1 (comme si la VM tombait) :
- NPM1 ne répond plus
- NPM2 prend le rôle MASTER + VIP + conteneur NPM
  
Sur NPM1 (choisir UNE des options) :

Option recommandée (réaliste) :
  
```bash
reboot
```
  
Option alternative (plus “soft”, mais proche d’un crash service HA) :
  
```bash
systemctl stop keepalived
systemctl stop npm-ha
```
  
> ⚠ Si tu utilises la 2nde option, assure-toi bien que npm-ha est stoppé sur NPM1.
  
Dès que NPM1 est down/non joignable, sur NPM2 :
  
```bash
ip a | grep 10.47.50.200
systemctl status keepalived | grep -E "Active|VI_NPM"
systemctl status npm-ha | grep Active
docker ps
```
  
Attendu :
- VIP **10.47.50.200** présente sur **ens192** de NPM2 
- **keepalived** indique **state MASTER**
- **npm-ha** est **active (exited)**
- **docker ps** montre **npm-app-2** en Up 
  
</details>

---

## ✅ Résumé des étapes – HA Nginx Proxy Manager

| Étape | Objectif | Résultat attendu |
|------|-----------|------------------|
| **1️⃣ Installation Docker & NPM** | Installer Docker, Compose et déployer NPM | Instance NPM fonctionnelle |
| **2️⃣ Mise en place Rsync + SSH** | Configurer la réplication sécurisée NPM1 → NPM2 | Synchro initiale OK |
| **3️⃣ Script Rsync + Inotify** | Réplication continue DATA + SSL | Données toujours à jour |
| **4️⃣ HA Keepalived (VRRP)** | VIP HA + rôles MASTER/BACKUP | Haute disponibilité opérationnelle |
| **5️⃣ Tests Failover / Failback** | Vérifier bascule automatique et retour | HA validée et stable |



## 🔗 Liens utiles

- [Documentation officielle Nginx Proxy Manager](https://nginxproxymanager.com/)
- [Documentation Docker Compose](https://docs.docker.com/compose/)
- [Documentation Keepalived (VRRP)](https://keepalived.readthedocs.io/en/latest/)

