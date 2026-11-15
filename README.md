# Lab GNS3 — Infrastructure réseau multi-sites sécurisée

![Topologie du lab](lab_topology.png)

## L'idée du projet

Je voulais construire, de A à Z, l'infrastructure réseau d'une PME marocaine avec un siège et deux agences, pour mettre en pratique — sur de l'IOS Cisco réel dans GNS3, pas un simulateur simplifié — tout ce que j'ai vu en CCNA et en stage : routage dynamique multi-site, haute disponibilité au cœur du réseau, et surtout une interconnexion WAN chiffrée entre les sites, comme ce qu'on ferait dans une vraie entreprise avec des agences distantes.

Le scénario : un siège à **Casablanca** (HQ), deux agences à **Rabat** et **Marrakech**, le tout relié à travers un **ISP simulé**. L'objectif n'était pas juste de faire "ça ping", mais de construire quelque chose de redondant, de sécurisé, et de le dépanner moi-même quand ça casse — ce qui, dans un lab virtualisé, arrive souvent pour des raisons qu'on ne voit jamais sur du vrai matériel.

## Étape 1 — Les routeurs Edge (R1-Edge et R2-Edge) : la porte d'entrée du siège

J'ai commencé par les deux routeurs de bordure du siège, **R1-Edge** et **R2-Edge**, connectés chacun à l'ISP d'un côté et aux switchs du siège de l'autre. L'idée dès le départ : ne pas avoir un seul point de sortie. Si R1 tombe, R2 doit prendre le relais tout seul.

Pour ça, j'ai mis en place un **DMVPN Phase 3** (mGRE + NHRP) avec les **deux routeurs en hub**, chacun avec son propre tunnel (`Tunnel0`, 172.168.0.1 pour R1, 172.168.0.2 pour R2). Par-dessus, j'ai chiffré tout le tunnel avec de l'**IPsec** (IKEv1, PSK, AES-256/SHA-256, DH group 14, ESP en mode transport) — donc même si quelqu'un capture le trafic entre les sites sur l'ISP, il ne voit que de l'ESP chiffré.

Côté routage, j'ai mis de l'**OSPF multi-area** : Area 0 sur le siège et sur l'overlay DMVPN, et chaque agence devient sa propre area (1 pour Rabat, 2 pour Marrakech). R1-Edge injecte la route par défaut en priorité (`default-information originate`), R2-Edge l'injecte aussi mais avec une métrique plus élevée — comme ça R1 reste le chemin préféré vers Internet, et R2 prend automatiquement le relais si R1 tombe.

Pour que les sites privés (HQ, Rabat, Marrakech) puissent sortir sur Internet, j'ai mis du **NAT/PAT overload** sur les deux routeurs Edge, avec une access-list qui couvre tous mes réseaux internes.

**Bug rencontré et corrigé ici** : à un moment, `show crypto isakmp sa` restait vide sur R2-Edge, et R1-Edge balançait des erreurs `%CRYPTO-6-IKMP_MODE_FAILURE`. En comparant la config crypto des deux routeurs, j'ai trouvé que j'avais tout simplement oublié la clé pre-share (`crypto isakmp key ... address ...`) et l'authentification sur R2-Edge — logique, les deux hubs doivent avoir exactement la même politique ISAKMP pour que l'IKE s'établisse. Une fois corrigé, les sessions sont passées à `QM_IDLE` / `ACTIVE` des deux côtés.

## Étape 2 — Les agences (R-RABAT et R-Marrakech) : les spokes DMVPN

Les deux routeurs d'agence sont de simples routeurs configurés en **router-on-a-stick** (sous-interfaces dot1Q, pas de L3 switch là-bas). Chacun a son propre tunnel DMVPN en tant que **spoke**, relié aux deux hubs du siège.

Le point intéressant du DMVPN Phase 3, c'est le **NHRP shortcut** : une fois que le trafic entre Rabat et Marrakech commence à passer, les deux spokes négocient directement un tunnel entre eux, sans repasser par le siège. J'ai voulu vérifier ça concrètement, pas juste faire confiance à la théorie.

**Adressage** : il fallait que chaque agence ait ses propres sous-réseaux (impossible de réutiliser les VLAN/IP du siège puisque les sites sont reliés en L3, pas dans le même domaine de diffusion). Rabat a eu 192.168.110.0/24 (USERS) et 192.168.130.0/24 (VOICE), Marrakech 192.168.210.0/24 et 192.168.230.0/24.

**Bug rencontré et corrigé ici** : sur R-RABAT, la route par défaut apprise par OSPF pointait à travers le tunnel lui-même (`via Tunnel0`), alors que la vraie destination (l'adresse NBMA du hub, 10.0.0.2) devait être jointe directement par l'interface physique. Ce routage récursif rendait les sessions NHRP instables. Corrigé avec une route statique directe vers l'ISP, de distance administrative plus faible que la route OSPF, pour forcer le bon chemin.

## Étape 3 — Les switchs L3 du siège (MLS-1 et MLS-2) : la haute disponibilité

Au cœur du siège, deux switchs multicouches, **MLS-1** et **MLS-2**, reliés par un **EtherChannel LACP** sur 3 liens physiques (mode `active` des deux côtés). Sur ces deux switchs tournent les VLANs du siège : 10 (USERS), 20 (SERVERS), 30 (VOICE), 99 (MANAGEMENT).

Pour la redondance de passerelle, j'ai mis du **HSRP** par VLAN, mais pas tout sur le même switch — j'ai réparti la charge : MLS-1 est actif pour les VLANs 10 et 20, MLS-2 est actif pour 30 et 99. Chaque switch sert donc du trafic réel en fonctionnement normal, pas juste en secours.

J'ai aussi aligné le **Rapid-PVST+** là-dessus : le root bridge de chaque VLAN est le même switch que l'actif HSRP de ce VLAN. Sans ça, le trafic peut faire un aller-retour inutile entre les deux switchs avant de sortir (hairpin L2/L3) — un détail qu'on ne voit dans aucun cours mais qui change vraiment le comportement du réseau.

Ces deux switchs servent aussi de **serveurs DHCP** pour tous les sites (siège, Rabat, Marrakech), en split-scope entre MLS-1 et MLS-2.

**Bug rencontré et corrigé ici** : le relais DHCP (`ip helper-address`) pour les agences distantes. Ma première tentative pointait vers l'IP virtuelle HSRP d'un VLAN — ce qui ne marche pas, parce que le DHCP relay a besoin de joindre l'IP réelle et stable du serveur DHCP, pas une IP qui peut changer de switch actif selon l'état HSRP. J'ai donc fait pointer les `ip helper-address` vers les **Loopback0** de MLS-1 et MLS-2 (10.255.0.3 et 10.255.0.4), qui restent toujours joignables quel que soit l'état HSRP.

## Étape 4 — Les switchs L2 d'accès (SW-1 et les autres)

En bout de chaîne, des switchs L2 purs (SW-1 au siège, un par agence) qui distribuent les VLANs vers les ports d'accès, avec un double uplink en trunk 802.1Q vers MLS-1 et MLS-2 pour le siège — encore une fois, pas de single point of failure.

## Les tests — est-ce que tout ça marche vraiment ?

Une fois que toutes les briques étaient montées, j'ai voulu vérifier concrètement, pas juste supposer que la théorie s'applique.

**OSPF** : `show ip ospf neighbor` sur R1-Edge confirme une adjacence FULL avec R2-Edge (BDR) et les deux spokes (DROTHER) sur le tunnel, plus les deux switchs du siège sur les liens physiques.

**DMVPN dual-hub** : `show dmvpn detail` sur R1-Edge et sur R2-Edge montre chacun les trois pairs (l'autre hub + les deux spokes), avec les tunnels spoke marqués `UP`.

**IPsec sur les deux hubs** : `show crypto isakmp sa` et `show crypto ipsec sa` confirment des sessions `QM_IDLE` / `UP-ACTIVE` avec `esp-256-aes esp-sha256-hmac`, zéro paquet perdu, sur R1-Edge comme sur R2-Edge après correction du bug du pre-share key.

**Spoke-to-spoke — la preuve la plus importante du projet** : j'ai voulu confirmer que le trafic Rabat ↔ Marrakech passe vraiment en direct, et pas via le siège.
- Sur R-RABAT, `show crypto isakmp sa` montre une session IKE active directement avec 10.0.0.14 (Marrakech), en plus de celle avec le hub.
- J'ai capturé le trafic avec Wireshark sur le lien ISP ↔ R-RABAT pendant un test PC Rabat → PC Marrakech. Résultat : une série de trames **ESP échangées directement entre 10.0.0.10 et 10.0.0.14**, jamais via 10.0.0.2 ou 10.0.0.6 (les hubs). C'est la preuve réseau, pas juste un `show` : le tunnel spoke-to-spoke est réellement emprunté et chiffré.
- Pour confirmer côté applicatif, j'ai fait un `traceroute` depuis PC3 (Rabat) vers un hôte à Marrakech (192.168.230.11) : le deuxième saut arrive directement sur `172.168.0.4` (le tunnel de R-Marrakech), en un seul saut, sans passer par un hub. Le dernier saut renvoie un "destination port unreachable" — ce qui n'est pas une erreur de routage, c'est la réponse normale d'un hôte qui a bien reçu les sondes UDP du traceroute mais n'écoute pas sur ce port précis. Le paquet est donc bien arrivé jusqu'au bout.

**NAT/PAT** : `show ip nat translations` et un `debug ip nat` en direct confirment la traduction d'un poste du LAN (192.168.10.x) vers l'IP publique du routeur Edge quand il sort vers Internet (8.8.8.8), et uniquement dans ce cas — le trafic entre sites privés (HQ ↔ Rabat ↔ Marrakech) continue de passer par le routage normal, sans NAT, puisque les deux bouts sont des réseaux privés routés.

**HSRP / EtherChannel / STP** : `show standby brief` confirme la répartition de charge entre MLS-1 et MLS-2, `show etherchannel summary` confirme le Port-channel1 en LACP actif sur les 3 liens, et `show spanning-tree root` confirme que le root STP suit bien l'actif HSRP VLAN par VLAN.

**DHCP relay multi-site** : `show ip dhcp binding` sur MLS-1 montre des baux actifs pour des postes de Rabat et de Marrakech, servis depuis les pools centralisés du siège — la preuve que le relais fonctionne de bout en bout.

## Résultat

Le lab répond à l'objectif de départ : un siège redondant avec deux hubs DMVPN et deux switchs L3 en haute disponibilité, deux agences connectées en DMVPN Phase 3 avec tunnel spoke-to-spoke chiffré confirmé par capture, et une sortie Internet mutualisée via NAT. Chaque bug rencontré pendant la construction (mismatch ISAKMP entre les deux hubs, routage récursif sur un spoke, mauvaise cible de DHCP relay) a été diagnostiqué par la même méthode : comparer les `show` entre les équipements censés être symétriques, puis vérifier par capture ou par test réel après correction — plutôt que de supposer que la config est bonne parce qu'elle "a l'air" correcte.

## Où mettre les captures d'écran

Place tes screenshots dans un dossier `screenshots/` à la racine du repo, avec un sous-dossier par équipement (`screenshots/R1-Edge/`, `screenshots/R2-Edge/`, `screenshots/R-RABAT/`, `screenshots/R-Marrakech/`, `screenshots/MLS-1/`, `screenshots/MLS-2/`, `screenshots/SW-1/`), puis référence-les dans ce README avec `![description](screenshots/R1-Edge/nom-image.png)` à l'endroit du paragraphe correspondant. Les plus utiles à montrer : `show dmvpn detail`, la capture Wireshark ESP spoke-to-spoke, `show crypto isakmp sa` avant/après le fix du bug R2-Edge, `show standby brief`, et le traceroute PC3 → Marrakech.

## Schéma complet

Le fichier [`lab_topology.svg`](lab_topology.svg) contient le plan d'adressage détaillé (IP, VLANs, liens point-à-point), le schéma de l'overlay DMVPN et la liste des protocoles mis en œuvre.

## Compétences mobilisées

`OSPF multi-area` `EtherChannel/LACP` `Rapid-PVST+` `HSRP` `VLAN/Trunking 802.1Q` `DHCP/DHCP Relay` `NAT/PAT` `DMVPN (mGRE/NHRP)` `IPsec/IKE` `Wireshark` `GNS3` `Troubleshooting CLI avancé`

---
*Lab personnel réalisé en complément du cycle CCNA (Introduction to Networks · Switching, Routing & Wireless Essentials · Enterprise Networking, Security & Automation) et du stage de fin d'études sur architecture DMVPN.*
