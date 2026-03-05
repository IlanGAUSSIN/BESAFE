# 🌐 VLAN & Plan d’adressage – BESAFE

> Cette page présente le **plan IP complet de l’infrastructure BESAFE**, issu du fichier Excel `PLAN - IP - BESAFE.xlsx`.

---

## 🧩 Tableau récapitulatif des VLANs

| VLAN | Nom | Sous-réseau | CIDR |
|:---:|:----|:------------|:---:|
| 11 | **ISCSI_1** | 10.47.11.X | /24 |
| 12 | **ISCSI_2** | 10.47.12.X | /24 |
| 20 | **VMOTION** | 10.47.20.X | /24 |
| 30 | **SRV** | 10.47.30.X | /24 |
| 50 | **DMZ** | 10.47.50.X | /24 |
| 100 | **MGMT_NETWORK** | 10.47.100.X | /24 |
| 101 | **MGMT_SYSTEM** | 10.47.101.X | /24 |
| 102 | **IDRAC** | 10.47.102.X | /24 |
| 130 | **Applicatif** | 10.47.130.X | /24 |
| 200 | **MGMT_FW** | 10.47.200.X | /30 |

---


## 🔁 Corrélation VLAN ↔ Services

| VLAN | Usage principal | Services hébergés |
|:---:|:------------------|:----------------|
| 11–12 | Stockage (iSCSI) | NAS, ESXi datastores |
| 20 | vMotion / HA | Cluster VMware |
| 30 | Serveurs AD / PKI / DB | Active Directory, GLPI, Centreon, Nextcloud, Vaultwarden |
| 50 | DMZ interne | WikiJS, NPM (Reverse Proxy) |
| 100 | Management réseau | Switchs, FW, VCSA |
| 101 | Management systèmes | ESXi, NAS |
| 102 | OOB | iDRAC/ILO |
| 130 | Applicatifs | Vaultwarden, services internes |
| 200 | Admin sécurité | Firewall / Accès restreint |

---

## 📡 Remarques & Bonnes pratiques

- 🧱 **Nom standard** : `NTE-<TYPE>-<NUM>` (ex: `NTE-SRV-001`, `NTE-FW-001`).  
- 🧩 **Masque 255.255.255.0** (/24) sauf exceptions (FW en /30).  
- 🔒 VLAN management et OOB **strictement isolés** (pas de routage inter-VLAN).  
- ⚙️ Routage entre SRV, APP, DMZ via **pare-feu unique** (Stormshield).  
- 🧾 Fichier Excel maître versionné sur **Nextcloud**, révision après chaque ajout d’équipement.  


