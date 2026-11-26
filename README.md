# Conception et mise en œuvre d'une infrastructure réseau pour un garage mécanique de luxe

Projet réalisé dans le cadre du cours **OSPF** (*Open Shortest Path First*, protocole de routage à état de liens), filière **ASSRI** (Administration et Sécurité des Systèmes et Réseaux Informatiques) / **RIT** (Réseaux et Informatique de Télécommunication), à l'**UATM** (Université Africaine de Technologie et de Management, GASA Formation), année scolaire 2026-2027.

## Présentation

Un garage mécanique de luxe compte **100 employés** répartis en deux départements :

| Département | Effectif | VLAN | Réseau | Masque | Passerelle | Hôtes utilisables |
|---|---|---|---|---|---|---|
| Administration | 20 | 10 | 192.168.10.0/27 | 255.255.255.224 | 192.168.10.1 | 192.168.10.1 – 192.168.10.30 |
| Opérations | 80 | 20 | 192.168.20.0/25 | 255.255.255.128 | 192.168.20.1 | 192.168.20.1 – 192.168.20.126 |

**Exigence principale :** isoler les deux départements tout en gardant la communication à l'intérieur de chaque VLAN, avec une infrastructure redondante et évolutive.

## Objectifs

- Segmenter le réseau en deux **VLAN** (*Virtual Local Area Network*, réseau local virtuel)
- Définir un plan d'adressage IP (*Internet Protocol*, protocole Internet) adapté aux besoins actuels et futurs
- Centraliser la gestion des VLAN avec **VTP** (*VLAN Trunking Protocol*)
- Configurer des liaisons **trunk** (liaisons transportant plusieurs VLAN)
- Assurer le routage inter-VLAN avec **Router-on-a-Stick**
- Empêcher les boucles de niveau 2 avec **STP** (*Spanning Tree Protocol*)
- Agréger les liens avec **EtherChannel / LACP** (*Link Aggregation Control Protocol*)
- Isoler les départements avec des **ACL** (*Access Control Lists*, listes de contrôle d'accès)
- Valider l'ensemble par des commandes de vérification et des tests de connectivité

## Architecture

Équipements : 1 routeur (**R1**), 1 switch central (**SW-CENTRAL**) et 3 switches d'accès (**SW-ADMIN**, **SW-OP1**, **SW-OP2**).

```
                 Internet
                    |
                [ Routeur R1 ]
                    | G0/0 <-> Fa0/1
              [ SW-CENTRAL ]  (serveur VTP, Root Bridge STP)
              /      |      \
         Fa0/2     Fa0/3    Fa0/4
            |        |        |
      [SW-ADMIN]=[SW-OP1]=[SW-OP2]
       VLAN 10    VLAN 20   VLAN 20
```

| Liaison | Type |
|---|---|
| R1 G0/0 ↔ SW-CENTRAL Fa0/1 | Trunk (Router-on-a-Stick) |
| SW-CENTRAL Fa0/2 ↔ SW-ADMIN Fa0/4 | Trunk |
| SW-CENTRAL Fa0/3 ↔ SW-OP1 Fa0/4 | Trunk |
| SW-CENTRAL Fa0/4 ↔ SW-OP2 Fa0/4 | Trunk |
| SW-ADMIN Fa0/5-6 ↔ SW-OP1 Fa0/5-6 | Port-Channel 1 (LACP) |
| SW-OP1 Fa0/7-8 ↔ SW-OP2 Fa0/5-6 | Port-Channel 2 (LACP) |

## Technologies et choix de configuration

- **VLAN** : VLAN 10 (ADMINISTRATION) et VLAN 20 (OPERATIONS), créés sur SW-CENTRAL
- **VTP** : domaine `GARAGE`, SW-CENTRAL en mode serveur, les trois autres switches en mode client
- **Ports d'accès** : Fa0/1 à Fa0/3 de chaque switch d'accès
- **Router-on-a-Stick** : sous-interfaces `G0/0.10` et `G0/0.20` en encapsulation 802.1Q (*dot1Q*)
- **STP** : SW-CENTRAL est désigné **Root Bridge** pour les VLAN 10 et 20
- **EtherChannel** : deux Port-Channels en mode LACP `active`
- **ACL étendues** : `BLOQUE-ADMIN-OPS` et `BLOQUE-OPS-ADMIN` appliquées en entrée sur les sous-interfaces du routeur

## Résultats des tests

| Test | Commande | Résultat |
|---|---|---|
| VLAN | `show vlan brief` | VLAN 10 et 20 présents, ports correctement affectés |
| VTP | `show vtp status` | Domaine GARAGE, rôles serveur/client vérifiés |
| Trunks | `show interfaces trunk` | Liaisons trunk vérifiées |
| STP | `show spanning-tree vlan 10 / 20` | SW-CENTRAL est le Root Bridge |
| EtherChannel | `show etherchannel summary` | Po1 et Po2 opérationnels avec LACP |
| Intra-Administration | `ping 192.168.10.3` | Réussi |
| Intra-Opérations | `ping 192.168.20.3` | Réussi |
| Vers les passerelles | `ping 192.168.10.1` / `192.168.20.1` | Réussi |
| Administration → Opérations | `ping 192.168.20.2` | Bloqué (isolement respecté) |
| Opérations → Administration | `ping 192.168.10.2` | Bloqué (isolement respecté) |

**Redondance :** la coupure d'un lien membre d'un Port-Channel (`shutdown` sur Fa0/5 de SW-ADMIN) laisse le service actif grâce au second lien Fa0/6.

## Contenu du dépôt

```
.
├── README.md
├── rapport/
│   └── rapport.pdf        # Rapport complet (configurations, captures, explications, glossaire)
└── simulation/
    └── projet.pkt         # Topologie Cisco Packet Tracer
```

> Adapte les noms de fichiers ci-dessus à ceux réellement présents dans ton dépôt.

## Utilisation

1. Installer **Cisco Packet Tracer** (simulateur réseau de Cisco).
2. Ouvrir le fichier `simulation/projet.pkt`.
3. Accéder à la console (onglet **CLI**, *Command Line Interface*) de chaque équipement pour consulter les configurations.
4. Rejouer les tests de connectivité depuis les postes (onglet **Desktop** → **Command Prompt**).

## Glossaire

| Sigle | Signification |
|---|---|
| VLAN | *Virtual Local Area Network* : segmentation logique du réseau |
| VTP | *VLAN Trunking Protocol* : diffusion des VLAN entre switches d'un même domaine |
| STP | *Spanning Tree Protocol* : prévention des boucles de niveau 2 |
| LACP | *Link Aggregation Control Protocol* : négociation des agrégations de liens |
| ACL | *Access Control List* : règles de filtrage du trafic |
| OSPF | *Open Shortest Path First* : protocole de routage à état de liens |
| ASSRI | Administration et Sécurité des Systèmes et Réseaux Informatiques |
| RIT | Réseaux et Informatique de Télécommunication |
| UATM | Université Africaine de Technologie et de Management |
| CLI | *Command Line Interface* : interface en ligne de commande |
| NVRAM | *Non-Volatile Random Access Memory* : mémoire non volatile qui conserve la startup-config |

## Équipe

**Groupe 3**
- DADJO Hélène
- EDJLOSSE Caleb
- AGBOKAN Mardoché

**Encadrant :** M. Joseph ELEGBEDE

## Remarque

Les mots de passe présents dans le projet (`cisco`, `class`) sont des valeurs pédagogiques utilisées uniquement dans le cadre de la simulation. Ne jamais les réutiliser sur un réseau réel.
