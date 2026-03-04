# 🖥️ Installation et configuration de VMware vCenter Server (VCSA)

> Cette page décrit l’installation et la configuration initiale de **VMware vCenter Server Appliance (VCSA)** pour la gestion centralisée des hôtes **ESXi** du projet **BESAFE**.

---

## ⚙️ Détails techniques

| Élément | Valeur |
|:--|:--|
| **Nom d’hôte** | NTE-VCSA-001 |
| **Adresse IP** | 10.47.101.204 |
| **Version** | vCenter Server Appliance 8.0.2 |
| **Hôtes gérés** | NTE-ESXI-001 (10.47.101.201) – NTE-ESXI-002 (10.47.101.202) |
| **Domaine** | BESAFE.LOCAL |
| **Système d’exploitation** | VMware Photon OS |
| **Rôle** | Gestion centralisée du cluster ESXi BESAFE |
| **Objectif** | Supervision, clustering, gestion des ressources et des licences |

---

<details>
<summary>🚀 Étape 1 : Installation du vCenter Server Appliance (VCSA)</summary>

> Objectif : déployer l’appliance vCenter sur un hôte ESXi existant, en deux phases (déploiement + configuration).

---

### 📸 Capture 1 – Lancement de l’installeur VCSA
![Lancement de l’installeur](/assets/installation-vcenter-001.png)

Créer une VM pour le VCSA, mettre le **.ova** de vCenter Server puis lancer la VM.
Configurer le réseau :

| Paramètre | Valeur |
|:--|:--|
| **Network** | Management Network |
| **Adresse IP** | 10.47.101.204 |
| **Masque** | 255.255.255.0 |
| **Passerelle** | 10.47.101.254 |
| **DNS** | 10.47.30.50 |
| **Nom d’hôte (FQDN)** | `NTE-VCSA-001.besafe.local` |

> 📘 Utiliser une adresse **statique** est obligatoire pour le bon fonctionnement du service vCenter.  
Choisir **“Install”** pour commencer le déploiement.

---

### 📸 Capture 2 – Sélection du déploiement
![Choix du déploiement](/assets/installation-vcenter-002.png)

Sélectionner **“Deploy vCenter Server”**.  

---

### 📸 Capture 3 – Connexion à l’hôte ESXi
![Connexion à l'hôte ESXi](/assets/installation-vcenter-003.png)

Renseigner les informations de l’hôte ESXi cible :

| Paramètre | Valeur |
|:--|:--|
| **Adresse ESXi** | 10.47.101.201 |
| **Utilisateur** | root |
| **Mot de passe** | ******** |

> 💡 Il est possible d’utiliser `10.47.101.202` pour un second déploiement ou un backup du vCenter.

---

### 📸 Capture 4 – Choix du nom et mot de passe du VCSA
![Nom et mot de passe VCSA](/assets/installation-vcenter-004.png)

Nom de l’appliance : `NTE-VCSA-001`  
Mot de passe administrateur : défini pour le compte **root** de la VCSA.

---

### 📸 Capture 5 – Validation du déploiement
![Validation du déploiement](/assets/installation-vcenter-005.png)

L’assistant vérifie la connectivité avec l’hôte ESXi et la configuration réseau.  
Cliquer sur **Finish** pour lancer le déploiement.

---

### 📸 Capture 6 – Installation de la VCSA
![Déploiement de la VCSA](/assets/installation-vcenter-006.png)

Le déploiement démarre : la VCSA est copiée, configurée et initialisée sur l’hôte.  
Cette étape peut durer plusieurs minutes selon les performances du stockage.

---

### 📸 Capture 7 – Étape 1 terminée
![Fin du déploiement étape 1](/assets/installation-vcenter-007.png)

Une fois la première phase terminée, un message s’affiche :  
> **Stage 1 completed successfully**

Cliquer sur **Continue** pour passer à la configuration initiale.
Vous pouvez également vous rendre sur l'interface **vCenter Server Management** sur le port **5480**

---

### 📸 Capture 8 – Démarrage de la configuration (Stage 2)
![Configuration initiale](/assets/installation-vcenter-008.png)

L’étape 2 permet de définir les paramètres du système et du SSO.

---

### 📸 Capture 9 – Configuration NTP et SSH
![Paramètres NTP et SSH](/assets/installation-vcenter-009.png)

Définir un serveur NTP (synchronisation horaire essentielle).  
Optionnel : activer **SSH access** pour maintenance.

---

### 📸 Capture 10 – Configuration du SSO
![Configuration du SSO](/assets/installation-vcenter-010.png)

Créer un domaine SSO :
| Paramètre | Valeur |
|:--|:--|
| **Domaine SSO** | vsphere.local |
| **Nom d’utilisateur** | administrator@vsphere.local |
| **Mot de passe** | ******** |

---

### 📸 Capture 11 – Résumé avant installation
![Résumé avant installation](/assets/installation-vcenter-011.png)

L’assistant affiche un résumé complet des paramètres avant finalisation.

---

### 📸 Capture 12 – Installation en cours
![Installation en cours](/assets/installation-vcenter-012.png)

La configuration des services du vCenter se lance (base de données, SSO, Tomcat, etc.).

---

### 📸 Capture 13 – Fin de l’installation
![Installation terminée](/assets/installation-vcenter-013.png)

Une fois l’installation terminée, le message suivant apparaît :  
> **vCenter Server has been successfully deployed.**

---

### 📸 Capture 14,15 et 16 – Accès au portail vCenter
![Accès au portail vCenter](/assets/installation-vcenter-014.png)
![Authentification au portail vCenter](/assets/installation-vcenter-015.png)
![Portail d'administration vCenter](/assets/installation-vcenter-016.png)

Ouvrir l’URL suivante :
https://10.47.101.204
ou  
https://NTE-VCSA-001.besafe.local

Se connecter avec le compte `administrator@vsphere.local`.

---

</details>

---

<details>
<summary>🧾 Étape 2 : Activation de la licence</summary>

> Objectif : activer la licence du vCenter Server via l’interface web et vérifier son enregistrement.

---

### 📸 Capture 17 et 18 – Accès à la gestion des licences
![Administration](/assets/licence-vcenter-017.png)
![Gestion des licences](/assets/licence-vcenter-018.png)

Dans le menu de gauche :
Administration → Licensing

---

### 📸 Capture 19 – Ajout d’une licence
![Ajout d’une licence](/assets/licence-vcenter-019.png)

Cliquer sur **Add New License** et saisir la clé fournie par VMware.  
Une fois validée, la licence apparaît dans la liste.

---

### 📸 Capture 20 – Attribution de la licence au vCenter
![Attribution licence](/assets/licence-vcenter-020.png)

Sélectionner la licence ajoutée et cliquer sur **Assign License**.  
L’état passe à **Active**.

---

</details>

---

<details>
<summary>🏗️ Étape 3 : Création du Datacenter</summary>

> Objectif : créer un conteneur logique “Datacenter” pour regrouper les hôtes ESXi et ressources associées.

---

### 📸 Capture 21 – Création d’un Datacenter
![vSphere Client](/assets/datacenter-vcenter-021.png)

Clic droit sur la racine :
vCenter Server → New Datacenter

---

### 📸 Capture 22 – Nommer le Datacenter
![Création Datacenter](/assets/datacenter-vcenter-022.png)

Nommer le datacenter :
BESAFE-DATACENTER

---

### 📸 Capture 23 – Datacenter créé
![Datacenter créé](/assets/datacenter-vcenter-023.png)

Le datacenter apparaît dans l’arborescence de vCenter.  
Il servira à héberger les clusters, hôtes et datastores.

---

</details>

---

<details>
<summary>🧩 Étape 4 : Création du Cluster vSphere</summary>

> Objectif : regrouper les hôtes ESXi dans un **cluster** pour activer les fonctionnalités HA, DRS, et vMotion.

---

### 📸 Capture 24 – Création du Cluster depuis le Datacenter
![Accès au Datacenter](/assets/cluster-vcenter-024.png)

Depuis le Datacenter créé :
BESAFE-DATACENTER → Actions → New Cluster

---

### 📸 Capture 25 – Nom du cluster
![Nom du cluster](/assets/cluster-vcenter-025.png)

Nom du cluster :
BESAFE-CLUSTER

---

### 📸 Capture 26 – Activation des services
![Activation services](/assets/cluster-vcenter-026.png)

Cocher :
- ✅ **vSphere HA** (High Availability)  
- ✅ **vSphere DRS** (Dynamic Resource Scheduler)  
- ✅ **vMotion**

---

### 📸 Capture 27 – Ajouter un hôte ESXi
![Ajout d’un hôte](/assets/cluster-vcenter-027.png)

Clic droit sur le cluster → **Add Host**

---

### 📸 Capture 28 – Saisie des informations de l’hôte
![Infos hôte](/assets/cluster-vcenter-028.png)

Renseigner :
| Champ | Valeur |
|:--|:--|
| **Nom ou IP** | `10.47.101.201` |
| **Utilisateur** | `root` |
| **Mot de passe** | ******** |

---

### 📸 Capture 29 – Hôtes ajoutés
![Hôtes ajoutés](/assets/cluster-vcenter-029.png)

Les deux hôtes apparaissent dans le cluster.  
Le statut passe à **Connected**.

Le cluster BESAFE est maintenant pleinement opérationnel.  
Les ressources (CPU, RAM, stockage) sont mutualisées.

---

</details>

---

<details>
<summary>🔐 Étape 5 : Intégration au domaine Active Directory</summary>

> Objectif : intégrer le vCenter au domaine **BESAFE.LOCAL** pour permettre l’authentification AD.

---

### 📸 Capture 30 – Accès à la configuration d’identité
![Accès configuration identité](/assets/joindomain-vcenter-030.png)

Dans vSphere Client :
Administration → Single Sign-On → Configuration

---

### 📸 Capture 31 – Ajout d’un domaine d’identité
![Ajout domaine identité](/assets/joindomain-vcenter-031.png)

Cliquer sur **Domain Active Directory → Add**

---

### 📸 Capture 32 – Configuration de la source AD
![Configuration source AD](/assets/joindomain-vcenter-032.png)

Choisir **Active Directory (Integrated Windows Authentication)**  
et renseigner :
| Champ | Valeur |
|:--|:--|
| **Domain name** | BESAFE.LOCAL |
| **User name** | Administrator |
| **Password** | ******** |

---

### 📸 Capture 33 – Ajout du vCenter au domaine
![Ajout vCenter domaine](/assets/joindomain-vcenter-033.png)

Toujours dans :
Administration → System Configuration → Nodes
Sélectionner `NTE-VCSA-001` → **Join Domain**

---

### 📸 Capture 34 – Validation finale
![Validation finale](/assets/joindomain-vcenter-034.png)

Le vCenter apparaît désormais comme membre du domaine **BESAFE.LOCAL**.  
Les comptes AD peuvent être utilisés pour se connecter à vCenter.

---

</details>

---

<details>
<summary> ✅ Étape 6 : Priorités HA des VMs (Restart Priority)</summary>
  
> Les VMs sont classées par priorité HA (Highest, High, Medium, Low, Lowest, Disabled)  
  
  
  
### Priorités HA des VMs (vSphere HA)

| **Priorité HA** | **VMs concernées** | **Rôle / Justification principale** |
|-|-|-|
| **Highest**     | `NTE-DC` | Contrôleurs de domaine (AD/DNS/Kerberos) – socle de l’infra. |
| **High**        | `NTE-VCSA`,`NTE-DB` | Management vSphere et  Cluster DB Galera. |
| **Medium**      | `NTE-K8S`,`NTE-NPM`,`NTE-NTP`| Nœuds K8S 3 nœuds, Reverse Proxy et Serveurs NTP. |
| **Low**         | `NTE-SUBCA`, `NTE-SERVICECA` | PKI interne. |
| **Lowest**      | `NTE-BACKUP` | Serveurs de sauvegarde. |
| **Disabled**    | `NTE-ROOTCA` | Root CA – volontairement hors périmètre HA. |
  
### 📸 Capture 35 – Priorités HA
![Priorités HA](/assets/ha-vcenter-035.png)  
</details>  
 
--- 
 
<details>
<summary>🧾  Étape 7 : Règles DRS</summary>  

### Host Groups 

| **Host Group**  | **Contenu (hôtes)**                      | **Commentaire**                      |
|-----------------|-------------------------------------------|--------------------------------------|
| HOST-GROUP-A    | `ESXi-01` (ex : NTE-SRV-001-ESXi)         | Hôte principal / plus chargé.        |
| HOST-GROUP-B    | `ESXi-02` (ex : NTE-SRV-002-ESXi)         | Hôte secondaire / plus calme.        |

### VM Groups (exemples)

| **VM Group**            | **VMs incluses**                                           |
|-------------------------|-------------------------------------------------------------|
| VM-GROUP-VCSA           | `NTE-VCSA-001`                                             |
| VM-GROUP-DC     | `NTE-DC-001`, `NTE-DC-002`                                           |
| VM-GROUP-DB    | `NTE-DB-001`,`NTE-DB-002`,`NTE-DB-003`                                           |

### 📸 Capture 36 – DRS
![DRS](/assets/drs-vcenter-036.png)  
  
  
  
### Règles VM-VM configurées

| **Nom de la règle**         | **Type**                     | **VMs incluses**                             | **Effet recherché**                                            |
|-----------------------------|------------------------------|----------------------------------------------|-----------------------------------------------------------------|
| Rule-DC-AntiAffinity        | Separate Virtual Machines    | `NTE-DC-001`, `NTE-DC-002`                   | Ne jamais perdre tout l’AD sur un seul host.                   |
| Rule-Galera-AntiAffinity    | Separate Virtual Machines    | `NTE-DB-001`, `NTE-DB-002`, `NTE-DB-003`     | Répartition 2–1 minimum des nœuds Galera.                      |


> 📝 Remarque importante
En cas de panne d’un ESXi :

- **HA prime sur DRS** : les VMs seront redémarrées sur l’hôte survivant même si cela viole temporairement l'anti-affinité.  
- Lorsque les deux hôtes sont de nouveau disponibles, **DRS rééquilibre automatiquement** les VMs pour satisfaire à nouveau les règles.
 

### Règles VM-to-Host

| **Nom de la règle**         | **Type**                    | **VM Group**              | **Host Group**     | **Policy**        | **Effet / justification**                                                                 |
|-----------------------------|-----------------------------|---------------------------|---------------------|-------------------|--------------------------------------------------------------------------------------------|
| Rule-VCSA-Placement         | Virtual Machines to Hosts   | VM-GROUP-VCSA             | HOST-GROUP-A        | **Should run on** | vCenter préférablement sur ESXi-01.                                                        |
| Rule-Backup001-Placement    | Virtual Machines to Hosts   | VM-GROUP-BACKUP-001       | HOST-GROUP-A        | **Must run on**   | BACKUP-001 lié au stockage local de ESXi-01.                                               |
| Rule-Backup002-Placement    | Virtual Machines to Hosts   | VM-GROUP-BACKUP-002       | HOST-GROUP-B        | **Must run on**   | BACKUP-002 lié au stockage local de ESXi-02.                                               |
### 📸 Capture 37 – DRS
![DRS](/assets/drs-vcenter-037.png)
</details>   
  
---

## ✅ Résumé global

| Étape | Objectif | Résultat attendu |
|:--|:--|:--|
| 1️⃣ **Installation de la VCSA** | Déploiement du vCenter sur ESXi | Serveur vCenter opérationnel |
| 2️⃣ **Activation de la licence** | Enregistrement d’une clé VMware | vCenter sous licence active |
| 3️⃣ **Création du Datacenter** | Conteneur logique pour les hôtes | Datacenter “BESAFE-DATACENTER” créé |
| 4️⃣ **Création du Cluster** | Regroupement des hôtes ESXi | Cluster “BESAFE-CLUSTER” actif |
| 5️⃣ **Intégration AD** | Authentification via Active Directory | vCenter intégré à BESAFE.LOCAL |
| **6️⃣  Priorités HA des VMs** | Définir l’ordre de redémarrage des VM critiques en cas de failover | HA structuré selon criticité |
| **7️⃣  Règles DRS** | Optimiser le placement automatique des VM | Anti-affinité + VM-to-Host appliqués

---

## 🔗 Liens utiles

- [Documentation VMware vCenter Server](https://docs.vmware.com/fr/VMware-vSphere/index.html)
- [Guide d’installation ESXi](/Esxi)
- [Schéma BESAFE](/Infrastructure/Schéma)

