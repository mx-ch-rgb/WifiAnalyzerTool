# WifiAnalyzerTool

> Outil web autonome d'analyse Wi-Fi pour audits IT et consulting réseau, basé sur les données collectées par l'application **Centra-Scan Wifi**.

**Auteur** : Marc Christiaens · `mx-ch`
**Version** : 1.0
**Type** : Application HTML statique (single-file, hébergeable sur GitHub Pages)

---

## Présentation

WifiAnalyzerTool est un dashboard d'analyse Wi-Fi qui transforme un scan brut en rapport d'audit professionnel exportable en PDF. Il s'inscrit dans une chaîne d'outils :

```
┌──────────────────┐    JSON    ┌─────────────────────┐    PDF
│  Centra-Scan     │  ───────▶  │  WifiAnalyzerTool   │  ─────▶  rapport client
│  Wifi (macOS)    │   v1       │  (HTML autonome)    │
└──────────────────┘            └─────────────────────┘
```

L'outil évalue le Wi-Fi audité selon les trois axes méthodologiques de référence :

- **Spectrum Quality** — qualité du spectre radio par bande (2,4 / 5 / 6 GHz)
- **Client Behavior** — comportement de la machine de scan (RSSI, débit négocié)
- **Broadcast & Multicast Control** — bruit de découverte (mDNS, SSDP)

Chaque axe est noté sur 100 et matérialisé par un cadran EFIS. L'analyse débouche sur un plan d'action adapté au modèle de routeur déployé.

---

## Fonctionnalités

### Saisie & import
- Identification client : date, nom, adresse, contact, référence dossier, marque/modèle routeur, nombre et emplacement des bornes Wi-Fi
- Import du JSON Centra-Scan Wifi (drag-drop ou sélection fichier)
- Validation du schéma `wifi-scanner/v1`
- Jeu de démonstration intégré
- Téléchargement du template JSON vide

### Analyse
- 5 cadrans demi-cercle EFIS animés (score global + 4 axes)
- Bandes 2,4 / 5 / 6 GHz détectées automatiquement à partir des numéros de canal
- Texte explicatif adaptatif selon le score (OPTIMAL / À SURVEILLER / CRITIQUE)
- Tableau complet des réseaux détectés (tri par RSSI, code couleur)
- Synthèse des anomalies par sévérité
- Métadonnées du scan (host, interface, méthode)

### Recommandations
- 7 profils de routeurs supportés avec procédures détaillées :
  - **UniFi** (Ubiquiti)
  - **Sophos** (Central / SFOS)
  - **Meraki** (Cisco)
  - **TP-Link** (Omada)
  - **D-Link** (Nuclias)
  - **Netgear** (Insight)
  - **Synology** (SRM 1.3 / 1.4)
- Plan d'action priorisé : `CRITIQUE` → `ALERTE` → `INFO`
- Chemins de menus exacts par marque (paths UI + commandes CLI lorsque pertinent)

### Export & persistance
- **Export PDF** propre (impression navigateur) : exclusion automatique de la page Import, pas de pages vides
- **Export JSON** : rapport complet (données + analyse + identification client)
- **Reset** : réinitialisation complète des données via bouton dédié
- **Persistance** : localStorage, conservation des données au refresh

---

## Mode d'emploi

### 1. Installation

L'outil est un **fichier HTML unique**, sans build ni dépendance serveur :

```bash
# Option A : ouvrir localement
open WifiAnalyzerTool.html

# Option B : héberger sur GitHub Pages
# placer WifiAnalyzerTool.html dans un repo public, activer Pages
```

Aucune installation requise. Compatible Chrome, Safari, Firefox.

### 2. Workflow type

| Étape | Action |
|---|---|
| 1 | **Audit terrain** — lancer Centra-Scan Wifi sur Mac, scanner sur site, exporter le JSON |
| 2 | **Identification** — onglet `01 Client`, renseigner les infos audit |
| 3 | **Import** — onglet `02 Import`, glisser-déposer le JSON Centra-Scan Wifi |
| 4 | **Analyse** — onglet `03 Analyse`, lecture des cadrans + tableau des réseaux |
| 5 | **Corrections** — onglet `04 Corrections`, sélection de la marque routeur, application des recos |
| 6 | **Livraison** — bouton `EXPORT PDF` → rapport client final |

### 3. Format JSON attendu

Schéma `wifi-scanner/v1` produit par Centra-Scan Wifi. Sections requises :

```json
{
  "schema": "wifi-scanner/v1",
  "version": "1.0.0",
  "timestamp": "2026-05-10T08:48:58Z",
  "host": { "hostname": "Mac.local", "wifi_interface": "en0" },
  "current_association": { "ssid": "...", "rssi": -38, "channel": 100, "tx_rate": "1200" },
  "scan": {
    "method": "system_profiler",
    "count": 12,
    "networks": [
      { "ssid": "...", "rssi": -37, "channel": 100, "security": "WPA2 Personal", "associated": true }
    ]
  },
  "capture": {
    "interface": "en0",
    "duration_seconds": 31,
    "total": 34,
    "mdns": 34,
    "ssdp": 0
  }
}
```

Section optionnelle : `diagnostics` (réutilisée pour enrichir le rapport si présente).

---

## Logique de scoring

### Spectrum Quality (par bande)
- Détection automatique de la bande à partir du canal :
  - canal 1-14 → 2,4 GHz
  - canal 32-177 → 5 GHz
  - canal > 177 → 6 GHz (Wi-Fi 6E)
- Pénalités appliquées :
  - densité de voisins (4 pts par voisin, max 35 pts)
  - canaux co-occupés (10 pts si 2 réseaux, 20 pts si 3+)
  - voisins forts > -65 dBm (10 pts par voisin)
  - voisins moyens -65 à -80 dBm (4 pts par voisin)

### Client Behavior
- Évalué sur la machine de scan (`current_association`)
- RSSI : bonus -50 dBm, pénalité progressive jusqu'à -75 dBm
- TX rate : pondération selon palier (1000 / 500 / 200 Mbps)

### Broadcast & Multicast
- Calcul des paquets/seconde depuis `capture.duration_seconds`
- mDNS : pénalité progressive à partir de 1 pps
- SSDP/uPnP : pénalité dès la première occurrence (indésirable en environnement pro)
- Volume total : pénalité au-delà de 10 pps

### Code couleur
| Score | Statut | Couleur |
|---|---|---|
| ≥ 75 | OPTIMAL | Vert |
| 50 – 74 | À SURVEILLER | Ambre |
| < 50 | CRITIQUE | Rouge |

---

## Architecture technique

| Aspect | Choix |
|---|---|
| Forme | Fichier HTML unique, ~85 ko |
| Dépendances | Google Fonts (Rajdhani + JetBrains Mono) uniquement |
| JavaScript | Vanilla, pas de framework |
| Persistance | `localStorage` clé `wat_state_v1` |
| Esthétique | Dark EFIS (instrumentation aviation) |
| Export PDF | API navigateur `window.print()` + stylesheet `@media print` dédié |
| PWA | Manifest inline, installable sur Mac/iOS/Android |

---

## Structure du dépôt suggérée pour GitHub

```
_WifiAnalyzerTool/
├── WifiAnalyzerTool.html      ← l'outil
├── README.md                   ← ce fichier
├── samples/
│   └── scan_demo.json          ← exemple de scan Centra-Scan Wifi
└── docs/
    └── screenshots/            ← captures du dashboard
```

---

## Roadmap

- [ ] Comparaison multi-scans (avant/après corrections)
- [ ] Heatmap de couverture si plusieurs scans depuis emplacements différents
- [ ] Profil routeur additionnel : Aruba Instant On
- [ ] Export DOCX en alternative au PDF
- [ ] Internationalisation (FR / EN)

---

## Licence

**Licence d'évaluation.** Copyright © 05/2026 M. Christiaens — `mx-ch`. Tous droits réservés.

L'auteur concède à l'utilisateur une licence limitée, non exclusive et non transférable pour installer et utiliser ce logiciel uniquement à des fins de test et d'évaluation.

Il est strictement interdit à l'utilisateur, sans l'accord écrit préalable de l'auteur :

- de modifier, adapter, traduire tout ou partie du code source ou du logiciel,
- de décompiler, désassembler ou procéder à toute forme d'ingénierie inverse,
- de redistribuer, publier, transférer, prêter ou vendre le logiciel, en tout ou partie, à des tiers,
- d'utiliser le logiciel à des fins commerciales.

Le logiciel est fourni « en l'état », sans aucune garantie de quelque nature que ce soit. L'auteur ne pourra en aucun cas être tenu responsable de tout dommage direct ou indirect résultant de l'utilisation du logiciel.
