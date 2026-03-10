# GarageHQ — Scripts d'installation automatisée

Scripts Python pour déployer et gérer [Garage](https://garagehq.deuxfleurs.fr/), un serveur de stockage objet compatible S3, conçu pour l'auto-hébergement.

---

## Contenu

```
GarageHQ/Install/
├── garage.py       # Installation, désinstallation et réinstallation de Garage (orienté objet)
├── install.py      # Version standard/procédurale de garage.py
├── storage.py      # Configuration du stockage SAN FC (multipath + LVM)
└── requirements.txt
```

---

## Prérequis

- Linux Debian/Ubuntu
- Python 3.10+
- Droits root (`sudo` ou `root`)
- Accès internet pour télécharger les binaires

```bash
pip install -r requirements.txt
```

**`requirements.txt`**
```
requests
```

---

## garage.py — Gestionnaire Garage

Installe, désinstalle ou réinstalle Garage ainsi que l'interface web optionnelle `garage-webui`.

### Fonctionnalités

- Détection automatique de la **dernière version stable** de Garage via l'API Gitea (tri sémantique)
- Génération automatique du `rpc_secret` et de l'`admin_token`
- Configuration TOML adaptée selon la version (`replication_mode` v1.x vs `replication_factor` v2.x+)
- Création de l'utilisateur système `garage`
- Configuration du layout du cluster (zone, capacité)
- Création d'une clé d'accès S3
- Installation optionnelle de **garage-webui** avec authentification bcrypt
- Désinstallation propre avec option de conservation des données

### Usage

```bash
# Installation complète
sudo python3 garage.py --install

# Désinstallation (supprime tout)
sudo python3 garage.py --uninstall

# Désinstallation en conservant les données
sudo python3 garage.py --uninstall --keep-data

# Réinstallation complète (supprime et réinstalle)
sudo python3 garage.py --reinstall

# Réinstallation en conservant les données
sudo python3 garage.py --reinstall --keep-data
```

### Déroulement de l'installation

Le script pose les questions suivantes de manière interactive :

| Étape | Question |
|-------|----------|
| 1 | Chemin du répertoire de données |
| 2 | Chemin du répertoire de métadonnées |
| 3 | Détection automatique ou saisie manuelle de l'IP |
| 4 | Zone/datacenter (ex: `dc1`) |
| 5 | Capacité en Go |
| 6 | Nom de la clé d'accès S3 |
| 7 | Installer garage-webui ? (optionnel) |
| 8 | Activer l'authentification webui ? (optionnel) |

### Ports utilisés

| Port | Rôle |
|------|------|
| `3900` | API S3 |
| `3901` | RPC inter-nœuds |
| `3902` | Hébergement web statique |
| `3903` | API d'administration |
| `3909` | Interface web garage-webui (si installée) |

### Résumé affiché en fin d'installation

```
============================================================
INSTALLATION TERMINEE
============================================================
  Version Garage : v2.2.0
  Config         : /etc/garage/garage.toml
  Donnees        : /data
  Metadonnees    : /data/meta
  RPC Secret     : <généré automatiquement>
  Admin Token    : <généré automatiquement>
  Access Key ID  : GKxxxxxxxxxxxx
  Secret Key     : xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

  API S3         : http://192.168.1.10:3900
  Web statique   : http://192.168.1.10:3902
  Admin API      : http://192.168.1.10:3903
  Interface Web  : http://192.168.1.10:3909
```

### Commandes utiles après installation

```bash
# Voir les logs
journalctl -xeu garage -n 20
journalctl -xeu garage-webui -n 20

# Créer un bucket
garage -c /etc/garage/garage.toml bucket create mon-bucket

# Autoriser une clé sur un bucket
garage -c /etc/garage/garage.toml bucket allow \
  --read --write --key admin-key mon-bucket

# Voir le statut du cluster
garage -c /etc/garage/garage.toml status
```

---

## storage.py — Configuration stockage SAN FC

Configure le stockage SAN Fibre Channel avec multipath et LVM avant d'installer Garage.

### Fonctionnalités

- Détection automatique des LUNs FC (`multipath -ll`, `lsblk`, fallback manuel)
- Configuration de `multipathd` avec les WWIDs des LUNs sélectionnés
- Création PV → VG → LV avec **striping LVM** optionnel sur plusieurs LUNs
- Formatage en `xfs`, `ext4` ou `btrfs`
- Montage et persistance automatique dans `/etc/fstab`
- Affichage du statut complet (multipath, LVM, FS, HBA FC)

### Usage

```bash
# Configurer le stockage SAN FC
sudo python3 storage.py --setup

# Voir le statut
sudo python3 storage.py --status

# Supprimer le stockage (avec confirmation)
sudo python3 storage.py --teardown

# Supprimer sans effacer les données
sudo python3 storage.py --teardown --keep-data
```

### Ordre recommandé de déploiement

```bash
# 1. Préparer le stockage SAN FC
sudo python3 storage.py --setup

# 2. Installer Garage sur les volumes créés
sudo python3 garage.py --install
```

### Dépendances système

Installées automatiquement si absentes :

```bash
apt install multipath-tools lvm2 xfsprogs btrfs-progs e2fsprogs
```

### Architecture de la stack stockage

```
SAN (FC)
   │
   ▼
multipathd          ← agrège les chemins FC redondants
   │
   ▼
LVM (PV → VG → LV) ← gestion des volumes logiques
   │                   striping optionnel sur plusieurs LUNs
   ▼
Filesystem          ← xfs (données) / ext4 (métadonnées)
   │
   ▼
/data               ← data_dir Garage
/data/meta          ← metadata_dir Garage
```

### Filesystems recommandés

| Volume | Filesystem recommandé | Raison |
|--------|-----------------------|--------|
| Données (`data_dir`) | `xfs` | Hautes performances, grands fichiers |
| Métadonnées (`metadata_dir`) | `ext4` ou `btrfs` | Stabilité LMDB, snapshots BTRFS utiles |

---

## Architecture du code

### garage.py — Architecture orientée objet (POO)

`garage.py` est conçu selon une architecture orientée objet stricte : chaque responsabilité est encapsulée dans une classe dédiée, ce qui facilite la lisibilité, la maintenance et l'extensibilité du code.

| Classe | Rôle |
|--------|------|
| `InstallConfig` | Dataclass centralisant toute la configuration |
| `Utils` | Méthodes statiques partagées (validation, subprocess, prompts) |
| `ConfigCollector` | Collecte interactive de la configuration |
| `GarageInstaller` | Installation en 7 étapes numérotées |
| `GarageUninstaller` | Désinstallation propre en 5 étapes |
| `GarageReinstaller` | Orchestration uninstall + collect + install |

### storage.py — Architecture orientée objet (POO)

`storage.py` suit la même philosophie orientée objet, avec une séparation claire entre détection, configuration et orchestration.

| Classe | Rôle |
| `StorageDetector` | Détection des LUNs FC disponibles |
| `MultipathConfigurator` | Installation et configuration de multipathd |
| `LVMConfigurator` | Création PV → VG → LV |
| `FilesystemConfigurator` | Formatage, montage et /etc/fstab |
| `StorageStatus` | Affichage de l'état complet |
| `StorageSetup` | Orchestration du setup complet |
| `StorageTeardown` | Suppression propre du stockage |

---

## Compatibilité Garage

| Version Garage | Paramètre de réplication | Webui compatible |
|----------------|--------------------------|-----------------|
| v1.x | `replication_mode = "none"` | v1.0.9 |
| v2.x+ | `replication_factor = 1` | v1.1.0+ |

Le script `garage.py` détecte automatiquement la version et adapte la configuration en conséquence.

---

## install.py — Version standard

`install.py` est la version **procédurale** de `garage.py` : elle offre exactement les mêmes fonctionnalités mais avec un code plus linéaire et sans architecture de classes. Elle est plus accessible pour qui souhaite lire ou modifier le script rapidement.

### Usage

```bash
# Installation complète
sudo python3 install.py --install

# Désinstallation (supprime tout)
sudo python3 install.py --uninstall

# Désinstallation en conservant les données
sudo python3 install.py --uninstall --keep-data

# Réinstallation complète
sudo python3 install.py --reinstall

# Réinstallation en conservant les données
sudo python3 install.py --reinstall --keep-data
```

> Les deux scripts sont interchangeables. `garage.py` est recommandé pour la maintenabilité à long terme.

---

## Licence

GPL-3.0 — voir [LICENSE](../../LICENSE) pour le texte complet.