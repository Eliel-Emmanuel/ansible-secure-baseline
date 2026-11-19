# Phase 1 — Une structure professionnelle

## Objectif

En phase 0, Ansible savait **se connecter** aux serveurs. En phase 1, je voulais qu'il les **configure**, avec un code organisé comme dans une vraie équipe : réutilisable, paramétrable selon l'environnement, sans aucun secret en clair, et qui donne le même résultat à chaque exécution.

Le critère de réussite : détruire toutes les VM, les recréer, lancer un seul playbook, et obtenir des serveurs entièrement configurés ; puis un second passage qui ne modifie rien.

```text
# Premier passage, sur des VM neuves
web01 : ok=12   changed=10   unreachable=0   failed=0

# Second passage
web01 : ok=11   changed=0    unreachable=0   failed=0
```

## Comment j'ai avancé

Avant de toucher au projet, j'ai suivi quatre cours, chacun terminé par un exercice dans un dossier séparé, exclu de Git :

1. **Playbooks, modules et idempotence** : décrire un état voulu plutôt qu'une suite de commandes, et voir concrètement pourquoi une tâche `shell` casse l'idempotence.
2. **Les variables et leur priorité** : `group_vars`, `host_vars`, faits, `register`, et l'ordre dans lequel Ansible choisit une valeur.
3. **Rôles, handlers et templates** : un rôle `chrony` complet, avec un handler qui ne redémarre le service que si sa configuration change.
4. **ansible-vault et les tags** : chiffrer un secret, vérifier qu'il apparaît quand même dans un `--diff`, puis le masquer avec `no_log`.

J'ai ensuite construit la phase en cinq étapes, chacune testée puis commitée :

| Étape | Contenu |
| --- | --- |
| 1.1 | Inventaires `dev` et `prod`, variables et vault |
| 1.2 | Rôle `common` : paquets de base, fuseau horaire et chrony |
| 1.3 | Comptes d'administration, clés SSH et `sudo` par environnement |
| 1.4 | Message d'accueil généré par un template |
| 1.5 | Reconstruction complète depuis des VM vierges, version `v0.1.0` |

## Ce que j'ai construit

### Deux environnements, un seul code

```text
inventories/
├── dev/                     # par défaut, dans ansible.cfg
│   ├── hosts.yml
│   └── group_vars/all/
│       ├── vars.yml         # en clair
│       └── vault.yml        # chiffré
└── prod/                    # même structure
```

Le même playbook s'applique aux deux environnements ; seules les valeurs changent. `dev` reste l'inventaire par défaut : viser la production demande toujours un `-i inventories/prod/hosts.yml` explicite.

Dans mon labo, les deux inventaires visent les **mêmes VM**. C'est une simulation : elle prouve que la structure gère plusieurs environnements, mais tester `prod` modifie réellement les serveurs, et il faut ensuite rejouer `dev`.

### Le rôle `common`

Le rôle est découpé en un fichier de tâches par thème, importés par `tasks/main.yml` avec un tag chacun :

| Fichier | Tag | Contenu |
| --- | --- | --- |
| `paquets.yml` | `paquets` | Paquets de base |
| `temps.yml` | `temps` | Fuseau horaire, chrony et son handler |
| `comptes.yml` | `comptes` | Groupe `admins`, comptes, clés SSH, droits `sudo` |
| `motd.yml` | `motd` | Message d'accueil |

`ansible-playbook playbooks/site.yml --tags temps` n'applique ainsi que la partie heure, en quelques secondes.

### Une différence réelle entre `dev` et `prod`

Mon compte d'administration personnel est créé par Ansible, avec ma propre clé SSH. Son `sudo` dépend de l'environnement :

```text
# dev                                   # prod
%admins ALL=(ALL:ALL) NOPASSWD: ALL     %admins ALL=(ALL:ALL) ALL
```

La différence ne vient ni du rôle ni de la tâche, mais d'une seule variable, `common_sudo_sans_mot_de_passe`, lue par une condition dans le template. Le `--diff` d'Ansible montre exactement cette ligne quand on passe d'un environnement à l'autre.

## Les choix techniques, et pourquoi

**Des variables préfixées par le nom du rôle.** Toutes les variables de `common` commencent par `common_`. Quand d'autres rôles arriveront, deux rôles ne pourront pas utiliser le même nom de variable par accident.

**Sûr par défaut.** Dans les `defaults` du rôle, `sudo` exige un mot de passe et la liste des comptes est vide. C'est l'environnement `dev` qui choisit explicitement d'assouplir. Une variable oubliée donne donc la configuration la plus stricte, jamais la plus permissive.

**Les secrets séparés des variables.** `vars.yml` reste lisible et montre quelles variables existent ; `vault.yml` ne contient que les valeurs secrètes, chiffrées en AES-256. Les mots de passe sont différents en `dev` et en `prod` : la fuite du premier n'ouvrirait pas la production. J'ai vérifié qu'ils différaient en comparant leurs empreintes SHA-256, sans jamais les afficher.

**Le mot de passe du vault hors du dépôt.** Il se trouve dans `~/.vault_pass` sur `ctrl`, avec des droits `600`, en dehors du dossier partagé. Une erreur de `.gitignore` ne peut donc pas le publier. Il est aussi conservé dans un gestionnaire de mots de passe, puisqu'un `vagrant destroy` efface `ctrl`.

**`no_log` sur les tâches sensibles.** Le vault protège les secrets dans le dépôt, mais pas dans la sortie d'Ansible : pendant le cours, j'ai vu un secret apparaître en clair dans un `--diff`. La tâche qui crée les comptes masque donc toute sa sortie.

**`validate` sur le fichier `sudo`.** Ansible génère le fichier, le fait vérifier par `visudo -cf`, et ne l'installe que si la vérification réussit. Une faute de syntaxe dans un fichier sudoers peut bloquer tout `sudo` sur une machine ; avec `validate`, elle n'atteint jamais le serveur. Le compte `ansible`, géré par Vagrant, garde de toute façon son propre accès : Ansible peut toujours corriger une erreur sur les comptes humains.

**`exclusive: true` pour les clés SSH.** Le fichier `authorized_keys` de chaque administrateur ne contient que la clé déclarée. Une clé ajoutée à la main est retirée au passage suivant : personne ne peut se créer un accès caché.

**Une clé personnelle dédiée au labo.** Ma clé `id_ed25519` habituelle sert déjà ailleurs ; j'ai créé une clé séparée, protégée par une phrase de passe, et je l'ai déclarée dans le fichier `config` d'OpenSSH pour les seules adresses du labo, avec `IdentitiesOnly yes`.

**Une variable du rôle pour l'environnement.** Le message d'accueil affiche l'environnement, mais le rôle ne dépend pas directement de la variable `environnement` de l'inventaire : il utilise `common_motd_environnement`, qui la reprend avec une valeur de secours. Le rôle reste utilisable dans un projet où cette variable n'existe pas.

**Aucune valeur changeante dans les templates.** Le message d'accueil ne contient que des informations stables : nom, environnement, groupes, système. Une date de déploiement rendrait le fichier différent à chaque passage, et le rôle ne serait plus idempotent.

**Le fuseau horaire passé de Vagrant à Ansible.** En phase 0, le Vagrantfile le réglait. C'est maintenant le rôle `common` : Vagrant ne prépare plus que l'accès aux machines, et toute la configuration vit dans le code Ansible.

**Toutes les dépendances écrites dans le dépôt.** `passlib` a été ajouté à `requirements-dev.txt`, même si ma version d'Ansible savait s'en passer : une dépendance installée à la main disparaîtrait à la prochaine reconstruction de `ctrl`.

## Les problèmes rencontrés et comment je les ai résolus

### Une variable introuvable

Le premier lancement des comptes a échoué avec `'common_groupe_admins' is undefined`, en indiquant le fichier, la ligne et la colonne. J'avais déclaré les comptes dans l'inventaire, mais pas les valeurs par défaut du rôle. Un rôle doit fonctionner même si l'inventaire ne définit pas toutes ses variables : chacune a maintenant sa valeur dans `defaults/main.yml`.

### Le fichier de configuration SSH ignoré

Mon fichier `config` d'OpenSSH n'était pas lu : je l'avais placé dans le dossier du projet, alors que SSH ne le cherche que dans `C:\Users\<moi>\.ssh\`. `ssh -G`, qui affiche la configuration effective sans se connecter, l'a montré immédiatement. Une fois déplacé, SSH l'a refusé : `Move-Item` avait conservé les droits du dossier d'origine, modifiable par tous les utilisateurs authentifiés. `icacls` a restreint les droits à mon seul compte. Ce fichier n'a jamais été commité.

### Le piège de `update_password: on_create`

En testant `prod`, `sudo` m'a demandé un mot de passe, et c'est celui de `dev` qui fonctionnait. Le compte avait été créé en `dev`, et `on_create` n'applique le mot de passe qu'à la création. C'est le prix de l'idempotence (voir les limites), et c'est maintenant documenté dans le README du rôle.

### Un dossier VirtualBox resté après `vagrant destroy`

Pendant la reconstruction, VirtualBox a refusé de recréer `asb-ctrl` : le dossier de l'ancienne VM n'avait pas été entièrement supprimé, probablement parce que l'interface graphique de VirtualBox avait encore un fichier ouvert. J'ai supprimé le clone interrompu avec `VBoxManage unregistervm --delete`, puis le dossier restant. Depuis, je ferme l'interface de VirtualBox avant un `vagrant destroy`.

## Une découverte de sécurité : les mêmes clés d'hôte partout

Après la reconstruction, SSH m'a présenté pour web01 **la même empreinte** qu'en phase 0, alors que la VM venait d'être recréée. En comparant les trois serveurs :

```text
web01 | 256 SHA256:VjufUgJC...Enlc root@vagrant (ED25519)
db01  | 256 SHA256:VjufUgJC...Enlc root@vagrant (ED25519)
mon01 | 256 SHA256:VjufUgJC...Enlc root@vagrant (ED25519)
```

Les clés d'hôte SSH ont été générées une seule fois, à la fabrication de la box (`root@vagrant`), et chaque VM clonée en hérite. Toutes mes machines ont donc la même identité SSH : n'importe laquelle pourrait se faire passer pour une autre sans déclencher d'avertissement. Le risque est nul dans un labo fermé, mais c'est un vrai défaut d'image qu'on rencontre en entreprise.

La correction est prévue au début de la phase 2, dans le **Vagrantfile** plutôt que dans Ansible : il faut régénérer les clés au premier démarrage, avant qu'Ansible ne s'y connecte. Si Ansible le faisait lui-même, ses connexions suivantes seraient bloquées par le changement d'empreinte. C'est le rôle que joue cloud-init sur une instance cloud.

## Vérifications

| Test | Résultat attendu | Obtenu |
| --- | --- | --- |
| Variables par environnement | `environnement` vaut `dev` ou `prod` selon l'inventaire | ✅ |
| Vault lu sans saisie de mot de passe | `admin_mot_de_passe` déchiffré via `vault_password_file` | ✅ |
| Vaults chiffrés dans le dépôt | `$ANSIBLE_VAULT;1.1;AES256` en tête des deux fichiers | ✅ |
| Mots de passe distincts | Empreintes SHA-256 différentes entre `dev` et `prod` | ✅ |
| Tags | `--tags temps` n'exécute que les tâches de l'heure | ✅ |
| Handler | chrony redémarré seulement quand sa configuration change | ✅ |
| Connexion personnelle | `ssh` avec ma clé, compte membre de `admins` | ✅ |
| `sudo` en `dev` | `sudo -n true` réussit | ✅ |
| `sudo` en `prod` | `sudo -n true` échoue, mot de passe demandé | ✅ |
| Message d'accueil | Serveur, environnement et groupes corrects ; avertissement en `prod` | ✅ |
| Reconstruction depuis zéro | `failed=0` sur trois VM neuves | ✅ |
| Idempotence | Second passage avec `changed=0` | ✅ |

## Limites assumées

- `dev` et `prod` visent les mêmes VM : la séparation des environnements est simulée.
- Un même mot de passe de vault chiffre `dev` et `prod`. En entreprise, j'utiliserais des identifiants de vault distincts (`--vault-id`), pour que l'accès à `dev` ne donne pas accès aux secrets de production.
- Le mot de passe d'un compte n'est appliqué qu'à sa création.
- Le compte `ansible` garde un `sudo` sans mot de passe dans les deux environnements, pour qu'Ansible puisse toujours intervenir.
- Les clés d'hôte SSH sont identiques sur toutes les VM, jusqu'à la correction prévue en phase 2.

## Et ensuite

La phase 2 durcit les serveurs : régénération des clés d'hôte, configuration de SSH, pare-feu, fail2ban, auditd et paramètres noyau, avec un score Lynis mesuré avant et après.
