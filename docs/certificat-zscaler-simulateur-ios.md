---
layout: default
title: Certificat Zscaler dans un simulateur iOS
---

# Installer un certificat Zscaler dans un simulateur iOS

## 🎯 Objectif
Installer le certificat `zscaler.pem` dans un simulateur iOS lorsque :
- AirDrop est desactive par un profil MDM
- Le drag & drop vers le simulateur est bloque
- *Device Directory* n'existe plus (Xcode 15+)

Methode 100% compatible Zscaler et environnements verrouilles :
➡️ Telecharger le certificat via Safari du simulateur.

---

## 📁 1. Placer le certificat dans un dossier simple

Exemple :
```bash
$HOME/certs/zscaler.pem
```

---

## 🌐 2. Lancer un serveur HTTP local

Se rendre dans le dossier contenant le certificat et demarrer un serveur :
```bash
cd $HOME/certs
python3 -m http.server 8000
```

Le serveur expose maintenant le fichier sur :
```
http://<ip-du-mac>:8000/
```

---

## 🔍 3. Recuperer l'adresse IP locale du Mac

```bash
ipconfig getifaddr en0
```

Exemple :
```
192.168.0.25
```

---

## 📱 4. Telecharger le certificat via Safari du simulateur

Dans Safari du simulateur, entrer :
```
http://192.168.0.25:8000/zscaler.pem
```

Safari proposera :
- **Autoriser**
- **Installer** le profil

---

## ⚙️ 5. Finaliser l'installation du certificat

Dans le simulateur :

1. Ouvrir **Reglages → General → VPN et gestion de l'appareil**
2. Selectionner le profil Zscaler → **Installer**

Puis activer la confiance totale :

1. Ouvrir **Reglages → Informations → Reglages des certificats**
2. Activer **Zscaler Root CA**

---

## 🟢 6. Verification

Dans Safari du simulateur, visiter :
```
https://votre-api.tld
```

Si aucune alerte SSL n'apparait → installation reussie.

---

## 💡 Notes
- Methode compatible avec tous les simulateurs iOS (Xcode 15+).
- Le certificat peut devoir etre reinstalle apres un *Reset Content & Settings*.
- Fonctionne meme avec les restrictions MDM les plus strictes.
