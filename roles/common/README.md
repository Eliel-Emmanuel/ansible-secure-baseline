# Rôle `common`

Socle commun appliqué à **tous les serveurs** du projet, avant leur rôle spécifique : paquets de base, fuseau horaire et synchronisation de l'heure avec chrony.

## Prérequis

- Ubuntu 24.04 (Noble)
- La collection `community.general`, pour le module `timezone` (listée dans `requirements.yml`)
- Un accès à Internet en UDP sur le port 123, pour joindre les serveurs de temps

## Variables

| Variable | Valeur par défaut | Description |
| --- | --- | --- |
| `common_paquets` | `curl`, `vim`, `htop`, `tree`, `jq`, `unzip`, `ca-certificates` | Paquets installés sur tous les serveurs |
| `common_fuseau_horaire` | `Africa/Lagos` | Fuseau horaire des serveurs |
| `common_ntp_serveurs` | `0`, `1` et `2.pool.ntp.org` | Serveurs de temps utilisés par chrony |

## Tags

| Tag | Partie du rôle |
| --- | --- |
| `paquets` | Installation des paquets de base |
| `temps` | Fuseau horaire et chrony |

Exemple : `ansible-playbook playbooks/site.yml --tags temps` n'applique que la partie heure.

## Dépendances

Aucune.

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

- Le fichier `/etc/chrony/chrony.conf` est entièrement géré par ce rôle : toute modification manuelle sera écrasée au prochain passage.
- chrony n'est redémarré que si sa configuration change.
- Le rôle est idempotent : un second passage ne modifie rien.

## Licence

MIT

## Auteur

Emmanuel Apovo
