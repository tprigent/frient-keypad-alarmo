# Frient Keypad + Alarmo

Blueprints Home Assistant pour synchroniser un ou plusieurs claviers Frient
(Zigbee2MQTT) avec un panneau d'alarme (Alarmo, `manual`, ...).

| Fichier | Usage |
| --- | --- |
| `blueprints/frient_keypad_with_alarmo.yaml` | Version amont, **1 clavier**. Copie conforme de [Bygood91/frient_keypad_alarmo](https://github.com/Bygood91/frient_keypad_alarmo), conservée comme référence. |
| `blueprints/frient_keypad_dual_with_alarmo.yaml` | Version adaptée, **2 claviers** sur la même instance Alarmo. |

## Pourquoi une version dédiée à 2 claviers

Importer deux fois le blueprint amont (une automatisation par clavier)
« fonctionne » à moitié : chaque changement d'état d'Alarmo déclenche **les deux**
automatisations, donc toutes les actions utilisateur (`Action Armed Away`,
notifications, sirène...) sont exécutées **en double**. Les codes PIN, le code du
panneau et les actions doivent aussi être saisis deux fois.

La version `dual` n'utilise **qu'une seule automatisation** : les deux claviers
sont mis à jour, mais les actions ne partent qu'une fois.

## Ce qui change par rapport à la version amont

**Multi-clavier**
- 2 jeux de topics MQTT (state + set) en entrée ; les publications d'état se font
  en boucle sur les deux topics `/set` (via `repeat.for_each`), donc un seul
  message par clavier.
- Un déclencheur MQTT par clavier : les deux peuvent armer / désarmer.
- Le retour `invalid_code` est renvoyé **uniquement au clavier utilisé**
  (routage via `trigger.id`), pas aux deux.

**Corrections de bugs de la version amont**
- `armed_home` n'avait **aucun déclencheur** : la branche `panel_armed_home`
  était donc morte. Résultat, passer en *Armed Home* ne mettait jamais le clavier
  à jour (`arm_day_zones`) et `Action Armed Home` ne s'exécutait jamais. Corrigé.
- Les codes PIN n'étaient pas « trimés » : `1234; 5678` produisait `" 5678"`,
  qui ne correspondait à aucun code saisi. Les espaces sont maintenant ignorés.
- Les codes PIN spéciaux étaient comparés avec `in` (sous-chaîne) : un code
  spécial `1` déclenchait son action pour *tout* code contenant un `1`.
  La comparaison est maintenant exacte (et accepte plusieurs codes séparés par `;`).

**Ajouts**
- **Saisie des codes une entrée par ligne** (sélecteurs `text` multiples) au lieu
  d'une chaîne à séparer par des `;` : plus de problème d'espaces ou de
  séparateur oublié. Les codes PIN et les tags RFID ont chacun leur liste, et les
  codes spéciaux aussi. Une ancienne configuration au format `a; b` reste
  acceptée telle quelle.
- Retour `invalid_code` sur le clavier également pour un **tag RFID inconnu** :
  la version amont ne répondait qu'aux codes numériques (`| int(-1) != -1`), un
  badge non autorisé ne provoquait donc aucun retour. Le rejet est maintenant
  déclenché par l'action d'armement/désarmement, ce qui couvre PIN et RFID sans
  répondre aux messages d'état sans code.
- Resynchronisation des claviers au démarrage de Home Assistant, et à chaque
  exécution manuelle de l'automatisation (utile quand un clavier a été
  débranché / réappairé et affiche un état obsolète). Les actions utilisateur ne
  sont pas rejouées dans ce cas.
- Prise en charge de `armed_vacation` (état Alarmo) avec son action dédiée.
- Les 8 branches quasi identiques de la version amont sont remplacées par une
  table état → mode clavier, ce qui évite ce type d'oubli à l'avenir.

Le format des messages MQTT publiés est inchangé
(`{"arm_mode": {"mode": "..."}}`), les noms des entrées d'action sont conservés.

## Installation

1. Copier `blueprints/frient_keypad_dual_with_alarmo.yaml` dans
   `config/blueprints/automation/<votre_dossier>/` puis recharger les
   automatisations (ou importer l'URL du fichier via **Paramètres → Automatisations
   → Blueprints → Importer**).
2. Créer une automatisation à partir du blueprint et renseigner :
   - Clavier 1 : `zigbee2mqtt/Keypad1` et `zigbee2mqtt/Keypad1/set`
   - Clavier 2 : `zigbee2mqtt/Keypad2` et `zigbee2mqtt/Keypad2/set`
   - les codes PIN et les tags RFID (une entrée par code, bouton « Ajouter »),
     communs aux deux claviers
   - l'entité `alarm_control_panel` et son code

> Le nom du clavier dans les topics est son *friendly name* Zigbee2MQTT.
> Si un clavier est renommé dans Z2M, mettre à jour l'automatisation.

### Trouver l'identifiant d'un tag RFID

Le tag est identifié par l'`action_code` publié par Zigbee2MQTT au moment où le
badge est présenté. Pour le relever : **Paramètres → Appareils et services →
MQTT → Écouter un sujet**, s'abonner à `zigbee2mqtt/Keypad1`, puis passer le
badge. Copier la valeur de `action_code` (par ex. `+ACF5678B`) dans la liste des
tags RFID.

Home Assistant 2024.10 minimum (syntaxe `triggers:` / `actions:`).

## Crédits

Blueprint d'origine : [Bygood91/frient_keypad_alarmo](https://github.com/Bygood91/frient_keypad_alarmo).
