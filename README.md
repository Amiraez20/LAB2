# Sécurité Android — Privilèges Élevés, Intégrité Système et Traçabilité

> **Cadre légal et éthique :** Ce travail pratique s'inscrit strictement dans un contexte pédagogique encadré. Toutes les manipulations s'effectuent sur un appareil virtuel dédié ou un appareil de laboratoire explicitement prévu à cet effet. Utiliser ces techniques sur un appareil personnel ou sur une application tierce sans autorisation constitue une infraction. L'objectif est uniquement d'observer et de comprendre les mécanismes de sécurité Android — jamais de les exploiter à des fins malveillantes.

---

## Avant de commencer — Mise en contexte (≈ 2 min)

### Qu'est-ce que le "rooting" ?

Obtenir les droits root sur Android revient à accéder au niveau d'administration le plus élevé du système d'exploitation, à l'image du compte administrateur sur Windows ou du `sudo` sur Linux. Sur un système Android standard, cet accès est délibérément verrouillé pour protéger le système et les données utilisateur.

> **Analogie :** Le root, c'est posséder le passe-partout du gardien d'immeuble. Pratique pour inspecter chaque recoin, mais dangereux s'il est mal utilisé ou perdu.

### Règles du périmètre

| Règle | Justification |
|---|---|
| Appareil virtuel ou matériel de labo uniquement | Éviter tout risque sur un appareil personnel |
| Données fictives uniquement | Aucune information réelle ne doit être exposée |
| Rien ne sort du périmètre de test | Limiter les effets de bord sur des systèmes externes |

### Vérification préalable de l'outillage

```bash
adb devices
```

Cette commande doit afficher votre émulateur avec le statut `device`. Si la sortie est vide, vérifier que l'émulateur est bien démarré et que les outils Android SDK sont correctement installés et accessibles dans votre `PATH`.

> **Pour les débutants :** ADB (*Android Debug Bridge*) est le pont de communication entre votre poste de travail et l'appareil Android (réel ou virtuel). Pensez-y comme à un câble de gestion réseau reliant un administrateur à son équipement.

---

## Phase 1 — Préparation de l'environnement

### Étape 1 — Démarrer un AVD propre

1. Ouvrir Android Studio → *Device Manager* → démarrer un AVD existant ou en créer un nouveau (API 29 minimum recommandé)
2. S'assurer que l'écran d'accueil Android s'affiche sans compte personnel configuré
3. Confirmer la détection par ADB :

```bash
adb devices
```

> **Bonne pratique :** Toujours partir d'un AVD vierge, sans résidus de sessions précédentes. Réutiliser un AVD "contaminé" par des tests antérieurs biaise les résultats, exactement comme réutiliser un tube à essai non stérilisé en chimie.

> **Pourquoi API 29+ ?** Les versions récentes d'Android intègrent des mécanismes de sécurité plus complets (scoped storage, partitioned permissions, etc.), ce qui permet d'observer les protections modernes en fonctionnement.

**Erreur fréquente :** Réutiliser un AVD ayant déjà servi pour d'autres tests sans effectuer de remise à zéro préalable.

---

### Étape 2 — Installer l'application de test

```bash
adb install app-evaluation.apk
```

> Alternativement, utiliser le bouton *Run* dans Android Studio pour les débutants. La ligne de commande offre davantage de contrôle et prépare à l'automatisation des tests.

Vérifier que l'application s'ouvre correctement et noter sa version dans le journal de session. La version est une information critique : les comportements de sécurité varient d'une release à l'autre.

---

### Étape 3 — Définir trois scénarios fonctionnels reproductibles

Choisir trois parcours utilisateur représentatifs de l'application. Exemples :

1. Afficher l'écran principal après lancement
2. Effectuer une recherche avec un terme prédéfini
3. Consulter le détail d'un élément spécifique

> **Principe :** Ces scénarios servent de protocole expérimental de référence. Ils doivent être suffisamment précis pour être reproduits à l'identique par n'importe quel membre de l'équipe, sans ambiguïté.

Documenter pour chaque scénario : l'action déclenchée, les données saisies (exactes), le résultat attendu.

---

## Phase 2 — Obtention des privilèges élevés

### Étape 4 — Élévation de privilèges sur l'AVD

**Pourquoi cette étape ?** Disposer des droits root sur un environnement isolé et jetable permet d'observer l'impact de cette élévation sur les mécanismes de protection Android et d'inspecter des zones du système normalement inaccessibles.

> **Concept clé :** Un système rooté donne accès à l'intégralité des partitions et processus. C'est comme disposer des clés de toutes les zones d'un bâtiment, y compris les locaux techniques réservés.

**Prérequis :** AVD démarré et détecté par `adb devices`.

```bash
# Lancer l'émulateur avec système inscriptible
emulator -avd NOM_DE_LAVD -writable-system

# Activer le serveur ADB avec droits élevés
adb root

# Remonter la partition système en lecture/écriture
adb remount
```

**Explication technique :**
- `adb root` redémarre le démon ADB avec les privilèges administrateur
- `adb remount` remonte `/system` en mode lecture/écriture, ce qui est normalement bloqué par des mécanismes de protection du noyau

#### Vérifications à effectuer et à documenter

```bash
# Confirmer l'identité de l'utilisateur courant (attendu : uid=0(root))
adb shell id

# Vérifier l'état du démarrage vérifié
adb shell getprop ro.boot.verifiedbootstate

# Vérifier le mode dm-verity
adb shell getprop ro.boot.veritymode

# Inspecter l'état du vbmeta
adb shell getprop ro.boot.vbmeta.device_state

# Tester la disponibilité de su dans le shell
adb shell "su -c id"
```

**Interprétation des résultats :**

| Résultat | Signification |
|---|---|
| `uid=0(root)` | Droits root obtenus avec succès |
| `verifiedbootstate = green` | Système signé et intègre |
| `verifiedbootstate = orange/yellow` | Système modifié, intégrité non garantie |
| `verifiedbootstate = red` | Intégrité compromise — à documenter impérativement |

#### Si dm-verity bloque le remontage

```bash
adb disable-verity   # Désactiver la vérification d'intégrité des partitions
adb reboot           # Redémarrer pour appliquer le changement
adb remount          # Remonter après le redémarrage
```

> **Concept de sécurité :** dm-verity vérifie en temps réel que les blocs du système de fichiers n'ont pas été altérés. Le désactiver, c'est retirer le scellé de sécurité d'un produit : la modification devient possible, mais la garantie d'intégrité disparaît.

#### Journalisation de l'état

```bash
adb logcat -d | tail -n 200 > journal_elevation_privileges.txt
```

> **Bonne pratique d'audit :** Conserver systématiquement des traces horodatées de chaque action. Ces journaux permettent de reconstituer la chronologie des événements et de documenter la démarche pour le rapport.

---

### Étape 5 — Fastboot (matériel de laboratoire uniquement)

> **Avertissement :** Ces commandes ne s'appliquent qu'à un appareil physique de laboratoire avec bootloader déverrouillé. Toute manipulation fastboot sur un appareil personnel risque de le rendre inutilisable ("brick") et invalide définitivement la garantie constructeur.

```bash
fastboot oem device-info         # Informations sur l'état du bootloader
fastboot getvar avb_boot_state   # État AVB
fastboot boot image_patched.img  # Démarrage temporaire (sans flasher)
```

> **Qu'est-ce que Magisk ?** Magisk est un outil de rooting dit "systemless" : il modifie l'image de démarrage sans toucher à la partition système, ce qui lui permet de contourner certaines détections de root par les applications.

---

## Phase 3 — Analyse des mécanismes de sécurité

### Étape 6 — Architecture de sécurité Android (synthèse, 6 lignes max)

Référence : [https://source.android.com/docs/security](https://source.android.com/docs/security)

Rédiger dans le rapport une synthèse couvrant ces trois piliers :

- **Isolation des processus (sandboxing) :** chaque application s'exécute dans son propre environnement cloisonné
- **Modèle de permissions :** l'accès aux ressources sensibles nécessite une autorisation explicite de l'utilisateur
- **Intégrité du système :** des mécanismes matériels et logiciels empêchent les modifications non autorisées

> **Analogie :** Le sandboxing, c'est comme attribuer à chaque application sa propre salle de classe fermée. Les permissions, c'est devoir demander l'autorisation avant d'utiliser certains équipements partagés. L'intégrité système, c'est verrouiller la structure du bâtiment pour qu'elle ne puisse pas être modifiée en secret.

---

### Étape 7 — Verified Boot : principe et vérification

Référence : [https://source.android.com/docs/security/features/verifiedboot](https://source.android.com/docs/security/features/verifiedboot)

Répondre aux trois questions suivantes dans le rapport :

1. **Objectif principal de Verified Boot :** garantir que le système qui démarre est exactement celui prévu par le fabricant, sans aucune altération
2. **Définition de la "chaîne de confiance" (2 lignes) :** succession de vérifications où chaque composant authentifie le suivant avant de lui transférer le contrôle — si un maillon est compromis, la chaîne entière est considérée non fiable
3. **Pourquoi l'intégrité au démarrage est critique :** un système compromis dès le démarrage peut neutraliser toutes les protections ultérieures

```bash
# Vérifier l'état du démarrage vérifié sur l'AVD
adb shell getprop ro.boot.verifiedbootstate
```

**Signification des états :**

| Couleur | Signification |
|---|---|
| `green` | Système signé et intègre — état nominal |
| `yellow` | Signature personnalisée — système fonctionnel mais non officiel |
| `orange` | Bootloader déverrouillé — modifications possibles |
| `red` | Intégrité compromise — démarrage potentiellement dangereux |

> **Analogie :** Verified Boot fonctionne comme un système de détection d'intrusion qui vérifie si quelqu'un a modifié les accès de votre bâtiment pendant votre absence. En cas d'anomalie, il refuse l'entrée ou déclenche une alerte.

---

### Étape 8 — AVB (Android Verified Boot 2.0)

Référence : [https://source.android.com/docs/security/features/verifiedboot/avb](https://source.android.com/docs/security/features/verifiedboot/avb)

Rédiger 3 lignes dans le rapport sur les apports d'AVB par rapport à Verified Boot 1.0 :
- Vérification d'intégrité plus granulaire (par partition)
- Protection anti-rollback empêchant l'installation de versions antérieures vulnérables
- Architecture modulaire adaptée aux appareils sans partition dédiée

```bash
# Sur matériel de laboratoire avec fastboot disponible
fastboot getvar avb_boot_state
```

> **Protection anti-rollback :** Ce mécanisme empêche de rétrograder vers une ancienne version du système d'exploitation qui pourrait contenir des failles de sécurité connues — comme empêcher de remplacer une serrure moderne par un modèle obsolète facilement crochetable.

---

### Étape 9 — Référentiel OWASP MASVS (2 exigences)

Référence : [https://mas.owasp.org/MASVS/](https://mas.owasp.org/MASVS/)

> **Contexte :** Le MASVS (*Mobile Application Security Verification Standard*) est le référentiel de l'OWASP définissant les exigences de sécurité pour les applications mobiles. Il structure l'évaluation de sécurité selon des domaines thématiques.

Résumer deux exigences dans le rapport. Exemples de domaines pertinents pour ce lab :

**MASVS-STORAGE :** Les données sensibles (identifiants, tokens, clés cryptographiques) ne doivent jamais être stockées en clair sur le stockage local. Leur accès doit être conditionné à une authentification.

**MASVS-NETWORK :** Toutes les communications réseau doivent être chiffrées via TLS avec une configuration correcte. Les certificats doivent être validés et les connexions non sécurisées (HTTP) doivent être bloquées.

> **Lien avec ce lab :** Avec les droits root, vous pouvez directement accéder aux fichiers de stockage d'une application (`/data/data/[package]/`) et vérifier si ces exigences sont respectées ou si l'application se repose uniquement sur les protections système.

---

### Étape 10 — OWASP MASTG (2 procédures de test)

Référence : [https://mas.owasp.org/MASTG/](https://mas.owasp.org/MASTG/)

> **Relation MASVS / MASTG :** Si le MASVS définit *ce qu'il faut vérifier*, le MASTG (*Mobile Application Security Testing Guide*) explique *comment le vérifier*. C'est votre guide opérationnel de test.

Documenter deux procédures de test dans le rapport :

1. **Inspection du stockage local :** avec les droits root, examiner le contenu de `/data/data/[package_name]/shared_prefs/` et `/data/data/[package_name]/databases/` pour détecter des données sensibles stockées en clair

2. **Analyse des journaux d'exécution :** surveiller les sorties de l'application en temps réel via `adb logcat` pendant l'exécution des scénarios définis, afin de détecter d'éventuelles fuites d'informations dans les logs

---

## Phase 4 — Documentation et gestion des risques

### Étape 11 — Définition du rooting (4 phrases à compléter)

Inclure dans le rapport une définition structurée en 4 points :

- Les droits root correspondent aux privilèges d'administration les plus élevés du système Android
- Leur obtention modifie fondamentalement les garanties d'intégrité et les mécanismes de confiance du système
- Dans un contexte de laboratoire contrôlé, ces droits permettent d'observer des comportements système normalement inaccessibles
- Leur usage requiert un environnement rigoureusement isolé, une traçabilité complète et une remise à zéro systématique en fin de session

---

### Étape 12 — Intérêt pédagogique (contexte laboratoire uniquement)

Compléter dans le rapport : *"Dans un environnement de laboratoire autorisé, les droits élevés permettent de..."*

- Inspecter les artefacts système normalement protégés par le cloisonnement des applications
- Examiner le comportement d'une application au niveau système, au-delà de ce qu'exposent ses interfaces publiques
- Évaluer la robustesse des mécanismes de protection des données face à un attaquant disposant d'un accès privilégié
- Vérifier si une application se repose uniquement sur les protections du système ou implémente ses propres mesures de sécurité

Mentionner explicitement : **"Laboratoire autorisé uniquement — ces manipulations sont illégales en dehors d'un cadre explicitement consenti."**

---

### Étape 13 — Matrice risques / contre-mesures

Compléter le tableau suivant dans le rapport :

| # | Risque identifié | Niveau | Contre-mesure associée |
|---|---|---|---|
| 1 | Intégrité du système non garantie → résultats potentiellement biaisés | Élevé | Documenter l'état de l'environnement avant chaque session |
| 2 | Surface d'exposition accrue si l'appareil quitte le périmètre de test | Élevé | Conserver l'appareil dans l'environnement isolé à tout moment |
| 3 | Données sensibles exposées si présentes sur l'appareil | Critique | Utiliser exclusivement des données fictives générées pour les tests |
| 4 | Instabilité système rendant les tests non reproductibles | Moyen | Prendre des snapshots de l'AVD avant chaque manipulation |
| 5 | Confusion entre comptes personnels et comptes de test | Élevé | Ne jamais configurer de compte personnel sur l'appareil de test |
| 6 | Résidus de test persistant après la session | Élevé | Effectuer une remise à zéro complète et documentée en fin de séance |
| 7 | Communications non contrôlées vers des systèmes externes | Moyen | Utiliser un réseau isolé ou un proxy d'interception dédié |
| 8 | Absence de traçabilité rendant l'audit impossible | Élevé | Horodater et capturer chaque étape significative |

---

### Étape 14 — Fiche périmètre (à inclure dans le rapport)

```
=== FICHE PÉRIMÈTRE — SESSION DE TEST ===

Application testée    : [Nom] v[Version]
Support utilisé       : [AVD / Appareil de laboratoire]
Version Android / API : [Ex. Android 13 / API 33]
Objectif déclaré      : Observer les mécanismes de protection Android face aux droits élevés
Périmètre des données : Données fictives uniquement — aucune donnée réelle
Configuration réseau  : Réseau de test isolé
Analyste              : [Prénom Nom]
Date de session       : [Date]
Durée estimée         : [Durée]
```

> **Pourquoi la fiche périmètre est indispensable :** Elle formalise le "contrat" de la session de test. Sans périmètre défini, un test de sécurité n'a pas de valeur probante et sa légitimité ne peut pas être établie.

---

### Étape 15 — Journal de session et traçabilité (1 page)

Compléter les champs suivants et joindre les captures correspondantes :

```bash
# Créer le journal de session
cat > journal_session_$(date +%Y%m%d).txt << EOF
Date et heure de début : $(date)
Analyste               : [Prénom Nom]
Support                : [AVD / Device labo]
Version Android / API  : [Version]
Application            : [Nom] v[Version]
Scénarios prévus       : [3 scénarios définis à l'étape 3]
Observations           : [À compléter au fil de la session]
Limites identifiées    : [À compléter]
Remise à zéro          : [Oui / Non] — Preuve : [Référence capture]
EOF
```

**Captures obligatoires à annexer :**

- Application démarrée sur l'AVD
- Sortie de `adb root` et `adb shell id`
- Sortie de `adb shell getprop ro.boot.verifiedbootstate`
- Preuve de remise à zéro en fin de session

> **Conseil :** Annoter chaque capture d'écran pour pointer précisément les éléments significatifs. Une capture non annotée perd l'essentiel de sa valeur probante dans un rapport.

---

## Phase 5 — Remise à zéro (obligatoire)

### Étape 16 — Réinitialisation de l'AVD

> **Pourquoi c'est non négociable :** Ne pas remettre à zéro l'environnement, c'est laisser un laboratoire avec des substances dangereuses sur les paillasses — risque pour la prochaine session et contamination des résultats futurs.

**Via l'interface Android Studio :**

Android Studio → *Device Manager* → *Wipe Data* (ou supprimer et recréer l'AVD)

**Via la ligne de commande :**

```bash
adb emu avd stop
adb emu avd wipe-data
```

**Preuve requise :** Capture de l'assistant de configuration initial d'Android apparaissant au redémarrage.

---

### Étape 17 — Réinitialisation d'un appareil physique de laboratoire (si utilisé)

1. *Paramètres* → *Système* → *Options de réinitialisation* → *Effacer toutes les données*
2. Vérifier l'absence de : comptes configurés, profils, certificats personnalisés, applications tierces
3. Capturer l'écran de l'assistant de configuration initial comme preuve

**Via fastboot (laboratoire uniquement) :**

```bash
fastboot erase userdata   # Effacement de bas niveau de la partition données
```

> **Différence importante :** La réinitialisation via les paramètres système efface les données utilisateur mais peut laisser des traces dans certaines partitions. La commande fastboot effectue un effacement de bas niveau plus complet et définitif.

**Vérification complémentaire :** Après réinitialisation, contrôler l'absence de certificats racine personnalisés dans *Paramètres → Sécurité → Certificats de confiance*, qui pourraient permettre l'interception de trafic HTTPS.

---

## Phase 6 — Livrables attendus

### Rapport final (1 à 2 pages)

Le rapport doit inclure, dans cet ordre :

| Livrable | Contenu attendu |
|---|---|
| Fiche périmètre | Complétée avec tous les champs |
| Définition du rooting | 4 phrases structurées |
| Synthèse sécurité Android | 6 lignes maximum |
| Schéma Verified Boot / AVB | Représentation visuelle de la chaîne de confiance |
| Matrice risques/contre-mesures | 8 risques + 8 mesures, tableau complet |
| MASVS | 2 exigences résumées avec référence |
| MASTG | 2 procédures de test décrites |
| Fiche environnement | Journal de session complété |
| Preuve de remise à zéro | Capture de l'état initial post-reset + checklist signée |

> **Conseil de présentation :** Privilegier les tableaux et les schémas sur les blocs de texte. Un bon rapport de sécurité doit rester lisible par un interlocuteur non spécialiste.

---

## Synthèse des commandes

### Commandes essentielles

```bash
# Vérification de la connexion
adb devices

# Élévation de privilèges
adb root
adb remount

# Vérifications système
adb shell id
adb shell getprop ro.boot.verifiedbootstate
adb shell getprop ro.boot.veritymode
adb shell getprop ro.boot.vbmeta.device_state
adb shell "su -c id"

# Journalisation
adb logcat -d | tail -n 200 > journal_elevation_privileges.txt
```

### Option dm-verity

```bash
adb disable-verity
adb reboot
adb remount
```

### Fastboot (laboratoire uniquement)

```bash
fastboot oem device-info
fastboot getvar avb_boot_state
fastboot boot image_personnalisee.img   # Démarrage temporaire, sans flash permanent
fastboot erase userdata                 # Remise à zéro complète
```

### Remise à zéro AVD

```bash
adb emu avd stop
adb emu avd wipe-data
```

---

## Dépannage

| Problème | Cause probable | Solution |
|---|---|---|
| `adb root` → "cannot run as root in production builds" | Image de production — root non autorisé | Utiliser une image d'émulateur (non `_r` / non Google Play) |
| `adb remount` → "remount failed" | dm-verity actif | Exécuter `adb disable-verity` puis redémarrer |
| `adb devices` ne détecte rien | Émulateur non démarré ou ADB non dans le PATH | Démarrer l'émulateur, vérifier `echo $PATH` |
| Commande `su` introuvable | Image sans `su` binaire | Utiliser une image AOSP générique ou installer un binaire `su` compatible |
| `fastboot` ne détecte pas l'appareil | Drivers USB non installés ou mode fastboot non activé | Installer les drivers constructeur, maintenir Volume Bas au démarrage |

---

## Schémas de référence

### Architecture de sécurité Android (vue simplifiée)

```
[Matériel sécurisé / TEE]
        ↓
[Bootloader vérifié et signé]
        ↓
[Noyau Linux durci]
        ↓
[Système Android (partitions signées)]
        ↓
[Applications — chacune dans son sandbox]
```

### Impact de l'élévation de privilèges

```
Fonctionnement normal :
  Application → Sandbox → Contrôle des permissions → Ressources système (protégées)

Après élévation (root) :
  Application → Sandbox → Contrôle des permissions → Ressources système (accessibles)
                                                               ↑
                                                         Accès root direct
```

### Chaîne de confiance au démarrage

```
ROM de démarrage (immuable)
      ↓ vérifie
Bootloader (signature constructeur)
      ↓ vérifie
Partition boot (noyau + ramdisk)
      ↓ vérifie
Partition système (Android OS)
      ↓ vérifie
Applications (signatures développeur)
```

---

## Références

| Ressource | URL |
|---|---|
| Architecture de sécurité Android | https://source.android.com/docs/security |
| Verified Boot | https://source.android.com/docs/security/features/verifiedboot |
| Android Verified Boot 2.0 | https://source.android.com/docs/security/features/verifiedboot/avb |
| OWASP Mobile Application Security | https://mas.owasp.org/ |
| OWASP MASVS | https://mas.owasp.org/MASVS/ |
| OWASP MASTG | https://mas.owasp.org/MASTG/ |
| Documentation ADB | https://developer.android.com/tools/adb |
| Documentation AVD | https://developer.android.com/studio/run/managing-avds |

---

## Glossaire

| Terme | Définition |
|---|---|
| **ADB** | Android Debug Bridge — interface de communication entre un poste de travail et un appareil Android |
| **AVD** | Android Virtual Device — émulateur Android configurable |
| **AVB** | Android Verified Boot 2.0 — système de vérification d'intégrité du démarrage |
| **Bootloader** | Programme chargé en premier lors du démarrage, responsable du chargement du système |
| **dm-verity** | Mécanisme noyau vérifiant en temps réel l'intégrité des blocs des partitions système |
| **Fastboot** | Mode de bas niveau permettant de flasher les partitions d'un appareil Android |
| **Partition** | Section logique du stockage dédiée à un usage spécifique (system, data, boot…) |
| **Root** | Compte administrateur disposant des droits les plus élevés sur un système Linux/Android |
| **Sandbox** | Environnement d'exécution isolé limitant l'accès d'une application aux ressources système |
| **TEE** | Trusted Execution Environment — zone d'exécution sécurisée isolée du reste du système |
| **Verity** | Voir dm-verity |
| **vbmeta** | Partition contenant les métadonnées de vérification utilisées par AVB |
