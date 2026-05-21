# LAB 16 — Inspection HTTPS Android : Désactivation du SSL Pinning avec Objection + Proxy (Burp/mitmproxy)

> **Auteurs :** Hamdi · Maroua  
> **Cours :** Sécurité des applications mobiles  
> **Outils principaux :** Objection · Frida · Burp Suite / mitmproxy

---

##  Avertissement éthique

> N'appliquez ce lab que dans un cadre légal (appareils/apps vous appartenant ou audit autorisé).  
> Le contournement du SSL pinning sert à **analyser la sécurité réseau** d'une application,  
> **pas à compromettre des systèmes en production**.

---

##  Objectifs du lab

- Installer et utiliser **Objection** (wrapper Frida en ligne de commande)
- Préparer un appareil/émulateur Android et démarrer `frida-server`
- Configurer un proxy HTTPS (Burp Suite ou mitmproxy) et installer la CA
- Lancer l'app cible via Objection et désactiver le SSL pinning en une commande
- Valider l'interception du trafic HTTPS dans le proxy

---

##  Prérequis

- Python 3.x + `pip` installés sur le PC
- Android Debug Bridge (`adb`) installé et dans le PATH
- Android Emulator (AVD) ou device physique rooté avec image `userdebug`
- Burp Suite Community/Pro **ou** mitmproxy
- `frida-server` correspondant à l'architecture de l'appareil

---

### Étape 1 — Installer Objection et Frida côté PC

Vérifier les prérequis (Python, pip, adb) :

<img width="2167" height="725" alt="pic1" src="https://github.com/user-attachments/assets/91402b12-4f9b-4871-ae59-c2a70f2b03e2" />


<img width="1363" height="1154" alt="pic2" src="https://github.com/user-attachments/assets/42ece6f6-56c7-4579-aae0-956244782c2b" />


```bash
# Vérifier les versions
python --version      # Python 3.13.0
pip --version
adb version           # ADB 1.0.41
```

Installer Frida et Objection :

```bash
pip install frida-tools objection
```

---

### Étape 2 — Préparer l'appareil et démarrer frida-server

#### 2.1 — Lister les packages installés sur l'émulateur

```bash
adb shell pm list packages
```

<img width="1606" height="979" alt="pic5" src="https://github.com/user-attachments/assets/6a472582-2b44-43ef-a004-a28e7b35fc31" />


#### 2.2 — Pousser et démarrer frida-server

```bash
adb push frida-server /data/local/tmp/
adb shell chmod +x /data/local/tmp/frida-server
adb shell /data/local/tmp/frida-server &
```

#### 2.3 — Forwarder les ports Frida et vérifier les processus

```bash
adb forward tcp:27042 tcp:27042
adb forward tcp:27043 tcp:27043
frida-ps -Uai
```

<img width="375" height="732" alt="pic3" src="https://github.com/user-attachments/assets/18618a39-9c40-4be4-bcb2-a27bb2a3890d" />


Repérer le package cible dans la liste (ex. `owasp.mstg.uncrackable1`, `com.pwnse.firestorm`, etc.).

---

### Étape 3 — Configurer le proxy et installer la CA

#### 3.1 — Proxy Android (Wi-Fi manuel)

Sur l'émulateur, aller dans **Paramètres → Wi-Fi → AndroidWifi → Modifier → Proxy Manuel** :

| Champ | Valeur |
|---|---|
| Proxy hostname | `192.168.1.1` *(IP de la machine hôte)* |
| Proxy port | `8080` |

<img width="822" height="83" alt="pic4" src="https://github.com/user-attachments/assets/68e4cacb-6a6f-4f2e-8ac2-84d913a7dd8c" />

#### 3.2 — Installer le certificat CA Burp

```bash
# Exporter le certificat depuis Burp : Proxy > Options > Import/Export CA Certificate
# Pousser et installer sur l'émulateur
adb push burp_ca.der /data/local/tmp/
adb shell "openssl x509 -inform DER -in /data/local/tmp/burp_ca.der \
  -out /data/local/tmp/burp_ca.pem"
# Installer via Settings > Security > Install from storage
```

---

### Étape 4 — Lancer l'app avec Objection et désactiver le pinning

```bash
objection -g <package.name> explore
```

Une fois dans le REPL Objection :

```
android sslpinning disable


**Résultats attendus :**

```
(agent) Custom TrustManager ready, overriding SSLContext.init()
(agent) Found com.android.org.conscrypt.TrustManagerImpl, overriding TrustManagerImpl.verifyChain()
(agent) Found com.android.org.conscrypt.TrustManagerImpl, overriding TrustManagerImpl.checkTrustedRecursive()
(agent) Registering job XXXXX. Name: android-sslpinning-disable
```

#### Variante — tout en une ligne (démarrage automatique)

```bash
objection -g <package.name> explore --startup-command "android sslpinning disable"
```

---

### Étape 5 — Validation

Ouvrir Burp Suite et vérifier que les requêtes HTTPS de l'app apparaissent dans l'onglet **Proxy > HTTP history**.

- [ ] Requêtes HTTPS interceptées sans erreur de certificat
- [ ] Logs Objection confirmant `android-sslpinning-disable` actif
- [ ] Screenshots du trafic intercepté joints au rapport

---

### Étape 6 — Dépannage (FAQ)

| Problème | Cause probable | Solution |
|---|---|---|
| `frida-ps` ne répond pas | frida-server non démarré ou mauvaise version | Vérifier `frida --version` == version du serveur |
| `objection` crash au démarrage | Incompatibilité Frida/Objection | `pip install --upgrade frida-tools objection` |
| Trafic non intercepté dans Burp | Proxy mal configuré ou CA non installée | Vérifier IP hôte + installer CA en certificat système |
| App crash après bypass | Pinning natif (BoringSSL) non couvert par Objection | Combiner avec `sslpin_bypass_native.js` via Frida |

---

### Étape 7 — Bonnes pratiques

- Toujours utiliser une image `userdebug` — ne jamais tester sur un device personnel avec données réelles
- Documenter chaque bypass appliqué et les packages ciblés dans le rapport
- Supprimer `frida-server` de l'appareil après les tests
- Ne pas conserver de captures de trafic contenant des données sensibles

---

## 📁 Structure du dépôt

```
.
├── README_LAB16.md
└── images/
    └── lab16/
        ├── pic1.png   ← Vue d'ensemble du lab (plateforme de cours)
        ├── pic2.png   ← Vérification versions python/pip/adb
        ├── pic3.png   ← frida-ps -Uai (liste des processus)
        ├── pic4.png   ← Config proxy Android
        ├── pic5.png   ← adb shell pm list packages
        └── pic6.png   ← Objection android sslpinning disable
```

---

## 📚 Références

- [Objection GitHub](https://github.com/sensepost/objection)
- [Frida Documentation](https://frida.re/docs/home/)
- [OWASP MSTG — Network Communication](https://mas.owasp.org/MASTG/tests/android/MASVS-NETWORK/)
- [Burp Suite — Installing CA Certificate](https://portswigger.net/burp/documentation/desktop/mobile/config-android-device)
