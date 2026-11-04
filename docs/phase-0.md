# Phase 0 — Un labo reproductible

## Objectif

Avant d'écrire la moindre ligne d'Ansible, je voulais un labo que je puisse détruire et recréer à volonté, toujours à l'identique. Dans mon précédent projet (un serveur de messagerie), j'avais créé la VM à la main ; si je la cassais, tout était à refaire, et jamais exactement de la même façon.

Le critère de réussite était simple : après un `vagrant up`, Ansible doit pouvoir joindre tous les serveurs depuis `ctrl`, sans aucune action manuelle.

```bash
(ansible) vagrant@ctrl:~/projet$ ansible all -m ping
web01 | SUCCESS => ...  ping: pong
db01  | SUCCESS => ...  ping: pong
mon01 | SUCCESS => ...  ping: pong
```

## Environnement

| Élément | Version |
| --- | --- |
| Poste hôte | HP EliteBook 840 G8, Intel Core i5 de 11e génération, Windows |
| VirtualBox | 7.2.12 |
| Vagrant | 2.4.9 |
| Box | `bento/ubuntu-24.04` (Ubuntu Server 24.04 LTS) |
| Ansible | ansible-core 2.21, installé dans un environnement virtuel sur `ctrl` |

## Comment j'ai avancé

J'ai construit le Vagrantfile par petites étapes, en testant chacune avant de passer à la suivante et en faisant un commit à chaque étape validée :

1. **Le dépôt** : `.gitattributes` et `.gitignore` en tout premier, avant le moindre fichier de code.
2. **`ctrl` seule** : box, réseau privé, ressources, partage par défaut désactivé.
3. **Les trois serveurs**, générés par une boucle sur une table plutôt que par trois blocs copiés-collés.
4. **Les clés, le compte `ansible` et les noms d'hôtes.**
5. **L'outillage sur `ctrl`**, la configuration d'Ansible, l'inventaire, puis le premier ping.

Avant chaque étape, j'ai d'abord travaillé les notions sur des exercices à part : le cycle de vie d'une VM Vagrant, les deux cartes réseau de VirtualBox, et l'authentification SSH par clés.

## Les choix techniques, et pourquoi

**Deux cartes réseau par VM.** La première, en NAT, donne l'accès à Internet pour installer les paquets ; c'est aussi par elle que passe `vagrant ssh`. La seconde, sur un réseau privé hôte, permet aux VM de se parler entre elles et d'être jointes depuis Windows. En NAT seul, chaque VM est isolée derrière son propre routeur virtuel et aucune ne peut joindre les autres.

**Des clones liés.** Au lieu de copier le disque de la box pour chaque VM, VirtualBox crée une VM de base et chaque VM n'enregistre que ses différences. Les VM se créent en quelques secondes et prennent beaucoup moins de place.

**La clé du labo est générée sur Windows, par le Vagrantfile.** La même paire de clés doit aller sur plusieurs machines : la clé publique sur les trois serveurs, la clé privée sur `ctrl`. Elle doit donc exister avant les VM, dans un endroit qui survit à un `vagrant destroy`. Le dossier `vagrant/keys/` est exclu de Git.

**Un compte de service `ansible` plutôt que le compte `vagrant`.** Le compte `vagrant` n'existe que dans Vagrant. Un compte dédié, qui n'accepte que la clé du labo et n'a pas de mot de passe, c'est ce que j'utiliserais sur de vrais serveurs ; mon code Ansible n'aura pas à changer le jour où il visera autre chose que des VM.

**Des scripts de provisionnement relançables.** Chaque script peut tourner plusieurs fois avec le même résultat : le compte n'est créé que s'il n'existe pas, les anciennes lignes de `/etc/hosts` sont supprimées avant d'ajouter les nouvelles, et le fichier `authorized_keys` est réécrit entièrement plutôt que complété. Je l'ai vérifié en relançant `vagrant provision` et en contrôlant l'absence de doublons.

**Le dossier du projet n'est partagé qu'avec `ctrl`, avec des droits stricts.** Par défaut, VirtualBox monte un dossier partagé en `777`. Or Ansible ignore volontairement un `ansible.cfg` placé dans un dossier modifiable par tous, pour éviter qu'un autre utilisateur y glisse une configuration malveillante. J'ai donc imposé `755` pour les dossiers et `644` pour les fichiers. Les serveurs gérés, eux, n'ont aucune raison de voir le code source.

**Ansible dans un environnement virtuel Python.** Ubuntu 24.04 interdit `pip install` dans le Python du système, pour ne pas casser les outils qui en dépendent. L'environnement virtuel isole les versions du projet ; il s'active automatiquement à chaque `vagrant ssh ctrl`.

**`StrictHostKeyChecking=accept-new`.** Ansible accepte automatiquement un serveur qu'il n'a jamais vu, mais refuse toujours de se connecter si l'empreinte d'un serveur connu change. C'est un bon compromis pour un labo où les VM sont souvent recréées.

**Le fuseau horaire `Africa/Lagos`.** C'est le nom officiel du fuseau UTC+1 du Bénin. L'alias `Africa/Porto-Novo` existe, mais il fait partie des anciens noms qu'Ubuntu 24.04 a déplacés dans un paquet séparé, et il risquait de faire échouer le provisionnement.

## Les problèmes rencontrés et comment je les ai résolus

### Des VM extrêmement lentes

**Symptôme.** Les VM dépassaient le délai de démarrage de Vagrant, même seules et avec un délai porté à dix minutes. Sur l'écran d'une VM, le noyau Linux affichait des traces d'appels signalant des tâches bloquées.

**Diagnostic.** J'ai d'abord soupçonné le manque de mémoire, puis Hyper-V. Mais `bcdedit` indiquait que l'hyperviseur de Windows ne devait pas démarrer, et aucune fonction Hyper-V, WSL ou plateforme de VM n'était activée. La preuve est venue du journal de la VM :

```text
HM: HMR3Init: Attempting fall back to NEM: VT-x is not available
```

VirtualBox n'avait pas accès directement à la virtualisation matérielle du processeur et tournait au-dessus de l'hyperviseur de Windows, dans un mode très lent. `systeminfo` confirmait qu'un hyperviseur était bien actif. En interrogeant Device Guard, j'ai trouvé la cause : la sécurité basée sur la virtualisation (VBS) était en cours d'exécution, alors qu'aucun service de sécurité ne l'utilisait.

**Correction.** Désactivation de VBS dans le registre (`EnableVirtualizationBasedSecurity` à 0), puis un vrai redémarrage. Un simple arrêt ne suffit pas quand le démarrage rapide de Windows est activé.

**Résultat.** Plus aucune trace de NEM dans le journal, et `ctrl` démarre en une minute environ.

### La clé privée refusée par SSH sous Windows

**Symptôme.** Au premier test de connexion depuis Windows, SSH a refusé la clé du labo (« UNPROTECTED PRIVATE KEY FILE ») et s'est rabattu sur une demande de mot de passe. Comme le compte `ansible` n'en a pas, la connexion était impossible, ce qui est d'ailleurs rassurant du point de vue de la sécurité.

**Cause.** La clé avait hérité des droits du dossier `C:\labs`, lisible par tous les comptes du PC. SSH refuse une clé privée que d'autres pourraient copier.

**Correction.** Suppression des droits hérités avec `icacls`, pour ne laisser que la lecture à mon compte. J'ai ajouté cette commande dans le Vagrantfile, juste après la génération de la clé : une clé recréée aura directement les bons droits.

### Les fins de ligne Windows

Git m'a averti que le Vagrantfile contenait des fins de ligne Windows (CRLF). Sans conséquence pour un fichier Ruby, mais un script shell exécuté dans une VM échouerait avec des erreurs comme `$'\r': command not found`. Le `.gitattributes` force les fins de ligne Unix (LF) dans le dépôt, et j'ai réglé VS Code pour écrire en LF et terminer chaque fichier par un saut de ligne. `git ls-files --eol` permet de vérifier.

### Quatre VM à la fois, c'est la limite du poste

En démarrant les quatre VM d'un coup, `mon01`, la plus lourde, a dépassé le délai : les trois autres occupaient déjà la mémoire. J'ai arrêté `db01`, terminé la configuration de `mon01` avec `vagrant reload mon01 --provision`, puis relancé `db01`. Le démarrage en deux temps est maintenant indiqué dans le README.

### Une leçon retenue pendant les exercices

En renommant une machine dans un Vagrantfile, j'ai créé une VM orpheline : Vagrant suit ses VM par leur nom de définition, donc l'ancienne VM existait toujours dans VirtualBox mais n'était plus gérée. Depuis, je fais toujours un `vagrant destroy` avant de renommer une machine.

## Vérifications

| Test | Résultat attendu | Obtenu |
| --- | --- | --- |
| Nom et réseau de `ctrl` | `hostname` = `ctrl`, `eth1` en 192.168.56.20 | ✅ |
| Partage par défaut désactivé | `/vagrant` absent | ✅ |
| Résolution des noms | `ping web01` depuis `ctrl` | ✅ |
| Idempotence de `/etc/hosts` | 4 lignes `lab.test` après deux provisionnements | ✅ |
| Compte `ansible` | Connexion par clé, `sudo -n true` réussi | ✅ |
| Droits du dossier partagé | `755` pour les dossiers, `644` pour les fichiers | ✅ |
| Idempotence de `ctrl` | Ligne d'activation présente une seule fois dans `.bashrc` | ✅ |
| Configuration lue par Ansible | `config file = /home/vagrant/projet/ansible.cfg` | ✅ |
| Connexion aux serveurs | `ansible all -m ping` : trois `SUCCESS` | ✅ |

## Limites assumées

- La clé du labo n'a pas de phrase de passe, pour qu'Ansible tourne sans intervention.
- Le compte `ansible` peut tout faire avec `sudo` sans mot de passe. En production, je restreindrais les droits ou j'exigerais un mot de passe.
- Désactiver VBS réduit la protection de Windows. C'est un compromis courant pour utiliser VirtualBox à pleine vitesse sur un poste de labo.

## Et ensuite

La phase 1 pose la structure professionnelle du projet : inventaires par environnement, variables, secrets chiffrés avec ansible-vault et un premier rôle, `common`.
