# Joindre une VM Debian 13 à Active Directory (Windows Server 2025)

## Résumé

Ce dépôt contient une documentation et un script pour joindre une machine **Debian 13** à un domaine **Active Directory** (ex. `bpx.fr`) géré par **Windows Server 2025**. La méthode utilise `realmd` + `sssd` pour une intégration propre et sécurisée.

---

## Structure du dépôt

```
README.md                # Cette documentation
join-domain.sh           # Script d'automatisation (bash)
LICENSE                  # MIT (optionnel)
```

---

## Prérequis

### Côté Windows Server (AD)

* Domaine opérationnel (ex. `bpx.fr`) et contrôleur de domaine accessible.
* Serveur DNS AD disponible pour la VM Debian.
* Compte avec droits pour joindre des machines au domaine (ex. `Administrateur`).
* Heure synchronisée (NTP) entre Debian et AD (Kerberos est sensible à l'heure).

### Côté Debian 13

* Système à jour (`apt update && apt upgrade`).
* Accès root / sudo.
* Résolution DNS pointant vers le(s) contrôleur(s) de domaine.
* Ports requis ouverts (Kerberos, LDAP/LDAPS, RPC si nécessaire).

---

## Variables importantes

* `DOMAIN` : nom du domaine AD (ex. `bpx.fr`)
* `AD_ADMIN` : compte utilisé pour joindre le domaine (ex. `Administrateur`)
* `AD_DNS` : adresse IP du serveur DNS AD (optionnel si déjà configuré)
* `HOSTNAME` : (optionnel) nom d'hôte de la VM côté Debian

---

## Instructions manuelles (résumé)

1. Vérifier DNS

```bash
cat /etc/resolv.conf
# doit pointer vers le DNS AD ou contenir search bpx.fr
```

2. Installer paquets

```bash
sudo apt update
sudo apt install -y realmd sssd sssd-tools libnss-sss libpam-sss adcli samba-common-bin oddjob oddjob-mkhomedir packagekit krb5-user
```

> Lors de l'installation de `krb5-user`, renseigne le realm si demandé (ex. `BPX.FR`) ou laisse vide et configure plus tard.

3. Découvrir le domaine

```bash
realm discover bpx.fr
```

4. Joindre le domaine

```bash
sudo realm join --user=Administrateur bpx.fr
```

* Entrer le mot de passe AD quand demandé.
* Si succès : la machine sera créée dans l'OU `Computers` par défaut.

5. Vérifier la jointure

```bash
realm list
id utilisateur@bpx.fr
getent passwd 'utilisateur@bpx.fr'
```

6. Activer création automatique de home dirs

```bash
sudo pam-auth-update --enable mkhomedir
# ou via sssd.conf:
# use_fully_qualified_names = False
# fallback_homedir = /home/%u
sudo systemctl restart sssd
```

---

## Script d'automatisation (`join-domain.sh`)

Un script prêt à l'emploi est fourni dans ce dépôt. Il :

* installe les paquets
* découvre le domaine
* joint la machine
* configure `sssd` pour ne pas exiger les noms qualifiés (`use_fully_qualified_names = False`)
* active la création automatique des homedirs

**Important :** lire et adapter les variables en haut du script (DOMAIN, ADMIN_USER, DNS_SERVER, etc.) avant exécution.

---

## Sécurité & bonnes pratiques

* Utiliser une connexion sécurisée pour transmettre le mot de passe (exécution locale ou via un mécanisme de vault).
* Révoquer les droits du compte utilisé pour la jointure si nécessaire, ou utiliser un compte dédié pour les opérations d'auto-join.
* Vérifier les ACLs et l'OU cible dans Active Directory si tu veux contrôler l'emplacement des objets ordinateurs.
* Mettre en place la journalisation (rsyslog/journald) et vérifier `/var/log/sssd/` en cas de problème.

---

## Dépannage rapide

* Erreur Kerberos (clock skew) : vérifier l'heure (`timedatectl`) et NTP.
* `realm discover` échoue : tester `nslookup _ldap._tcp.bpx.fr` ou `dig` sur les SRV records.
* Utilisateurs AD non résolus : vérifier `/etc/sssd/sssd.conf`, redémarrer `sssd` et vérifier `/var/log/sssd/`.
* SSH : configurer `PermitRootLogin` et `PasswordAuthentication` selon la politique ; tester en se connectant avec un utilisateur AD (`ssh 'user@bpx.fr'@host` ou après `use_fully_qualified_names = False` : `ssh user@host`).

---

## Exemple de `sssd.conf` minimal (généré par realmd mais utile pour debug)

```ini
[sssd]
services = nss, pam
config_file_version = 2
domains = bpx.fr

[domain/bpx.fr]
ad_domain = bpx.fr
krb5_realm = BPX.FR
realmd_tags = manages-system joined-with-samba
cache_credentials = True
id_provider = ad
ldap_id_mapping = True
use_fully_qualified_names = False
fallback_homedir = /home/%u
default_shell = /bin/bash
```

---

## Utilisation GitHub / CI

* Si tu veux automatiser la jointure via CI (p. ex. création d'images cloud-init), préfère utiliser des mécanismes sécurisés (secrets vault) et éviter de stocker des mots de passe en clair dans le repo.
* Exemple : exécuter `join-domain.sh` dans un job GitHub Actions **sur une VM provisionnée** (attention aux permissions réseau / DNS).

---

## Licence

Propose d'utiliser MIT. Ajouter `LICENSE` si souhaité.

---

## Contribuer / Contact

Pour modifications, tests supplémentaires (ex. OU de destination, LDAPS, restrictions d'accès), ouvre une issue ou crée une pull request.

---

Bonne mise en place — si tu veux, je peux :

* t'ajouter un `cloud-init` pour images cloud (Debian 13),
* fournir un workflow GitHub Actions d'exemple pour tester le script,
* générer le fichier `join-domain.sh` prêt à l'emploi.
