# Rapport de Lab — Bypass de Root Detection avec Medusa

**Auteur :** Maryam IKHERAZEN 
**Environnement :** Windows 11 — Genymotion Android 9.0 (Pie) x86 — Frida 17.10.1 — Medusa (dev)  
**App cible :** RootBeer Sample — `com.scottyab.rootbeer.sample`

> ⚠️ **Avertissement éthique :** Ces techniques sont utilisées uniquement dans un cadre pédagogique sur un appareil et une application pour lesquels une autorisation explicite a été obtenue.

---

## Sommaire

1. [Environnement de travail](#1-environnement-de-travail)
2. [Installation et preuve](#2-installation-et-preuve)
3. [Déploiement et visibilité](#3-déploiement-et-visibilité)
4. [Bypass avec Medusa](#4-bypass-avec-medusa)
5. [Plan B Frida pur](#5-plan-b-frida-pur)
6. [Comparaison Medusa vs Frida pur](#6-comparaison-medusa-vs-frida-pur)
7. [Conclusion](#7-conclusion)

---

## 1. Environnement de travail

| Composant | Détail |
|---|---|
| OS hôte | Windows 11 |
| Émulateur | Genymotion — Android 9.0 (Pie) API 28 x86 |
| Adresse ADB | 192.168.206.103:5555 |
| Python | 3.11 / 3.14 |
| Frida client | 17.10.1 |
| frida-server | frida-server-17.10.1-android-x86 |
| Medusa | Version dev — Ch0pin/medusa |
| App cible | RootBeer Sample |
| Package | com.scottyab.rootbeer.sample |

---

## 2. Installation et preuve

> 📌 **Note :** L'installation complète de Frida (client, frida-server, configuration ADB) a été réalisée et documentée dans le **Lab 10 — Installation et prise en main de Frida**. Les preuves ci-dessous confirment l'état opérationnel de l'environnement pour ce lab.

### 2.1 Versions Frida

```
frida --version
17.10.1

frida-ps --version
17.10.1

C:\Users\mayam\AppData\Local\Programs\Python\Python311\python.exe -c "import frida; print(frida.__version__)"
17.10.1
```

<!-- SCREEN : version.png — sortie frida --version et frida-ps --version affichant 17.10.1 -->

### 2.2 ADB devices

```
adb devices
List of devices attached
192.168.206.103:5555    device
```

<!-- SCREEN : adbdevices.png — sortie adb devices montrant le device Genymotion connecté -->

### 2.3 Installation de Medusa

```powershell
# Clonage du dépôt
git clone https://github.com/Ch0pin/medusa.git
cd medusa

# Installation des dépendances
pip install -r requirements.txt
pip install packaging

# Vérification
python medusa.py --help
```

**Sortie :**
<img width="946" height="390" alt="image" src="https://github.com/user-attachments/assets/72b899a5-ebfc-4fa6-9054-dbd686a58196" />

---

## 3. Déploiement et visibilité

### 3.1 Lancement de frida-server

```bash
adb shell "/data/local/tmp/frida-server-17.10.1-android-x86 -l 0.0.0.0 &"

# Vérification
adb shell ps | findstr frida
root    2650    1    937644    102864    poll_schedule_timeout    ea5e6bb9    S    frida-server-17.10.1-android-x86

````
### 3.2 Lancement de Medusa et connexion au device

```powershell
# Ajout de ADB au PATH
$env:PATH += ";C:\Users\mayam\AppData\Local\Android\Sdk\platform-tools"

# Lancement de Medusa
python medusa.py -p com.scottyab.rootbeer.sample
```

**Sélection du device :**

<img width="828" height="457" alt="image" src="https://github.com/user-attachments/assets/b2be2ebc-60d9-4e4f-b8f1-ab35b4c7974d" />

**Informations du device détectées par Medusa :**
<img width="503" height="420" alt="image" src="https://github.com/user-attachments/assets/674d9f19-c204-45a8-948c-fe2cb347dd7e" />

---

## 4. Bypass avec Medusa

### 4.1 État AVANT bypass

Lancement de RootBeer Sample sans instrumentation :
<img width="452" height="962" alt="rooted" src="https://github.com/user-attachments/assets/364186b3-9006-4a0a-bf85-8d62ad2c0c82" />

| **Verdict** | 🔴 **ROOTED*** |

---

### 4.2 Recherche du module root bypass dans Medusa

<img width="551" height="106" alt="image" src="https://github.com/user-attachments/assets/8259e4c5-bdbb-4806-994f-1ba2cd06f2b8" />

### 4.3 Module utilisé — `rootbeer_detection_bypass_no_obfuscation`

Contenu du module `anti_root_beer_no_obfuscation.med` :

```json
{
    "Name": "root_detection/rootbeer_detection_bypass_no_obfuscation",
    "Description": "Bypass rootbeer checks",
    "Code": "
    console.log('\\n---------LOADING ANTI ROOT DETECTION SCRIPT-------------------');
    try {
        var targetClass12 = Java.use('com.scottyab.rootbeer.RootBeer');
        var methods = targetClass12.class.getDeclaredMethods();
        methods.forEach(function (method) {
            var methodName = method.getName();
            if (method.getReturnType().getName() === 'boolean') {
                console.log('Hooking method: ' + methodName);
                var overloads = targetClass12[methodName].overloads;
                overloads.forEach(function (overload) {
                    overload.implementation = function () {
                        console.log('Hooked method: ' + methodName);
                        return false;
                    };
                });
            }
        });
    } catch (error) {
        console.error('Error: ' + error);
    }"
}
```

**Stratégie :** Le module énumère dynamiquement toutes les méthodes de `com.scottyab.rootbeer.RootBeer` qui retournent un `boolean` et force leur retour à `false` — sans avoir besoin de connaître les noms exacts des méthodes.

### 4.4 Chargement et exécution
<img width="672" height="145" alt="image" src="https://github.com/user-attachments/assets/e75ca064-d474-4588-a603-0efbf24c43f6" />


**Logs Medusa :**

<img width="566" height="473" alt="image" src="https://github.com/user-attachments/assets/d331dcd0-220d-4be1-a7dd-028a995f8186" />

### 4.5 État APRÈS bypass Medusa

| Check | Avant | Après Medusa |
|---|---|---|
| Root Management Apps | ❌ | ✅ |
| Potentially Dangerous Apps | ❌ | ✅ |
| Root Cloaking Apps | ❌ | ✅ |
| TestKeys | ❌ | ✅ |
| BusyBoxBinary | ❌ | ✅ |
| SU Binary | ❌ | ✅ |
| 2nd SU Binary check | ❌ | ✅ |
| For RW Paths | ❌ | ✅ |
| Dangerous Props | ❌ | ✅ |
| Root via native check | ❌ | ✅ |
| SE Linux Flag Is Enabled | ✅ | ✅ |
| Magisk specific checks | ✅ | ✅ |
| **Score** | 0/12 | **12/12** |
| **Verdict** | 🔴 ROOTED* | 🟢 **NOT ROOTED** |

<img width="455" height="931" alt="rooted3" src="https://github.com/user-attachments/assets/382a341d-dc03-4160-bc39-893d2e809a05" />

---

## 5. Plan B Frida pur

> 📌 **Note :** Le Plan B (Frida pur) a été intégralement réalisé dans le **Lab précédent — Bypass de Root Detection avec Frida**. Les résultats sont résumés ici pour comparaison.

### 5.1 Résultats Plan B

```bash
frida -U -f com.scottyab.rootbeer.sample -l bypass_root.js -l bypass_native.js
```

**Logs :**
```
[+] Hook Build.TAGS -> release-keys
[+] RootBeer hooks installés
[+] File.exists bypass for /system/bin/su
[+] Blocked open on /sbin/su
[+] Blocked open on /vendor/bin/su
[+] Blocked access on /data/local/magisk
[+] Blocked access on /system/bin/magisk
...
```

| Check | Frida pur |
|---|---|
| 10 checks Java/natifs | ✅ bypassés |
| Dangerous Props | ❌ résiste |
| **Score** | **9/12** |

<img width="457" height="931" alt="rooted2" src="https://github.com/user-attachments/assets/f132e204-25d4-42c8-abdb-84bd1c11a9d0" />

---

## 6. Comparaison Medusa vs Frida pur

| Critère | Frida pur | Medusa |
|---|---|---|
| Score bypass | 9/12 | **12/12** |
| Verdict final | 🟡 ROOTED* | 🟢 **NOT ROOTED** |
| Dangerous Props | ❌ | ✅ |
| Root via native check | ✅ | ✅ |
| Facilité d'utilisation | Scripts manuels | Module prêt à l'emploi |
| Temps de mise en place | ~30 min | ~5 min |
| Flexibilité | ✅ Haute | ✅ Haute |
| Modules disponibles | Scripts custom | 124 modules intégrés |

**Avantage Medusa :** Le module `rootbeer_detection_bypass_no_obfuscation` énumère dynamiquement **toutes** les méthodes boolean de `RootBeer` sans liste prédéfinie — il s'adapte automatiquement aux versions futures de RootBeer.

**Avantage Frida pur :** Plus de contrôle granulaire, pas de dépendance externe, utile pour des apps sans lib tierce connue.

---

## 7. Conclusion

### Récapitulatif des objectifs

| Objectif | Statut |
|---|---|
| Installer et configurer Medusa | ✅ |
| Connecter Medusa au device Genymotion | ✅ |
| Identifier le bon module root bypass | ✅ `root_detection/rootbeer_detection_bypass_no_obfuscation` |
| Bypass complet avec Medusa | ✅ 12/12 — NOT ROOTED |
| Plan B Frida pur documenté | ✅ 9/12 |
| Comparaison Medusa vs Frida | ✅ |

### Points clés

**Medusa** automatise l'instrumentation Android via des modules `.med` (JSON + code Frida). Le module `rootbeer_detection_bypass_no_obfuscation` utilise la réflexion Java pour hooker dynamiquement toutes les méthodes boolean de `RootBeer`, ce qui le rend robuste face aux changements de versions et à l'obfuscation partielle.

**Dépannage rencontré :**
- `adb` absent du PATH lors du lancement de Medusa → résolu via `$env:PATH`
- Module `load` incompatible (attend lib.so) → résolu en utilisant `use`
- `IndexError` sur modules avec JSON invalide → résolu via `reload` + `use` correct
- App non lancée au premier `run` → résolu via `adb shell am start` avant

---

*Lab Medusa Root Detection Bypass — Frida 17.10.1*
