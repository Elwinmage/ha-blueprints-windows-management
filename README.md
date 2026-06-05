# 🪟 Surveillance fenêtre — aération par pièce

Blueprint Home Assistant qui surveille une fenêtre ouverte et alerte quand il n'est plus utile de laisser la pièce s'aérer. Optionnellement, coupe et rallume la climatisation en fonction de l'état de la fenêtre.

Une instance du blueprint est créée **par pièce**.

---

## Fonctionnement

Le blueprint évalue toutes les minutes le delta de température entre l'intérieur de la pièce et l'extérieur :

```
delta = T_intérieur − T_extérieur
```

Quand la fenêtre est ouverte et que le delta passe sous le seuil configuré, une notification est envoyée pour indiquer qu'il ne sert plus à rien de laisser la fenêtre ouverte (l'extérieur n'est plus plus frais que l'intérieur).

Si le capteur de tendance est activé, l'alerte est affinée : elle ne se déclenche que si la température de la pièce est en hausse (tendance `on`), évitant les faux positifs quand la pièce se refroidit naturellement.

### Logique complète

```
Toutes les minutes
  └─ Fenêtre ouverte ?
       └─ delta < seuil ?
            └─ (si tendance activée) température montante ?
                 └─ → Notification "ferme la fenêtre"

Fenêtre ouverte (événement)
  └─ Gestion clim activée ?
       └─ Clim en marche ?
            ├─ oui → mémoriser + couper la clim
            └─ non → réinitialiser la mémoire

Fenêtre fermée (événement)
  └─ Gestion clim activée ?
       └─ Clim était en marche ?
            └─ oui → rallumer la clim + effacer la mémoire
```

---

## Prérequis

### Entités à créer par pièce

Avant d'instancier le blueprint, créer dans `configuration.yaml` (ou via l'UI **Paramètres → Entrées**) :

**Obligatoire :**
```yaml
input_boolean:
  fenetre_salon:
    name: Fenêtre Salon
    icon: mdi:window-closed-variant
```

**Si gestion clim activée :**
```yaml
input_boolean:
  clim_salon_etait_allumee:
    name: Clim Salon — était allumée
```

### Assigner les entités à la zone

Les sélecteurs du blueprint filtrent automatiquement les entités sur la zone choisie. Pour que cela fonctionne, chaque entité doit être assignée à sa zone dans le registre :

**Paramètres → Appareils et services → Entités** → ouvrir l'entité → champ **Zone**.

Entités à assigner à leur zone respective :
- Le capteur de température intérieure
- Le capteur de tendance (si utilisé)
- L'`input_boolean` de la fenêtre
- L'entité `climate` (si gestion clim)
- L'`input_boolean` de mémoire clim (si gestion clim)

Le capteur extérieur n'est pas filtré sur une zone (il est partagé entre toutes les instances).

### Carte Mushroom (dashboard)

Voir le fichier `carte_mushroom_fenetres.yaml` pour les chips cliquables associées. Nécessite [lovelace-mushroom](https://github.com/piitaya/lovelace-mushroom) (HACS).

---

## Installation

1. Copier `blueprint_surveillance_fenetres.yaml` dans :
   ```
   config/blueprints/automation/surveillance_fenetres/
   ```

2. Recharger les blueprints : **Outils de développement → YAML → Blueprints d'automations**.

3. Créer une automation par pièce : **Paramètres → Automations → + Créer une automation → Blueprint**.

---

## Paramètres

### Obligatoires

| Paramètre | Description |
|---|---|
| **Pièce** | Zone HA à surveiller. Filtre tous les sélecteurs d'entités. |
| **Capteur température intérieure** | `sensor` avec `device_class: temperature`, assigné à la zone. |
| **Capteur température extérieure** | `sensor` avec `device_class: temperature`, non zonifié. Partagé entre instances. |
| **État de la fenêtre** | `input_boolean` assigné à la zone. Bascule manuellement via la carte dashboard. |
| **Seuil de fermeture** | Delta (°C) en dessous duquel l'alerte se déclenche. Défaut : `0.3 °C`. |
| **Durée avant alerte** | Délai de vérification (toutes les N minutes). Défaut : `2 min`. |
| **Cible de notification** | Service `notify` ou groupe de notification. |

### Optionnels — Tendance

| Paramètre | Description |
|---|---|
| **Utiliser le capteur de tendance** | Active la prise en compte de la tendance thermique. Défaut : `false`. |
| **Capteur de tendance** | `binary_sensor` assigné à la zone. État `on` = température en hausse. |

Quand la tendance est activée, l'alerte est supprimée si la température est déjà en baisse : la pièce se refroidit d'elle-même, il n'est pas urgent de fermer.

### Optionnels — Climatisation

| Paramètre | Description |
|---|---|
| **Gérer la climatisation** | Active la coupure/restauration automatique de la clim. Défaut : `false`. |
| **Entité climatisation** | Entité `climate` assignée à la zone. |
| **Mémoire état clim** | `input_boolean` assigné à la zone, dédié à cette pièce. Stocke si la clim tournait avant l'ouverture de la fenêtre. |

---

## Notification

Exemple de message reçu :

> **🪟 Ferme la fenêtre — Salon**
> Plus utile de laisser ouvert : delta 0.2°C (intérieur 26.3°C / extérieur 26.1°C)

---

## Carte dashboard (Mushroom chips)

La carte associée (`carte_mushroom_fenetres.yaml`) affiche un chip par pièce :

- **Icône** : `mdi:window-open` ou `mdi:window-closed-variant` selon l'état de l'`input_boolean`
- **Couleur** :
  - 🔵 Bleu — delta > seuil : utile d'aérer
  - 🔴 Rouge — delta < 0 : la pièce est plus froide que dehors, refermer
  - ⚫ Gris — delta neutre
  - Atténué — fenêtre fermée
- **Tap** : bascule l'état ouvert/fermé de la fenêtre
- **Hold** : ouvre la fiche détail de l'entité

---

## Compatibilité

- Home Assistant ≥ 2024.6
- [lovelace-mushroom](https://github.com/piitaya/lovelace-mushroom) pour la carte (HACS)
