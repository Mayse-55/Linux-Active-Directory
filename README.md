Voici une version légèrement améliorée :

---

# Joindre une VM Debian 13 à Active Directory

## 📦 Prérequis

### **Windows Server 2025 (AD)**
* Domaine opérationnel (ex. `bpx.local`) et contrôleur de domaine accessible
* Serveur DNS AD disponible pour la VM Debian
* Compte avec droits pour joindre des machines au domaine (ex. `Administrateur`)
* Heure synchronisée (NTP) entre Debian et AD — Kerberos est sensible à l'heure

### **Debian 13**
* Système à jour (`apt update && apt upgrade`)
* Accès root / sudo
* Résolution DNS pointant vers le(s) contrôleur(s) de domaine
* Ports requis ouverts (Kerberos, LDAP/LDAPS, RPC si nécessaire)

---

## 🚀 Instructions

> [!CAUTION]
> Assurez-vous que le **serveur Debian** et le **contrôleur AD** partagent la **même heure système**. Une différence peut provoquer **des échecs d'authentification Kerberos**.

### **1. Installer les paquets nécessaires**
```bash
sudo apt update
sudo apt install -y realmd sssd sssd-tools libnss-sss libpam-sss adcli samba-common-bin oddjob oddjob-mkhomedir packagekit krb5-user
```

### **2. Configuration DNS**

**Modifier le fichier de résolution DNS :**
```bash
nano /etc/resolv.conf
```

Le serveur DNS **doit pointer vers votre contrôleur Active Directory** pour la résolution des noms de domaine.

```bash
nameserver 30.31.3.182
# Remplacez par l'adresse IP de votre contrôleur AD
```

**Vérifier la connectivité :**
```bash
ping 30.31.3.182
# Remplacez par l'adresse IP de votre serveur AD

# OU

ping bpx.local
# Remplacez par le domaine de votre serveur AD
```

### **3. Découvrir le domaine**
```bash
realm discover bpx.local
```

### **4. Joindre le domaine**
```bash
sudo realm join --user=Administrateur bpx.local
```

* Entrez le mot de passe AD lorsque demandé
* En cas de succès : la machine sera créée dans l'OU `Computers` par défaut
