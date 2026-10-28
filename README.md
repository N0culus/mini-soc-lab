# Mini SOC / Purple Team Lab

Laboratoire isolé pour **simuler des attaques** (Atomic Red Team), **les détecter** (Wazuh + Sysmon) et **documenter** ce qui est vu, raté, puis corrigé.

> ⚠️ Lab 100 % isolé : réseau interne uniquement, aucune exposition à Internet ni au réseau personnel. Les tests d'attaque ne sont lancés que sur mes propres machines virtuelles.

## Objectif

Relier attaque et défense sur un même environnement :
1. Exécuter des techniques MITRE ATT&CK connues sur un poste Windows d'un domaine AD
2. Vérifier si Wazuh les détecte
3. Écrire ou ajuster des règles pour combler les angles morts
4. Documenter chaque test (technique, résultat, règle, leçon)

## Architecture

| VM | Rôle | vCPU | RAM | Disque |
|---|---|---|---|---|
| `soc-wazuh` | Wazuh all-in-one (manager, indexer, dashboard) sur Ubuntu Server | 4 | 8 Go | 60 Go |
| `dc01` | Windows Server, contrôleur de domaine AD | 2 | 4 Go | 60 Go |
| `ws01` | Windows 10/11, poste cible (Sysmon, agent Wazuh, Atomic Red Team) | 2 | 4 Go | 60 Go |
| `lnx01` | Ubuntu Server, cible Linux | 1 | 2 Go | 25 Go |
| `kali` | Machine d'attaque manuelle (optionnelle, à allumer au besoin) | 2 | 4 Go | 50 Go |

Total : environ 22 Go de RAM et 255 Go de disque en provisionnement léger (thin provisioning). Sur 32 Go de RAM hôte, il reste de quoi faire tourner Ubuntu et un navigateur.

**Réseau :** un seul réseau virtuel isolé (par exemple `10.10.10.0/24`), sans accès Internet une fois les installations terminées.

## Étapes

### 0. Préparation de l'hôte
- Vérifier la virtualisation : `egrep -c '(vmx|svm)' /proc/cpuinfo` (doit être > 0) puis `kvm-ok`
- Installer KVM/libvirt et virt-manager
- Créer le réseau virtuel isolé
- Télécharger les ISO d'évaluation Microsoft (Evaluation Center, 180 jours) et l'ISO Ubuntu Server

### 1. Wazuh
- Installer Wazuh en all-in-one sur `soc-wazuh` en suivant la documentation officielle (le script d'installation assisté)
- Noter l'adresse du dashboard et le mot de passe admin généré
- Figer un snapshot : `01-wazuh-propre`

### 2. Active Directory
- Installer le rôle AD DS sur `dc01`, créer le domaine (ex. `lab.local`)
- Créer quelques utilisateurs de test et un compte admin de lab
- Snapshot : `02-ad-propre`

### 3. Poste cible
- Joindre `ws01` au domaine
- Installer **Sysmon** avec une configuration de référence reconnue (SwiftOnSecurity ou Olaf Hartong)
- Installer l'**agent Wazuh** et l'enregistrer auprès du manager
- Configurer la collecte du journal `Microsoft-Windows-Sysmon/Operational` dans l'agent
- Vérifier que les événements arrivent dans le dashboard
- Snapshot : `03-ws01-supervise`

### 4. Atomic Red Team
- Installer `Invoke-AtomicRedTeam` sur `ws01`
- Ajouter une exclusion Defender sur le dossier Atomic Red Team (documenter ce choix : en production, ce dossier serait un faux positif massif)
- Snapshot **avant** chaque série de tests pour pouvoir revenir en arrière

### 5. Premiers tests à documenter
À vérifier avec `Invoke-AtomicTest <ID> -ShowDetailsBrief` avant exécution :

| Technique | Description |
|---|---|
| T1059.001 | Exécution PowerShell |
| T1053.005 | Tâche planifiée |
| T1547.001 | Persistance par clé de registre Run |
| T1136.001 | Création de compte local |
| T1003.001 | Accès à la mémoire de LSASS |
| T1110 | Force brute |

Pour chaque test, remplir une fiche dans `docs/tests/`.

### 6. Detection engineering
- Repérer les tests **non détectés** ou mal classés
- Écrire des règles Wazuh personnalisées (`local_rules.xml`) et les tester
- Mesurer avant/après

### 7. Publication
- Schéma d'architecture, captures du dashboard, fiches de test
- Article sur la page Veille du portfolio

## Structure du dépôt

```
mini-soc-lab/
├── README.md
├── docs/
│   ├── architecture.md
│   ├── installation.md
│   └── tests/
│       ├── T1059.001-powershell.md
│       └── ...
├── configs/
│   ├── sysmon/
│   └── wazuh/
│       └── local_rules.xml
└── img/
```

## Modèle de fiche de test

```markdown
# T1059.001 : Exécution PowerShell

- **Date :**
- **Machine :** ws01
- **Commande exécutée :**
- **Détecté par Wazuh ?** Oui / Non / Partiellement
- **Règle déclenchée :** (ID et niveau)
- **Ce qui manquait :**
- **Règle ajoutée :**
- **Nouveau résultat :**
- **Leçon retenue :**
```

## Règles d'hygiène du dépôt
- Ne jamais publier de mots de passe, clés ou adresses réelles
- Ne publier que des configurations de lab
- Mentionner clairement que tout s'exécute en environnement isolé

## Ressources
- Documentation officielle Wazuh
- MITRE ATT&CK
- Documentation d'Atomic Red Team
- Configurations Sysmon de référence
