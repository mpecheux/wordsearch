# Plan : du jeu web `index.html` à une app Android publiée

Point de départ : un jeu « Mots mêlés 3D » en un seul fichier `index.html` (JS vanilla, ~250 lignes) + `mots.txt`.

## Choix de la technologie

| Option | Effort | Avantages | Inconvénients |
|---|---|---|---|
| **Capacitor (recommandé)** | Faible | Réutilise 100 % du code web, plugins natifs, Android Studio | Nécessite Node + Android Studio |
| TWA / Bubblewrap | Très faible | PWA publiée sur le Play Store | Nécessite un site HTTPS hébergé |
| Cordova | Faible | Mature | Moins actif que Capacitor |
| Réécriture Kotlin / Flutter | Élevé | Natif | Inutile pour ce jeu |

→ On retient **Capacitor** (WebView embarquée, fonctionne hors-ligne).

## Phase 0 – Prérequis
1. Installer : Node.js LTS, JDK 17, Android Studio (SDK + émulateur), Git.
2. Compte **Google Play Console** (25 $ une fois). Vérification d'identité : prévoir quelques jours.
3. Choisir un identifiant de paquet unique, ex. `com.mpecheux.motsmeles3d`.

## Phase 1 – Préparer le code web
1. Structurer le projet :
   ```
   wordsearch/
     package.json
     www/            (index.html, mots.txt, icônes, css/js séparés)
   ```
2. Séparer le JS/CSS de `index.html` (facilite maintenance et CSP).
3. **`mots.txt` : ne pas le charger via `fetch`** (bloqué/instable en `file://`). Capacitor sert en `https://localhost`, donc `fetch` fonctionne, mais l'embarquer dans un `words.js` est plus robuste.
4. Adapter à mobile :
   - `<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">`
   - Événements tactiles/pointer (pas seulement souris), zones tactiles ≥ 48 dp.
   - Orientation et gestion du « safe area » (encoche).
   - Désactiver zoom/sélection de texte involontaires (`touch-action`, `user-select`).
5. Persistance : `localStorage` (meilleurs scores) ou plugin `@capacitor/preferences`.
6. Zéro dépendance CDN → l'app doit marcher hors-ligne.

## Phase 2 – Intégrer Capacitor
```powershell
npm init -y
npm i @capacitor/core @capacitor/cli @capacitor/android
npx cap init "Mots mêlés 3D" com.mpecheux.motsmeles3d --web-dir=www
npx cap add android
npx cap sync android
npx cap open android      # ouvre Android Studio
```
À chaque modification web : `npx cap sync android`.

## Phase 3 – Configuration Android (Android Studio)
1. `AndroidManifest.xml` : ne demander **aucune permission** inutile (le jeu n'en a pas besoin, même pas INTERNET si 100 % hors-ligne).
2. `build.gradle` : `minSdk 24`, `targetSdk` = dernière version exigée par Google Play (vérifier la politique en vigueur), `versionCode` (entier, +1 à chaque release), `versionName`.
3. Icône adaptative (Image Asset Studio) et splash screen (`@capacitor/splash-screen`).
4. Plein écran immersif / barre d'état (`@capacitor/status-bar`), verrouillage d'orientation si besoin.
5. Tester sur émulateur + au moins un vrai appareil (`adb`), perf du rendu 3D (WebGL/CSS 3D) sur appareil modeste.

## Phase 4 – Sécurité
- `android:allowBackup="false"` (ou règles de backup explicites), `android:usesCleartextTraffic="false"`.
- CSP dans `index.html` : `default-src 'self'; img-src 'self' data:; style-src 'self' 'unsafe-inline'`.
- Aucun secret/clé API dans le code (tout APK est décompilable).
- Pas de `eval`, pas de `innerHTML` avec données non maîtrisées.
- Activer **R8/minification** en release ; dépendances à jour (`npm audit`, Dependabot).
- Signature : créer un **keystore** (`keytool -genkey -v -keystore release.jks -alias upload -keyalg RSA -keysize 2048 -validity 10000`). Ne **jamais** le committer (`.gitignore`), sauvegarder en 2 endroits + mot de passe dans un gestionnaire.
- Activer **Play App Signing** (Google garde la clé de signature finale, vous gardez la clé d'upload).
- Si vous ajoutez pubs/analytics : déclarer les données collectées + consentement RGPD.

## Phase 5 – Qualité et tests
1. Tests manuels : rotation, retour arrière, mise en veille, petits écrans, Android 7 → dernière version.
2. Tests auto (optionnel) : Jest pour la logique de grille/mots ; Playwright pour le web.
3. **Pre-launch report** de la Play Console (tests automatiques sur appareils réels).
4. Accessibilité : contrastes, tailles de texte, TalkBack minimal.
5. Les nouveaux comptes personnels doivent faire un **test fermé (≈12 testeurs pendant 14 jours)** avant l'accès à la production — à anticiper.

## Phase 6 – Build de release
1. Android Studio : *Build > Generate Signed Bundle* → **AAB** (format obligatoire pour le Play Store).
   Ou CLI : `cd android; .\gradlew bundleRelease`.
2. Vérifier : taille, `versionCode`, installation via `bundletool` pour tester l'AAB.

## Phase 7 – Documentation et éléments légaux
- `README.md` : description, build, structure, licence.
- `CHANGELOG.md` (versionnage sémantique).
- **Politique de confidentialité** (obligatoire, URL publique – GitHub Pages suffit) : même si « aucune donnée collectée ».
- Choisir une licence (MIT, etc.) ; vérifier les droits sur la liste de mots.
- Fiche Play Store :
  - Titre (≤30 car.), description courte (≤80), description longue (≤4000)
  - Icône 512×512, bannière 1024×500
  - ≥2 captures d'écran téléphone (idéalement 4–8) + tablettes si ciblées
  - Catégorie (Jeux > Mots), langue(s), email de contact

## Phase 8 – Publication (Play Console)
1. Créer l'application (nom, langue, gratuite/payante).
2. Remplir : **Data safety**, classification du contenu (IARC), public cible, déclaration des pubs, accès à l'app.
3. Piste **Test interne** → **Test fermé** (obligatoire, voir Phase 5) → **Production**.
4. Téléverser l'AAB, notes de version, pays de distribution.
5. Envoyer en examen (de quelques heures à quelques jours).

## Phase 9 – Après publication
- Suivre Android vitals (crashs/ANR), avis utilisateurs.
- Corriger via `versionCode+1` → `npx cap sync` → nouveau AAB → mise à jour par déploiement progressif.
- CI/CD (GitHub Actions) : build + signature avec secrets chiffrés.
- Idées d'évolution : niveaux/thèmes, sons, classement local, multilingue.

## Check-list rapide
- [ ] Jeu utilisable au tactile, hors-ligne
- [ ] Capacitor + projet Android générés
- [ ] Icône, splash, permissions minimales
- [ ] CSP, backup désactivé, keystore sauvegardé
- [ ] AAB signé testé sur appareil réel
- [ ] Politique de confidentialité en ligne
- [ ] Fiche store + captures complètes
- [ ] Test fermé 14 jours terminé
- [ ] Publication en production

## Ordre de grandeur
Adaptation + Capacitor : ½–1 jour · Tests/assets/fiche store : 1–2 jours · Test fermé : 14 jours · Validation Google : 1–7 jours.
