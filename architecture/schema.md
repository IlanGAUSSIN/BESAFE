# 🗺️ Schéma – BESAFE

Cette page présente **l’architecture de référence** du projet BESAFE :
- le **schéma physique** de la baie (rackage, câblage, alimentation) ;
- le **schéma logique** (rôles, segments réseau, services et flux).

> ℹ️ Les visuels ci-dessous sont la **source de vérité**. Tout changement d’infra doit être répercuté ici dans les 24h.

---

## 🧱 Schéma physique – Baie / Datacenter

![schema-logique-besafe.png](/schema-physique-besafe.png)

### 🎯 Objectif
Donner une vue **matérielle** : équipements, U occupées, chemins d’alimentation, agrégations réseau, liens iSCSI/Backups.

### 📌 Légende / Repères rapides
| Étiquette | Élément (exemple) | Rôle / Détails |
|---|---|---|
| **NTE-FW-001** | Pare-feu | Sécurité périmétrique, NAT, IPsec |
| **NTE-SW-001/002** | Switchs ToR | Accès/agrégation, LACP, VLAN trunk |
| **NTE-NAS-001** | NAS/Backup | Repos sauvegardes, repo ISO, iSCSI/NFS |
| **NTE-SRV-001/002** | Hôtes ESXi | Cluster de virtualisation |
| **NTE-OND-001** | Onduleur | Redondance électrique, arrêt propre |

---

## 🧩 Schéma logique – Architecture & Services

![Schéma logique de l’infrastructure](/schema-logique-besafe.png)

### 🗂️ Couches logiques
- **WAN** : accès Internet, VPN/IPsec.
- **Sécurité** : Pare-feu (politiques inter-VLAN, NAT, objets).
- **Réseau** : VLAN/segments (MGMT, SRV, APP, DMZ, BACKUP).
- **Systèmes** : AD DS/DNS/DHCP, PKI/AD CS.
- **Virtualisation** : ESXi/vCenter, stockage (iSCSI/NFS).
- **Services applicatifs** : GLPI, Centreon, Nextcloud, Vaultwarden, Wazuh, Nginx Proxy Manager.
- **Sauvegarde & Restauration** : VeeamBackup.

### 🧾 Inventaire logique
| Domaine | Composant | Exemple d’instances |
|---|---|---|
| **AD / DNS** | DCs | `NTE-DC-001`, `NTE-DC-002` |
| **PKI** | Root / Sub CA | `NTE-ROOTCA-001`, `NTE-UCA-001/002` |
| **Mgmt** | vCenter / ESXi | `VCSA`, `NTE-SRV-001/002` |
| **Apps** | GLPI / Centreon / Nextcloud / Vaultwarden / Wazuh | `NTE-GLPI-001/002`, `NTE-CENTREON-001/002`, `NTE-CLOUD-001/002`, `NTE-VAULT-001/002`, `NTE-WAZUH-001` |
| **Proxy** | Reverse-Proxy | `NTE-NPM-001` (ou HAProxy) |
| **Stockage** | NAS / iSCSI | `NTE-NAS-001` |
| **Backup** | Sauvegarde | `Veeam` + repo |

---

## 🧑‍🏭 Opérations – comment maintenir les schémas

### A. Où sont stockées les sources ?
- **Sources** : fichiers `.drawio` (physique & logique) + **export PNG**.  
- **Emplacement** : `/25INFNA-LICISASRS-1AN/Entreprise Pédagogique/BeSafe IT`  
  - `schema-physique-besafe.drawio`  
  - `schema-logique-besafe.drawio`  

### B. Comment mettre à jour (draw.io)
1. Ouvrir le `.drawio` correspondant.  
2. Mettre à jour équipements/labels/ports/VLAN **et** la **légende**.  
3. **Exporter** : `Fichier → Exporter → PNG (transparent)`, largeur 1920 px min.  
4. Uploader le PNG dans Wiki.js (dossier `_assets`) et **remplacer le lien** de l’image.  
5. Incrémenter la version (ci-dessous) + ajouter une **entrée de change**.

### C. Règles de formalisme
- Nommage homogène : `NTE-<ROLE>-<###>` (ex. `NTE-SRV-001`).  

---

## ✅ Checklist validation

- [ ] Tous les **rôles** sont visibles (AD, PKI, vCenter, Apps, Proxy, Wazuh, Backup).  
- [ ] Les **VLAN/Segments** utilisés dans le schéma existent dans le **Plan IP**.  
- [ ] Les **sources `.drawio`** sont versionnées et l’export PNG est à jour.  


