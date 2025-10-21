# Joindre une VM Debian 13 à Active Directory

## 📦 Prérequis

**Windows Server 2025 (AD)**
* Domaine opérationnel (ex. `bpx.local`) et contrôleur de domaine accessible.
* Serveur DNS AD disponible pour la VM Debian.
* Compte avec droits pour joindre des machines au domaine (ex. `Administrateur`).
* Heure synchronisée (NTP) entre Debian et AD (Kerberos est sensible à l'heure).

**Debian 13**
* Système à jour (`apt update && apt upgrade`).
* Accès root / sudo.
* Résolution DNS pointant vers le(s) contrôleur(s) de domaine.
* Ports requis ouverts (Kerberos, LDAP/LDAPS, RPC si nécessaire).

---

## 📋 Variables importantes

* `DOMAIN` : nom du domaine AD (ex. `bpx.local`)
* `AD_ADMIN` : compte utilisé pour joindre le domaine (ex. `Administrateur`)
* `AD_DNS` : adresse IP du serveur DNS AD (optionnel si déjà configuré)
* `HOSTNAME` : (optionnel) nom d'hôte de la VM côté Debian

---

## 🚀 Instructions

> [!CAUTION]
> **🕐 Attention : Synchronisation Horaire**
> 
> Assurez-vous que le **serveur Debian** et le **client** partagent la **même heure système**. Une différence de temps peut entraîner des **échecs d'authentification** Kerberos.

### 1. Vérifier DNS

```bash
cat /etc/resolv.conf
# doit pointer vers le DNS AD
```

### 2. Installer les paquets

```bash
sudo apt update
sudo apt install -y realmd sssd sssd-tools libnss-sss libpam-sss adcli samba-common-bin oddjob oddjob-mkhomedir packagekit krb5-user
```

### 3. Découvrir le domaine

```bash
realm discover bpx.local
```

### 4. Joindre le domaine

```bash
sudo realm join --user=Administrateur bpx.local
```

* Entrer le mot de passe AD quand demandé.
* Si succès : la machine sera créée dans l'OU `Computers` par défaut.
