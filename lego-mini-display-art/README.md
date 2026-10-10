# LEGO Display – Panneau d'affichage pour ville LEGO

Un mini panneau d'affichage pour ville LEGO basé sur un **ESP32** et un écran **ST7735**. Il affiche en boucle une série de posters (affiches de films, publicités, etc.) téléchargés depuis un serveur web local, avec une liste modifiable sans reflasher l'ESP.

## Fonctionnalités

- Affichage de posters PNG 96x128 sur écran ST7735 (128x160)
- Rotation automatique toutes les 30 secondes
- Liste des posters dynamique : un simple fichier `liste.txt` sur le serveur
- Rafraîchissement automatique de la liste toutes les 10 minutes
- Intégration Home Assistant (API ESPHome) et mises à jour OTA
- Rétroéclairage contrôlable depuis Home Assistant

## Matériel

| Composant | Détail |
|-----------|--------|
| Microcontrôleur | ESP32 (`esp32dev`) |
| Écran | ST7735 128x160 (SPI) |
| Serveur web local | N'importe quelle machine du réseau (NAS, Raspberry Pi, PC...) |

### Câblage

| Écran ST7735 | ESP32 |
|--------------|-------|
| SCL / CLK | GPIO18 |
| SDA / MOSI | GPIO23 |
| CS | GPIO27 |
| DC | GPIO21 |
| RESET | GPIO22 |
| BL (rétroéclairage) | GPIO25 |
| VCC | 3V3 |
| GND | GND |

## Installation

### 1. Préparer le serveur web

Sur une machine du réseau, créez un dossier contenant vos posters et le fichier de liste :

```
posters/
├── blade_runner.png
├── alien.png
├── matrix.png
└── liste.txt
```

**Contraintes sur les images :**
- Format **PNG**
- Taille exacte **96x128 pixels** (une image plus grande provoque une erreur de mémoire)

Pour redimensionner en lot (ImageMagick) :

```bash
mogrify -resize 96x128! *.png
```

**`liste.txt`** : un nom de fichier par ligne.

```
blade_runner.png
alien.png
matrix.png
```

Pour le générer automatiquement :

```bash
ls *.png > liste.txt
```

**Servir le dossier** (exemple avec Python, port 8080) :

```bash
cd posters
python3 -m http.server 8080
```

Vérifiez dans un navigateur que `http://IP_DU_SERVEUR:8080/liste.txt` s'affiche correctement.

### 2. Configurer ESPHome

1. Copiez `lego-display-1.yaml` dans votre dossier ESPHome.
2. Créez ou complétez `secrets.yaml` :

   ```yaml
   wifi_ssid: "MonWifi"
   wifi_password: "MonMotDePasse"
   lego_display_1__encryption_key: "votre_cle_api"
   ```

3. Adaptez l'adresse du serveur dans le YAML :

   ```yaml
   substitutions:
     serveur: "http://192.168.1.10:8080"
   ```

4. Compilez et flashez :

   ```bash
   esphome run lego-display-1.yaml
   ```

### 3. Vérifier le fonctionnement

Dans les logs, vous devriez voir :

```
[liste] 3 posters charges
[liste] Poster 0 : http://192.168.1.10:8080/blade_runner.png
```

Puis un nouveau poster toutes les 30 secondes.

## Fonctionnement

1. Au démarrage (après 5 s), l'ESP télécharge `liste.txt` et la stocke en mémoire.
2. Toutes les 30 s, il passe au poster suivant : il libère l'image précédente, télécharge la nouvelle, puis redessine l'écran.
3. Toutes les 10 min, la liste est rechargée : ajouter ou retirer un poster ne demande **aucun reflash**.

## Personnalisation

| Paramètre | Où | Valeur par défaut |
|-----------|----|-------------------|
| Durée d'affichage d'un poster | `interval: 30s` | 30 s |
| Fréquence de rechargement de la liste | `interval: 10min` | 10 min |
| Position de l'image | `it.image(16, 16, ...)` | centrée |
| Orientation de l'écran | `rotation` | `0°` |

## Dépannage

| Problème | Cause probable | Solution |
|----------|----------------|----------|
| `Out of memory` / image non affichée | Image trop grande | Redimensionner en 96x128 |
| `Echec telechargement liste` | Serveur injoignable ou mauvaise IP | Tester l'URL depuis un navigateur |
| `Liste vide` | `liste.txt` vide ou introuvable | Vérifier le contenu et l'emplacement |
| Écran blanc ou décalé | Mauvais modèle ou offsets | Essayer un autre `model` (`INITR_BLACKTAB`, `INITR_GREENTAB`...) ou ajuster `col_start` / `row_start` |
| Écran noir | Rétroéclairage éteint | Activer l'entité `Backlight` dans Home Assistant |
| Erreur de compilation `http_request` | API modifiée selon la version | Utiliser `http_request.get` avec `capture_response: true` (voir le YAML) |

## Structure du dépôt

```
.
├── lego-display-1.yaml    # Configuration ESPHome
├── secrets.yaml.example   # Modèle de fichier de secrets
├── posters/               # Exemples de posters (optionnel)
└── README.md
```

> ⚠️ Ne publiez jamais votre vrai `secrets.yaml`. Ajoutez-le à votre `.gitignore`.

## Prérequis

- ESPHome 2026.4.0 ou supérieur
- Un serveur HTTP accessible depuis l'ESP sur le réseau local

## Licence

MIT (ou la licence de votre choix)
