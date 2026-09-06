# Frient Keypad + Alarmo

Blueprints Home Assistant pour synchroniser des claviers Frient (Zigbee2MQTT)
avec un panneau d'alarme (Alarmo, `manual`, ...).

| Fichier | Usage |
| --- | --- |
| `blueprints/frient_keypad_dual_with_alarmo.yaml` | **2 claviers** sur un seul panneau. C'est celui à utiliser. |
| `blueprints/frient_keypad_with_alarmo.yaml` | Version amont, **1 clavier**. Copie conforme de [Bygood91/frient_keypad_alarmo](https://github.com/Bygood91/frient_keypad_alarmo), gardée comme référence. |

## Configuration

Neuf champs, dont trois obligatoires.

| Section | Champ | Détail |
| --- | --- | --- |
| Panneau | Panneau d'alarme | l'entité `alarm_control_panel`, sélectionnée une fois |
| Claviers | Clavier 1, Clavier 2 | le nom dans *Zigbee2MQTT → Devices* (la casse compte) |
| Code | Codes acceptés | obligatoire, une entrée par code PIN ou tag RFID |
| Boutons | Absent, Maison, Nuit, Désarmer, SOS | l'action déclenchée sur le panneau |

Les topics MQTT sont déduits du nom : `zigbee2mqtt/<nom>` pour l'état,
`zigbee2mqtt/<nom>/set` pour les commandes.

**Le code est défini une seule fois**, dans la section Code, et rien d'autre
n'est à renseigner ailleurs :

- **toute action exige un code valide**, y compris le bouton SOS ;
- un code inconnu, ou une action lancée sans code, renvoie `invalid_code` au
  clavier utilisé (et à lui seul) ;
- pour exempter le SOS, une ligne à changer dans le blueprint — la variable
  `code_ok` porte le commentaire qui l'explique. **À savoir** : si ton clavier
  n'envoie aucun code avec l'appui SOS (cas courant, c'est un bouton de
  panique), l'exiger revient à désactiver ce bouton.

**Prérequis côté Alarmo** : les codes ci-dessus sont la seule autorité, le
panneau est piloté **sans code transmis**. Dans Alarmo, *Général → Armement*,
désactiver « Code requis pour l'armement » et « Code requis pour le
désarmement » — sinon l'appel de service échoue.
Conséquence à connaître : Alarmo devient désarmable sans code par les autres
chemins (carte du tableau de bord, autre automatisation, API REST).

### Mapping des boutons

Chaque bouton du clavier se voit attribuer une action parmi : armer absent,
armer maison, armer nuit, armer vacances, désarmer, déclencher l'alarme, ou ne
rien faire. Les valeurs par défaut correspondent au marquage des touches.

En sens inverse, l'état du panneau est affiché sur les deux claviers sans rien à
configurer :

| État du panneau | Affichage clavier |
| --- | --- |
| `arming` / `pending` | `exit_delay` / `entry_delay` |
| `armed_away`, `armed_vacation` | `arm_all_zones` |
| `armed_home` / `armed_night` | `arm_day_zones` / `arm_night_zones` |
| `disarmed` / `triggered` | `disarm` / `in_alarm` |

Les claviers sont aussi resynchronisés au démarrage de Home Assistant et à
chaque exécution manuelle de l'automatisation — utile après un réappairage Z2M.

## Installation

1. Copier `blueprints/frient_keypad_dual_with_alarmo.yaml` dans
   `config/blueprints/automation/<un_sous_dossier>/`, puis *Outils de
   développement → YAML → Recharger les automatisations*.
   Le fichier ne doit pas être dans `automations.yaml` ni tiré par un
   `!include` : c'est un modèle, pas une automatisation.
2. *Paramètres → Automatisations → Blueprints*, créer une automatisation à
   partir du blueprint et remplir les neuf champs.
3. Dans Alarmo, désactiver les deux options « code requis » (voir plus haut).

> Si un clavier est renommé dans Zigbee2MQTT, mettre à jour l'automatisation :
> le nom est ce qui construit les topics.
> Le préfixe `zigbee2mqtt` est en dur (c'est le `base_topic` par défaut de Z2M) ;
> si tu l'as changé, deux lignes sont à adapter dans le blueprint,
> `tv_base_topic` sous `trigger_variables` et `base_topic` sous `variables`.

### Trouver l'identifiant d'un tag RFID

*Paramètres → Appareils et services → MQTT → Écouter un sujet*, s'abonner à
`zigbee2mqtt/Keypad1`, puis passer le badge. Copier la valeur de `action_code`
(par ex. `+ACF5678B`) dans la liste des codes.

## Différences avec la version amont

**Multi-clavier**
- Importer deux fois le blueprint amont exécute les actions **en double** à
  chaque changement d'état du panneau. Ici une seule automatisation pilote les
  deux claviers : un message publié par clavier, actions déclenchées une fois.
- Un déclencheur MQTT par clavier, les deux peuvent armer et désarmer.
- Le retour `invalid_code` ne part que vers le clavier utilisé.

**Bugs corrigés**
- Un code accepté n'était **jamais confirmé au clavier**. Le clavier envoie un
  numéro de transaction avec chaque demande d'armement et attend qu'on le lui
  renvoie ; sans cette réponse il considère le code comme refusé, quoi que
  fasse le panneau ensuite. La version amont ne renvoyait la transaction que
  sur code invalide. L'accusé de réception est maintenant publié sur le clavier
  utilisé avant même l'appel au panneau.
- `armed_home` n'avait **aucun déclencheur** : la branche correspondante était
  morte, passer en *Armed Home* ne mettait jamais le clavier à jour.
- Les codes n'étaient pas « trimés » : `1234; 5678` produisait `" 5678"`, qui ne
  correspondait à aucune saisie.
- Les codes spéciaux étaient comparés en sous-chaîne : un code `1` déclenchait
  son action pour tout code contenant un `1`.
- Le retour `invalid_code` était réservé aux codes numériques ; un tag RFID
  inconnu ne provoquait aucun retour.

**Simplifications**
- Le nom du clavier remplace ses deux topics MQTT.
- Le code est défini à un seul endroit, plus de code de panneau séparé.
- Les huit branches d'état quasi identiques deviennent une table
  état → mode, et les quatre appels de service une table bouton → action.

## Ce qui n'est pas géré

Le blueprint amont proposait des actions Home Assistant libres à chaque
changement d'état (notifications, sirène...) et trois codes PIN spéciaux
déclenchant des scripts. Ils ont été retirés au profit du mapping bouton →
action de panneau. Pour une notification, une automatisation séparée
déclenchée sur l'entité `alarm_control_panel` fait le travail et se relit plus
facilement.

## Crédits

Blueprint d'origine : [Bygood91/frient_keypad_alarmo](https://github.com/Bygood91/frient_keypad_alarmo).
