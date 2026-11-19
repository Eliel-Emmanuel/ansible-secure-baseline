# Rôle `common`

Socle commun appliqué à **tous les serveurs** du projet, avant leur rôle spécifique :

- installation des paquets de base ;
- fuseau horaire et synchronisation de l'heure avec chrony ;
- comptes d'administration, avec leur clé SSH et leurs droits `sudo` ;
- message d'accueil indiquant le serveur et son environnement.

## Prérequis

- Ubuntu 24.04 (Noble)
- Les collections `community.general` (module `timezone`) et `ansible.posix` (module `authorized_key`), listées dans `requirements.yml`
- La bibliothèque Python `passlib` sur le contrôleur, pour calculer l'empreinte des mots de passe (listée dans `requirements-dev.txt`)
- Un accès à Internet en UDP sur le port 123, pour joindre les serveurs de temps

## Variables

| Variable | Valeur par défaut | Description |
| --- | --- | --- |
| `common_paquets` | `curl`, `vim`, `htop`, `tree`, `jq`, `unzip`, `ca-certificates` | Paquets installés sur tous les serveurs |
| `common_fuseau_horaire` | `Africa/Lagos` | Fuseau horaire des serveurs |
| `common_ntp_serveurs` | `0`, `1` et `2.pool.ntp.org` | Serveurs de temps utilisés par chrony |
| `common_admins` | `[]` | Liste des comptes d'administration (voir ci-dessous) |
| `common_groupe_admins` | `admins` | Groupe qui donne les droits `sudo` |
| `common_sudo_sans_mot_de_passe` | `false` | `true` pour que `sudo` ne demande pas de mot de passe |
| `common_motd_environnement` | la variable `environnement` de l'inventaire, sinon `non défini` | Environnement affiché dans le message d'accueil |

Le rôle est **sûr par défaut** : sans réglage explicite, `sudo` exige un mot de passe, et aucun compte n'est créé.

### Déclarer les comptes d'administration

Chaque administrateur est un élément de la liste `common_admins` :

```yaml
common_admins:
  - nom: emmanuel
    commentaire: "Emmanuel Apovo"
    mot_de_passe: "{{ admin_mot_de_passe }}"    # valeur chiffrée dans le vault
    cle_ssh: "ssh-ed25519 AAAA... emmanuel@labo-ansible"
```

Le mot de passe sert à `sudo` quand `common_sudo_sans_mot_de_passe` vaut `false`. La connexion SSH se fait uniquement par clé.

## Tags

| Tag | Partie du rôle |
| --- | --- |
| `paquets` | Installation des paquets de base |
| `temps` | Fuseau horaire et chrony |
| `comptes` | Groupe, comptes, clés SSH et droits `sudo` |
| `motd` | Message d'accueil |

Exemple : `ansible-playbook playbooks/site.yml --tags comptes` n'applique que la partie comptes.

## Dépendances

Aucune dépendance de rôle.

## Exemple d'utilisation

```yaml
- name: Appliquer le socle commun à tous les serveurs
  hosts: all
  become: true
  roles:
    - role: common
      tags: common
```

Pour changer une valeur, la définir dans l'inventaire plutôt que dans le rôle, par exemple dans `inventories/prod/group_vars/all/vars.yml`.

## Comportement

- Les fichiers `/etc/chrony/chrony.conf`, `/etc/sudoers.d/admins` et `/etc/motd` sont entièrement gérés par ce rôle : toute modification manuelle sera écrasée au prochain passage.
- chrony n'est redémarré que si sa configuration change.
- Le fichier `sudo` est vérifié avec `visudo` avant d'être installé : une configuration invalide n'est jamais mise en place.
- `authorized_keys` ne contient que les clés déclarées : une clé ajoutée à la main est retirée au passage suivant.
- Les mots de passe ne sont jamais affichés dans la sortie d'Ansible (`no_log`).
- Le rôle est idempotent : un second passage ne modifie rien.

## Limites connues

- **Le mot de passe d'un compte n'est appliqué qu'à sa création** (`update_password: on_create`). Le filtre `password_hash` produit une empreinte différente à chaque calcul ; sans cette option, le mot de passe serait réécrit à chaque passage et le rôle ne serait plus idempotent. Changer le mot de passe dans le vault ne modifie donc pas un compte existant.
- Le message d'accueil affiche aussi les lignes `# Ansible managed`, car `/etc/motd` est affiché tel quel. C'est un choix : elles rappellent à l'administrateur que le fichier est géré par Ansible.

## Licence

MIT

## Auteur

Emmanuel Apovo
