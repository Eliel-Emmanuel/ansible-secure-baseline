# ansible-secure-baseline

Un socle d'infrastructure Linux **durci, supervisé, sauvegardé et testé automatiquement**, entièrement décrit en code avec Ansible.

J'ai construit ce projet pour apprendre Ansible en profondeur et pour montrer comment je travaille : un labo que n'importe qui peut recréer en quelques commandes, du code testé, et une documentation honnête sur ce qui a marché du premier coup et ce qui m'a demandé du diagnostic.

> **Projet de laboratoire.** Tout tourne en local, sur des machines virtuelles. Ce n'est pas un déploiement de production, et les choix propres au labo sont signalés dans la documentation.

## Architecture du labo

Quatre machines virtuelles Ubuntu Server 24.04, créées par Vagrant sur VirtualBox, reliées par un réseau privé `192.168.56.0/24` :

| VM | Adresse | RAM | Rôle |
| --- | --- | --- | --- |
| `ctrl` | 192.168.56.20 | 1 Go | Contrôleur : Ansible, Molecule, Docker |
| `web01` | 192.168.56.21 | 768 Mo | Serveur web (Nginx et application) |
| `db01` | 192.168.56.22 | 768 Mo | Base de données (PostgreSQL) |
| `mon01` | 192.168.56.23 | 1,5 Go | Supervision (Prometheus, Grafana, Loki) |

Ansible tourne sur `ctrl` et se connecte aux trois serveurs en SSH, avec un compte de service `ansible` et une clé dédiée au labo.

## Prérequis

- Windows 10 ou 11 (le labo devrait aussi fonctionner sous Linux ou macOS sur processeur Intel, mais je ne l'ai testé que sous Windows)
- [VirtualBox](https://www.virtualbox.org/) 7.2 (testé avec 7.2.12)
- [Vagrant](https://developer.hashicorp.com/vagrant) 2.4.9 ou plus récent
- Git et le client OpenSSH de Windows
- Environ **4 Go de RAM libres** pour les quatre VM, et 60 Go d'espace disque
- La virtualisation matérielle (VT-x) accessible à VirtualBox : sous Windows, la sécurité basée sur la virtualisation (VBS) et Hyper-V doivent être désactivés, sinon les VM sont extrêmement lentes. Le détail est dans [docs/phase-0.md](docs/phase-0.md).

## Démarrage rapide

### 1. Créer les machines

```powershell
git clone https://github.com/Eliel-Emmanuel/ansible-secure-baseline.git
cd ansible-secure-baseline
vagrant up ctrl web01 db01
vagrant up mon01
vagrant ssh ctrl
```

Les VM démarrent en deux temps pour ménager la mémoire du poste. Dans `ctrl`, l'environnement Ansible est déjà activé et on se trouve dans le dossier du projet.

### 2. Fournir le mot de passe du vault

Les secrets du projet sont chiffrés avec ansible-vault. Ansible lit le mot de passe du vault dans `~/.vault_pass`, sur `ctrl`, **en dehors** du dossier du projet :

```bash
printf '%s\n' 'MOT-DE-PASSE-DU-VAULT' > ~/.vault_pass
chmod 600 ~/.vault_pass
```

C'est la seule action manuelle après une reconstruction, et elle est volontaire : un secret ne doit jamais se trouver dans le dépôt.

Les vaults de ce dépôt sont chiffrés avec mon propre mot de passe. Pour reproduire le labo chez vous, remplacez-les par les vôtres :

```bash
rm inventories/dev/group_vars/all/vault.yml inventories/prod/group_vars/all/vault.yml
ansible-vault create inventories/dev/group_vars/all/vault.yml     # vault_password: "..."
ansible-vault create inventories/prod/group_vars/all/vault.yml    # vault_password: "..."
```

Pensez aussi à remplacer la clé SSH publique de `common_admins` par la vôtre, dans les deux fichiers `vars.yml`.

### 3. Appliquer le socle

```bash
ansible all -m ping
ansible-playbook playbooks/site.yml
```

## Utilisation

| Commande | Effet |
| --- | --- |
| `ansible-playbook playbooks/site.yml` | Applique le socle en **dev** (l'inventaire par défaut) |
| `ansible-playbook -i inventories/prod/hosts.yml playbooks/site.yml` | Applique le socle en **prod** |
| `ansible-playbook playbooks/site.yml --tags temps` | N'applique qu'une partie du socle |
| `ansible-playbook playbooks/site.yml --check --diff` | Montre ce qui changerait, sans rien modifier |
| `ansible-playbook playbooks/site.yml --list-tags` | Liste les tags disponibles |

Tags disponibles : `common`, `paquets`, `temps`, `comptes`, `motd`.

L'inventaire `dev` est celui par défaut : viser la production doit toujours être un choix explicite.

## Ce que fait le Vagrantfile

Vagrant ne fait que préparer les machines ; toute la configuration est laissée à Ansible.

- Il génère une fois pour toutes une paire de clés SSH du labo, dans `vagrant/keys/` (exclu de Git).
- Il crée sur `web01`, `db01` et `mon01` un compte `ansible` qui n'accepte que cette clé et peut utiliser `sudo`.
- Il renseigne `/etc/hosts` sur chaque VM pour que les machines se joignent par leur nom.
- Il installe sur `ctrl` Ansible, ansible-lint, Molecule et yamllint dans un environnement virtuel Python, ainsi que Docker pour les tests.
- Il partage le dossier du projet avec `ctrl` uniquement : j'écris le code sous Windows et je l'exécute dans la VM.

## Le socle commun (rôle `common`)

Appliqué à tous les serveurs, il installe les paquets de base, règle le fuseau horaire, synchronise l'heure avec chrony, crée les comptes d'administration avec leur clé SSH et leurs droits `sudo`, et affiche à la connexion un message indiquant le serveur et son environnement. Le détail est dans [roles/common/README.md](roles/common/README.md).

Le même code se comporte différemment selon l'environnement : par exemple, `sudo` ne demande pas de mot de passe en `dev`, mais en exige un en `prod`.

## Avancement

| Phase | Contenu | État |
| --- | --- | --- |
| 0 | Labo reproductible (Vagrant, réseau, SSH, contrôleur Ansible) | ✅ Terminée — [docs/phase-0.md](docs/phase-0.md) |
| 1 | Structure professionnelle : inventaires, variables, vault, rôle `common` | ✅ Terminée (`v0.1.0`) — [docs/phase-1.md](docs/phase-1.md) |
| 2 | Durcissement des serveurs, mesuré avec Lynis | À venir |
| 3 | Services et exploitation : application, PostgreSQL, supervision, sauvegardes | À venir |
| 4 | Qualité et intégration continue : Molecule, ansible-lint, GitHub Actions | À venir |
| 5 | Collection Ansible et niveau avancé | À venir |
| 6 | Gestion d'un Windows Server (Active Directory) via WinRM | À venir |

## Structure actuelle du dépôt

```text
ansible-secure-baseline/
├── Vagrantfile                  # Les quatre VM du labo
├── ansible.cfg                  # Configuration d'Ansible
├── requirements-dev.txt         # Outils Python installés sur ctrl
├── requirements.yml             # Collections Ansible
├── inventories/
│   ├── dev/                     # Environnement de développement (par défaut)
│   │   ├── hosts.yml
│   │   └── group_vars/all/      # vars.yml (en clair) et vault.yml (chiffré)
│   └── prod/                    # Environnement de production
├── playbooks/
│   └── site.yml                 # Playbook principal
├── roles/
│   └── common/                  # Socle appliqué à tous les serveurs
└── docs/                        # Une page par phase
```

## Sécurité

- Aucune clé privée ni aucun mot de passe n'est versionné en clair : `vagrant/keys/` est exclu par `.gitignore`, et les secrets sont chiffrés avec ansible-vault.
- Le mot de passe du vault est stocké hors du dépôt, sur `ctrl`, avec des droits `600`.
- Les tâches qui manipulent un secret utilisent `no_log: true`, pour qu'il n'apparaisse jamais dans la sortie d'Ansible.
- Les droits `sudo` sont vérifiés par `visudo` avant d'être installés.
- La clé du labo n'a pas de phrase de passe, pour qu'Ansible tourne sans intervention. C'est acceptable dans un labo isolé, pas en production.
