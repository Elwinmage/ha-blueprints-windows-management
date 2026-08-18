# 🪟 Surveillance fenêtre — aération par pièce

**Version : 1.2.0**

Blueprint Home Assistant qui surveille une fenêtre ouverte et alerte quand il n'est plus utile de laisser la pièce s'aérer. Les entités sont **détectées automatiquement** dans la zone via `area_entities()`. Optionnellement, coupe et rallume la climatisation en fonction de l'état de la fenêtre.

Une instance du blueprint est créée **par pièce**.

---

## Fonctionnement

Le blueprint évalue toutes les N minutes le delta de température entre l'intérieur de la pièce et l'extérieur :

```
delta = T_intérieur − T_extérieur
```

Quand la fenêtre est ouverte et que le delta passe sous le seuil configuré, une notification est envoyée. Si la tendance est activée, l'alerte est supprimée quand la pièce se refroidit naturellement (inutile d'alerter si la température baisse déjà toute seule).

### Détection automatique des entités

Le blueprint utilise `area_entities()` pour scanner la zone sélectionnée et y trouver automatiquement :

| Entité recherchée | Critère de détection |
|---|---|
| Capteur température intérieure | `sensor.*` + `device_class: temperature` |
| Capteur de tendance | `binary_sensor.*` + `"tendance"` dans l'entity_id |
| Fenêtre (input_boolean) | `input_boolean.*` + `"fenetre"` dans l'entity_id |
| Climatisation | `climate.*` (premier trouvé dans la zone) |

Si plusieurs capteurs de température sont présents dans la zone, leurs valeurs sont **moyennées** automatiquement.

Le capteur extérieur est le seul à sélectionner manuellement (partagé entre toutes les instances).

### Logique complète

```
Toutes les N minutes
  └─ Fenêtre ouverte depuis X (délai configurable) ?
       └─ delta calculable et < seuil ?
            └─ (si tendance activée) température montante ?
                 └─ → Notification "ferme la fenêtre"

Fenêtre ouverte depuis X (délai configurable)
  └─ Gestion clim activée ?
       └─ Clim en marche ?
            ├─ oui → mémoriser + couper la clim
            └─ non → réinitialiser la mémoire
  (fenêtre refermée avant X → rien ne se passe : ni coupure clim, ni notif)

Fenêtre fermée (événement, immédiat)
  └─ Gestion clim activée ?
       └─ Clim était en marche ?
            └─ oui → rallumer la clim + effacer la mémoire
```

### Délai de prise en compte de l'ouverture

Un délai unique (**Délai de prise en compte de l'ouverture**, défaut : 1 minute)
temporise les deux réactions à l'ouverture :

| Réaction | Temporisée par | Mécanisme |
|---|---|---|
| Coupure de la clim | oui | `for:` sur le trigger `fenetre_ouverte` |
| Notification d'aération | oui | `for:` sur la condition d'état du cycle de 2 min |
| Rallumage de la clim | **non — immédiat** | trigger `fenetre_fermee` |
| Reset du flag de notif | **non — immédiat** | trigger `fenetre_fermee` |

Conséquences :

- une ouverture plus courte que le délai n'a **aucun effet** : pas de coupure
  clim, pas de notification ;
- le décompte s'appuie sur le `last_changed` de la fenêtre, il est donc
  **restauré après un redémarrage** de Home Assistant ;
- la notification étant portée par le cycle de 2 minutes, elle part au premier
  cycle qui suit la fin du délai (granularité de 2 min, inchangée) ;
- mettre `0` restaure le comportement de la v1.0 (réaction dès l'ouverture).

---

## Prérequis

### Convention de nommage

La détection automatique repose sur la présence de mots-clés dans les `entity_id`. Respecter ces conventions :

| Entité | Mot-clé requis dans l'entity_id | Exemple |
|---|---|---|
| Fenêtre | `fenetre` | `input_boolean.fenetre_salon` |
| Tendance | `tendance` | `binary_sensor.tendance_temperature_salon` |
| Mémoire clim | libre (sélection manuelle) | `input_boolean.clim_salon_etait_allumee` |

Les capteurs de température et les entités `climate` sont détectés uniquement par domaine et `device_class`, sans contrainte de nommage.

### Entités à créer par pièce

**Obligatoire** — l'`input_boolean` fenêtre (via UI ou `configuration.yaml`) :
```yaml
input_boolean:
  fenetre_salon:
    name: Fenêtre Salon
    icon: mdi:window-closed-variant
```

**Si gestion clim activée** — l'`input_boolean` de mémoire :
```yaml
input_boolean:
  clim_salon_etait_allumee:
    name: Clim Salon — était allumée
```

### Assigner les entités à la zone

Toutes les entités détectées automatiquement **doivent être assignées à leur zone** dans le registre HA, sinon `area_entities()` ne les trouvera pas :

**Paramètres → Appareils et services → Entités** → ouvrir l'entité → champ **Zone**.

Entités à assigner :
- Capteur de température intérieure
- `input_boolean.fenetre_<piece>`
- `binary_sensor.tendance_*` (si tendance activée)
- Entité `climate` (si gestion clim)

Le capteur extérieur et l'`input_boolean` de mémoire clim ne nécessitent pas d'assignation de zone.

---

## Installation

### Blueprint par pièce

1. Copier `blueprint_surveillance_fenetres.yaml` dans :
   ```
   config/blueprints/automation/surveillance_fenetres/
   ```
2. Recharger les blueprints : **Outils de développement → YAML → Blueprints d'automations**.
3. Créer une automation par pièce : **Paramètres → Automations → + Créer → Blueprint**.

### Blueprint global (contrôle maître)

1. Copier `blueprint_fenetres_global.yaml` dans le même dossier.
2. Créer **une seule instance** de ce blueprint avec la liste de tous les `input_boolean` fenêtres.
3. Créer l'`input_boolean` maître :
```yaml
input_boolean:
  fenetres_global:
    name: Fenêtres — contrôle global
    icon: mdi:window-open
```

---

## Paramètres

### Blueprint par pièce

**Obligatoires**

| Paramètre | Description |
|---|---|
| **Pièce / Zone** | Zone HA. Toutes les entités sont auto-détectées dans cette zone. |
| **Capteur température extérieure** | `sensor` avec `device_class: temperature`. Partagé entre instances. |
| **Seuil de fermeture** | Delta (°C) en dessous duquel l'alerte se déclenche. Défaut : `0.3 °C`. |
| **Fréquence de vérification** | Intervalle de réévaluation du delta. Défaut : `2 min`. |
| **Cible de notification** | Service `notify` ou groupe de notification. |

**Optionnels — Tendance**

| Paramètre | Description |
|---|---|
| **Utiliser la tendance** | Active la prise en compte du `binary_sensor` de tendance. |

Quand activée, l'alerte est supprimée si la température baisse déjà : la pièce se refroidit naturellement, fermer n'est pas urgent.

**Optionnels — Climatisation**

| Paramètre | Description |
|---|---|
| **Gérer la climatisation** | Active la coupure/restauration de la clim. |
| **Mémoire état clim** | `input_boolean` pour mémoriser l'état de la clim avant ouverture. Sélection manuelle. |
| **Délai de prise en compte de l'ouverture** | Durée d'ouverture continue avant réaction (coupure clim **et** notification). Défaut : `1 min`. `0` = réaction immédiate. |

### Blueprint global

| Paramètre | Description |
|---|---|
| **input_boolean global** | Le bouton maître (`input_boolean.fenetres_global`). |
| **Fenêtres à piloter** | Multi-sélection de tous les `input_boolean` fenêtres individuels. |

---

## Notification

Exemple de message reçu :

> **🪟 Ferme la fenêtre — Salon**
> Plus utile de laisser ouvert : delta 0.2°C (intérieur 26.3°C / extérieur 26.1°C)

---

## Carte dashboard (Mushroom chips)

La carte `carte_mushroom_fenetres.yaml` affiche :

**Chip global** (premier chip) :
- ⚫ Gris — tout fermé
- 🟠 Orange — partiellement ouvert (`N/5 ouvertes`)
- 🟢 Vert — tout ouvert
- Tap → toggle `input_boolean.fenetres_global` → propagé à toutes les fenêtres par le blueprint global

**Chips individuels** :
- Icône `mdi:window-open` / `mdi:window-closed-variant` selon l'état
- 🔵 Bleu — delta > seuil : utile d'aérer
- 🔴 Rouge — delta < 0 : refermer
- ⚫ Gris — delta neutre
- Chip atténué si fenêtre fermée
- Tap → toggle la fenêtre · Hold → détail de l'entité

---

## Changelog

### 1.2.0
- Le délai s'applique désormais **aussi à la notification** d'aération :
  aucune alerte tant que la fenêtre n'est pas ouverte depuis au moins le délai
  configuré (`for:` sur la condition d'état du cycle périodique).
- Paramètre renommé en **Délai de prise en compte de l'ouverture** (la clé
  interne `delai_coupure_clim` est conservée : les automations existantes
  gardent leur réglage, aucune reconfiguration nécessaire).

### 1.1.0
- Nouveau paramètre **Délai avant coupure de la clim** (`duration`, défaut 1 min) :
  la clim n'est coupée que si la fenêtre reste ouverte en continu pendant ce délai.
- La remise en marche de la clim à la fermeture reste immédiate.
- Version affichée dans le nom des deux blueprints (`… (v1.1.0)`), visible
  directement dans la liste des blueprints de Home Assistant pour suivre
  les mises à jour.

### 1.0.0
- Version initiale : alerte d'aération sur delta de température,
  gestion clim coupure/restauration, blueprint global, carte Mushroom.

---

## Compatibilité

- Home Assistant ≥ 2024.6
- [lovelace-mushroom](https://github.com/piitaya/lovelace-mushroom) pour la carte (HACS)
