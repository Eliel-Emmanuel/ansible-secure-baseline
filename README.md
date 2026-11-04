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

```powershell
git clone https://github.com/Eliel-Emmanuel/ansible-secure-baseline.git
cd ansible-secure-baseline
vagrant up
vagrant ssh ctrl
```

Dans `ctrl`, l'environnement Ansible est déjà activé et on se trouve dans le dossier du projet :

```bash
ansible all -m ping
```

Les trois serveurs doivent répondre `SUCCESS` avec `ping: pong`.

Sur un poste juste en mémoire, mieux vaut démarrer les VM en deux temps : `vagrant up ctrl web01 db01`, puis `vagrant up mon01`.

## Ce que fait le Vagrantfile

Vagrant ne fait que préparer les machines ; toute la configuration métier est laissée à Ansible.

- Il génère une fois pour toutes une paire de clés SSH du labo, dans `vagrant/keys/` (exclu de Git).
- Il crée sur `web01`, `db01` et `mon01` un compte `ansible` qui n'accepte que cette clé et peut utiliser `sudo`.
- Il renseigne `/etc/hosts` sur chaque VM pour que les machines se joignent par leur nom.
- Il installe sur `ctrl` Ansible, ansible-lint, Molecule et yamllint dans un environnement virtuel Python, ainsi que Docker pour les tests.
- Il partage le dossier du projet avec `ctrl` uniquement : j'écris le code sous Windows et je l'exécute dans la VM.

## Avancement

| Phase | Contenu | État |
| --- | --- | --- |
| 0 | Labo reproductible (Vagrant, réseau, SSH, contrôleur Ansible) | ✅ Terminée — [docs/phase-0.md](docs/phase-0.md) |
| 1 | Structure professionnelle : inventaires, variables, vault, rôle `common` | À venir |
| 2 | Durcissement des serveurs, mesuré avec Lynis | À venir |
| 3 | Services et exploitation : application, PostgreSQL, supervision, sauvegardes | À venir |
| 4 | Qualité et intégration continue : Molecule, ansible-lint, GitHub Actions | À venir |
| 5 | Collection Ansible et niveau avancé | À venir |
| 6 | Gestion d'un Windows Server (Active Directory) via WinRM | À venir |

## Structure actuelle du dépôt

```text
ansible-secure-baseline/
├── Vagrantfile              # Les quatre VM du labo
├── ansible.cfg              # Configuration d'Ansible
├── inventories/dev/         # Inventaire de l'environnement de dev
├── requirements-dev.txt     # Outils Python installés sur ctrl
├── requirements.yml         # Collections Ansible
└── docs/                    # Une page par phase
```

## Sécurité

- Aucune clé ni aucun secret n'est versionné : `vagrant/keys/` et le mot de passe du vault sont exclus par `.gitignore`.
- La clé du labo n'a pas de phrase de passe, pour qu'Ansible tourne sans intervention. C'est acceptable dans un labo isolé, pas en production.
