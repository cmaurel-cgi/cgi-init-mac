---
layout: default
title: Zscaler, Node.js et Android Studio
---

# Zscaler, Node.js et Android Studio

Guide pour faire fonctionner un environnement front/mobile derriere Zscaler sur macOS.

## Sommaire
- [1. Preparer les certificats Zscaler](#1-preparer-les-certificats-zscaler)
- [2. Configurer npm et pnpm](#2-configurer-npm-et-pnpm)
- [3. Rendre Node visible pour Android Studio](#3-rendre-node-visible-pour-android-studio)
- [4. Fix SSL Gradle et Java (PKIX)](#4-fix-ssl-gradle-et-java-pkix)
- [5. Verification et nettoyage](#5-verification-et-nettoyage)
- [6. Depannage rapide](#6-depannage-rapide)
- [7. Expo Android : cert Zscaler dans l'emulateur](#7-expo-android--cert-zscaler-dans-lemulateur)

---

## 1. Preparer les certificats Zscaler

Creer un dossier local :

```bash
mkdir -p ~/certs
```

Y placer le fichier fourni par la DSI :

```
~/certs/zscaler.pem
```

Creer aussi une version `.crt` pour Java :

```bash
cp ~/certs/zscaler.pem ~/certs/zscaler.crt
```

---

## 2. Configurer npm et pnpm

Zscaler intercepte le HTTPS : Node doit explicitement faire confiance au certificat Zscaler.

### Ajouter dans `~/.zshrc`

```bash
export NODE_EXTRA_CA_CERTS="$HOME/certs/zscaler.pem"
```

Recharge la config :

```bash
source ~/.zshrc
```

### Configurer npm

```bash
npm config set cafile "$HOME/certs/zscaler.pem"
```

### Configurer pnpm

```bash
pnpm config set cafile "$HOME/certs/zscaler.pem"
```

Resultat :
- `npm install` OK
- `pnpm install` OK
- plus d'erreurs `SELF_SIGNED_CERT_IN_CHAIN`

---

## 3. Rendre Node visible pour Android Studio

Android Studio ne lit pas ton `.zshrc` : il ne voit pas le Node installe via Homebrew.

Node ARM est generalement ici :

```
/opt/homebrew/opt/node@24/bin/node
```

Fix universel : creer un symlink dans `/usr/local/bin` (visible par les apps GUI).

```bash
sudo mkdir -p /usr/local/bin
sudo ln -sf /opt/homebrew/opt/node@24/bin/node /usr/local/bin/node
```

Cela resout l'erreur :

```
Cannot run program "node": No such file or directory
```

---

## 4. Fix SSL Gradle et Java (PKIX)

Erreur typique avec Zscaler :

```
PKIX path building failed
unable to find valid certification path to requested target
```

Java (donc Gradle) n'accepte pas le certificat Zscaler : il faut l'ajouter au truststore du JDK utilise par Gradle.

### 4.1 Identifier le JDK utilise par Android Studio

Android Studio :
`Settings > Build, Execution, Deployment > Build Tools > Gradle > Gradle JDK`

Deux cas :

#### A) Embedded JDK (JBR)

Chemin :

```
/Applications/Android Studio.app/Contents/jbr/Contents/Home
```

#### B) Zulu 17 (si configure)

Chemin :

```
/Library/Java/JavaVirtualMachines/zulu-17.jdk/Contents/Home
```

### 4.2 Importer le certificat pour Java et Gradle

#### A) Si Gradle utilise l'Embedded JDK (JBR)

```bash
sudo keytool -importcert \
  -alias zscaler \
  -file "$HOME/certs/zscaler.crt" \
  -keystore "/Applications/Android Studio.app/Contents/jbr/Contents/Home/lib/security/cacerts" \
  -storepass changeit \
  -noprompt
```

#### B) Si Gradle utilise Zulu 17

```bash
sudo keytool -importcert \
  -alias zscaler \
  -file "$HOME/certs/zscaler.crt" \
  -keystore "/Library/Java/JavaVirtualMachines/zulu-17.jdk/Contents/Home/lib/security/cacerts" \
  -storepass changeit \
  -noprompt
```

Mot de passe du truststore Java : `changeit`.

Cela resout les erreurs `PKIX`, `SSLHandshakeException`, `Could not GET https://plugins.gradle.org`.

---

## 5. Verification et nettoyage

### Verifier que le cert est installe

Pour l'Embedded JDK :

```bash
keytool -list \
  -keystore "/Applications/Android Studio.app/Contents/jbr/Contents/Home/lib/security/cacerts" \
  -storepass changeit | grep -i zscaler
```

Ou Zulu :

```bash
keytool -list \
  -keystore "$JAVA_HOME/lib/security/cacerts" \
  -storepass changeit | grep -i zscaler
```

### Nettoyage Gradle

```bash
cd android
./gradlew --stop
./gradlew clean
```

Puis relancer Android Studio.

---

## 6. Depannage rapide

### npm/pnpm : SELF_SIGNED_CERT

Verifier :

```bash
npm config get cafile
pnpm config get cafile
echo $NODE_EXTRA_CA_CERTS
```

### Android Studio : Cannot run node

Recreer le lien :

```bash
sudo ln -sf /opt/homebrew/opt/node@24/bin/node /usr/local/bin/node
```

### Gradle : PKIX / SSLHandshakeException

- Mauvais JDK : re-verifie le `Gradle JDK`.
- Importe le cert dans ce JDK.

---

## Resultat final

Avec ce guide, tout fonctionne derriere Zscaler :
- npm et pnpm
- Node accessible pour Android Studio
- Gradle telecharge les dependances
- plus d'erreurs PKIX
- environnement mobile operationnel

---

## Bonus : truststore dedie

Si tu veux un truststore dedie plutot que de modifier `cacerts`, cree `~/certs/zscaler.jks` et ajoute ceci dans `~/.gradle/gradle.properties` :

```properties
org.gradle.jvmargs=-Djavax.net.ssl.trustStore=$HOME/certs/zscaler.jks -Djavax.net.ssl.trustStorePassword=zscalerpass
```

---

## 7. Expo Android : cert Zscaler dans l'emulateur

Cette section couvre le cas React Native/Expo pour :
- autoriser le cleartext vers des IP de dev locales
- faire confiance aux certificats utilisateur sur Android
- importer le certificat Zscaler dans l'emulateur, comme sur iOS

### 7.1 Ajouter le plugin dans `app.config.json`

Dans la section `expo.plugins`, ajouter :

```json
[
  "./plugins/withNetworkSecurityConfig.js",
  {
    "cleartextIps": ["127.0.0.1", "192.168.1.112", "localhost"]
  }
]
```

Exemple d'integration :

```json
{
  "expo": {
    "plugins": [
      [
        "./plugins/withNetworkSecurityConfig.js",
        {
          "cleartextIps": ["127.0.0.1", "192.168.1.112", "localhost"]
        }
      ]
    ]
  }
}
```

### 7.2 Ajouter le dossier et le fichier plugin

Creer ce dossier dans le projet mobile :

```
plugins/
  withNetworkSecurityConfig.js
```

Contenu de `plugins/withNetworkSecurityConfig.js` :

```js
const { withAndroidManifest } = require('@expo/config-plugins');
const fs = require('fs');
const path = require('path');

/**
 * Expo plugin to add network security config for:
 * 1. Trusting user certificates (like Zscaler) in debug builds
 * 2. Allowing cleartext traffic to specific IPs (for local development)
 *
 * @param {object} config - Expo config
 * @param {object} props - Plugin configuration
 * @param {string[]} props.cleartextIps - Array of IPs to allow cleartext traffic
 */
module.exports = function withNetworkSecurityConfig(config, props = {}) {
  return withAndroidManifest(config, async (configWithManifest) => {
    const androidManifest = configWithManifest.modResults;

    const mainApplication = androidManifest.manifest.application[0];
    mainApplication.$['android:networkSecurityConfig'] = '@xml/network_security_config';

    const projectRoot = configWithManifest.modRequest.projectRoot;
    const resXmlDir = path.join(
      projectRoot,
      'android',
      'app',
      'src',
      'main',
      'res',
      'xml'
    );

    if (!fs.existsSync(resXmlDir)) {
      fs.mkdirSync(resXmlDir, { recursive: true });
    }

    const networkSecurityConfigPath = path.join(resXmlDir, 'network_security_config.xml');

    const cleartextIps = props.cleartextIps || ['127.0.0.1', 'localhost'];
    const ipDomains = cleartextIps
      .filter((ip) => ip && ip.length > 0)
      .map((ip) => `        <domain includeSubdomains="true">${ip}</domain>`)
      .join('\n');

    const networkSecurityConfig = `<?xml version="1.0" encoding="utf-8"?>
<network-security-config>

    <domain-config cleartextTrafficPermitted="true">
${ipDomains}
    </domain-config>

    <base-config>
        <trust-anchors>
            <certificates src="system" />
            <certificates src="user" />
        </trust-anchors>
    </base-config>
</network-security-config>
`;

    fs.writeFileSync(networkSecurityConfigPath, networkSecurityConfig);

    return configWithManifest;
  });
};
```

### 7.3 Regenerer le projet Android

Apres ajout/modification du plugin :

```bash
npx expo prebuild -p android --clean
```

Puis lancer l'app :

```bash
npx expo run:android
```

### 7.4 Importer le certificat dans l'emulateur Android (meme logique que iOS)

1. Placer le certificat dans un dossier simple :

```bash
$HOME/certs/zscaler.pem
```

2. Lancer un serveur HTTP local :

```bash
cd $HOME/certs
python3 -m http.server 8000
```

3. Recuperer l'IP locale du Mac :

```bash
ipconfig getifaddr en0
```

4. Dans le navigateur de l'emulateur Android, ouvrir :

```
http://<ip-du-mac>:8000/zscaler.pem
```

5. Installer le certificat dans Android :
- Ouvrir `Settings > Security & privacy > Encryption & credentials > Install a certificate`
- Choisir `CA certificate`
- Selectionner le fichier telecharge

### 7.5 Verification

- Relancer l'application dans l'emulateur.
- Verifier qu'un appel HTTPS vers votre API ne remonte plus d'erreur SSL.
- En debug, les IP declarees dans `cleartextIps` sont autorisees pour le cleartext.
