# Lab GNS3 — Infrastructure réseau multi-sites sécurisée

## 🗺️ Topologie du lab

<img width="762" height="409" alt="labe" src="https://github.com/user-attachments/assets/b9545bbd-f90c-4fc3-bcf0-c142ff8bc69f" />

## L'idée du projet

Je voulais construire, de A à Z, l'infrastructure réseau d'une PME marocaine avec un siège et deux agences, pour mettre en pratique — sur de l'IOS Cisco réel dans GNS3, pas un simulateur simplifié — tout ce que j'ai vu en CCNA et en stage : routage dynamique multi-site, haute disponibilité au cœur du réseau, et surtout une interconnexion WAN chiffrée entre les sites, comme ce qu'on ferait dans une vraie entreprise avec des agences distantes.

Le scénario : un siège à **Casablanca** (HQ), deux agences à **Rabat** et **Marrakech**, le tout relié à travers un **ISP simulé**. L'objectif n'était pas juste de faire "ça ping", mais de construire quelque chose de redondant, de sécurisé, et de le dépanner moi-même quand ça casse — ce qui, dans un lab virtualisé, arrive souvent pour des raisons qu'on ne voit jamais sur du vrai matériel.

## Topologie
 
| Site | Rôle | Équipements |
|---|---|---|
| **ISP** | Cœur Internet simulé | 1 routeur (pas de route vers les réseaux privés) |
| **HQ — Casablanca** | Siège | 2 routeurs Edge (**R1-Edge**, **R2-Edge**), 2 switchs L3 (**MLS-1**, **MLS-2**), switchs L2 d'accès (**SW-1**, SW-2/3/4) |
| **Branch — Rabat** | Agence 1 | 1 routeur router-on-a-stick (**R-RABAT**), 1 switch L2 (**SW-RABAT**) |
| **Branch — Marrakech** | Agence 2 | 1 routeur router-on-a-stick (**R-Marrakech**), 1 switch L2 (**SW-5**) |
 
## Plan d'adressage (extrait)
 
| Site | VLAN / Rôle | Subnet |
|---|---|---|
| HQ | VLAN 10 — USERS | 192.168.10.0/24 |
| HQ | VLAN 20 — SERVERS | 192.168.20.0/24 |
| HQ | VLAN 30 — VOICE | 192.168.30.0/24 |
| HQ | VLAN 99 — MANAGEMENT | 192.168.99.0/24 |
| Rabat | USERS | 192.168.110.0/24 |
| Rabat | VOICE | 192.168.130.0/24 |
| Rabat | MANAGEMENT | 192.168.90.0/24 |
| Marrakech | USERS | 192.168.210.0/24 |
| Marrakech | VOICE | 192.168.230.0/24 |
| Overlay DMVPN | Tunnel0 | 172.168.0.0/24 |
| Loopbacks (Management/BGP-free stable IDs) | R1/R2/MLS-1/MLS-2/R-RABAT/R-Marrakech | 10.255.0.1 – 10.255.0.7 |


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

<img width="620" height="260" alt="sh cdp ne" src="https://github.com/user-attachments/assets/40b3435f-83f2-401f-8722-ee66eaf2e517" />
<img width="621" height="233" alt="sh cdp ne" src="https://github.com/user-attachments/assets/ee838720-a0e5-44ed-994e-2236c30f2f99" />

Au cœur du siège, deux switchs multicouches, **MLS-1** et **MLS-2**, reliés par un **EtherChannel LACP** sur 3 liens physiques (mode `active` des deux côtés). Sur ces deux switchs tournent les VLANs du siège : 10 (USERS), 20 (SERVERS), 30 (VOICE), 99 (MANAGEMENT).

<img width="626" height="279" alt="sh vlan b" src="https://github.com/user-attachments/assets/293c8897-957a-4f3c-a7db-06179a37a904" />
<img width="644" height="260" alt="sh vlan b" src="https://github.com/user-attachments/assets/ad49632f-3283-43c7-aef7-5f3b40568bd0" />

Pour la redondance de passerelle, j'ai mis du **HSRP** par VLAN, mais pas tout sur le même switch — j'ai réparti la charge : MLS-1 est actif pour les VLANs 10 et 20, MLS-2 est actif pour 30 et 99. Chaque switch sert donc du trafic réel en fonctionnement normal, pas juste en secours.

J'ai aussi aligné le **Rapid-PVST+** là-dessus : le root bridge de chaque VLAN est le même switch que l'actif HSRP de ce VLAN. Sans ça, le trafic peut faire un aller-retour inutile entre les deux switchs avant de sortir (hairpin L2/L3) — un détail qu'on ne voit dans aucun cours mais qui change vraiment le comportement du réseau.

Ces deux switchs servent aussi de **serveurs DHCP** pour tous les sites (siège, Rabat, Marrakech), en split-scope entre MLS-1 et MLS-2.

**Bug rencontré et corrigé ici** : le relais DHCP (`ip helper-address`) pour les agences distantes. Ma première tentative pointait vers l'IP virtuelle HSRP d'un VLAN — ce qui ne marche pas, parce que le DHCP relay a besoin de joindre l'IP réelle et stable du serveur DHCP, pas une IP qui peut changer de switch actif selon l'état HSRP. J'ai donc fait pointer les `ip helper-address` vers les **Loopback0** de MLS-1 et MLS-2 (10.255.0.3 et 10.255.0.4), qui restent toujours joignables quel que soit l'état HSRP.

*R-RABAT*

<img width="391" height="366" alt="show run | include helper-address" src="https://github.com/user-attachments/assets/bc616896-cff9-401f-83e0-edf49159923a" />

*R-MARRAKECH* 

<img width="391" height="366" alt="show run | include helper-address" src="https://github.com/user-attachments/assets/ea2a9840-bce8-4077-b08a-e5b724e669d7" />

## Étape 4 — Les switchs L2 d'accès (SW-1,2,3 et les autres)

En bout de chaîne, des switchs L2 purs (SW-1 au siège, un par agence) qui distribuent les VLANs vers les ports d'accès, avec un double uplink en trunk 802.1Q vers MLS-1 et MLS-2 pour le siège — encore une fois, pas de single point of failure.

<img width="630" height="146" alt="sh cdp ne" src="https://github.com/user-attachments/assets/2aa4e9d0-e6f4-4536-8c61-3086e2531a25" />
<img width="648" height="245" alt="sh vlan b" src="https://github.com/user-attachments/assets/943905d4-b66d-4766-a8c9-a117d43b7f9e" />
<img width="577" height="294" alt="sh int trunk" src="https://github.com/user-attachments/assets/ef434fa1-ed94-4188-be35-74549a840d2c" />



## Les tests — est-ce que tout ça marche vraiment ?

Une fois que toutes les briques étaient montées, j'ai voulu vérifier concrètement, pas juste supposer que la théorie s'applique.

**OSPF** : `show ip ospf neighbor` sur R1-Edge confirme une adjacence FULL avec R2-Edge (BDR) et les deux spokes (DROTHER) sur le tunnel, plus les deux switchs du siège sur les liens physiques.

<img width="634" height="143" alt="r1-ip ospf" src="https://github.com/user-attachments/assets/db1ad6b6-4e0e-4971-a290-8f09ed54f326" />

<img width="628" height="157" alt="Screenshot from 2026-11-14 17-00-30" src="https://github.com/user-attachments/assets/6f4dbe15-0dbd-4087-90fa-dd3464d7de74" />

**DMVPN dual-hub** : `show dmvpn detail` sur R1-Edge et sur R2-Edge montre chacun les trois pairs (l'autre hub + les deux spokes), avec les tunnels spoke marqués `UP`.

<img width="625" height="281" alt="sh dmvpn" src="https://github.com/user-attachments/assets/1c840e4c-27fb-4677-b9ee-8f60ef3d385f" />

<img width="632" height="356" alt="image" src="https://github.com/user-attachments/assets/763d1418-8205-4044-a926-29bc7b76c941" />

**IPsec sur les deux hubs** : `show crypto isakmp sa` et `show crypto ipsec sa` confirment des sessions `QM_IDLE` / `UP-ACTIVE` avec `esp-256-aes esp-sha256-hmac`, zéro paquet perdu, sur R1-Edge comme sur R2-Edge après correction du bug du pre-share key.

<img width="598" height="378" alt="show crypto ipsec sa" src="https://github.com/user-attachments/assets/7b52b3bc-e344-4409-8e9a-249b106d0381" />
<img width="586" height="365" alt="show crypto ipsec sa " src="https://github.com/user-attachments/assets/ebf0c5e4-160d-44e7-9112-5aa6db4dee36" />

**Spoke-to-spoke — la preuve la plus importante du projet** : j'ai voulu confirmer que le trafic Rabat ↔ Marrakech passe vraiment en direct, et pas via le siège.
- Sur R-RABAT, `show crypto isakmp sa` montre une session IKE active directement avec 10.0.0.14 (Marrakech), en plus de celle avec le hub.
  
  <img width="525" height="107" alt="shcry" src="https://github.com/user-attachments/assets/7006f101-f782-493c-b06a-dbefd73045a1" />
  <img width="504" height="114" alt="sh crypto iskamp " src="https://github.com/user-attachments/assets/f606ec2f-bc7f-414a-864f-df003349869e" />

- J'ai capturé le trafic avec Wireshark sur le lien ISP ↔ R-RABAT pendant un test PC Rabat → PC Marrakech. Résultat : une série de trames **ESP échangées directement entre 10.0.0.10 et 10.0.0.14**, jamais via 10.0.0.2 ou 10.0.0.6 (les hubs). C'est la preuve réseau, pas juste un `show` : le tunnel spoke-to-spoke est réellement emprunté et chiffré.
- Pour confirmer côté applicatif, j'ai fait un `traceroute` depuis PC3 (Rabat) vers un hôte à Marrakech (192.168.230.11) : le deuxième saut arrive directement sur `172.168.0.4` (le tunnel de R-Marrakech), en un seul saut, sans passer par un hub. Le dernier saut renvoie un "destination port unreachable" — ce qui n'est pas une erreur de routage, c'est la réponse normale d'un hôte qui a bien reçu les sondes UDP du traceroute mais n'écoute pas sur ce port précis. Le paquet est donc bien arrivé jusqu'au bout.

<img width="649" height="190" alt="image" src="https://github.com/user-attachments/assets/ee648807-9618-4732-82bf-6de0f9bc3f95" />
<img width="1370" height="396" alt="ping" src="https://github.com/user-attachments/assets/f1e9a577-8e1b-41f8-b083-f9b09f53ad41" />

**NAT/PAT** : `show ip nat translations` et un `debug ip nat` en direct confirment la traduction d'un poste du LAN (192.168.10.x) vers l'IP publique du routeur Edge quand il sort vers Internet (8.8.8.8), et uniquement dans ce cas — le trafic entre sites privés (HQ ↔ Rabat ↔ Marrakech) continue de passer par le routage normal, sans NAT, puisque les deux bouts sont des réseaux privés routés.

<img width="470" height="180" alt="image" src="https://github.com/user-attachments/assets/927aeed6-cd00-479d-a92a-70c56d0f483a" />
<img width="628" height="108" alt="image" src="https://github.com/user-attachments/assets/ec1c1820-0749-4133-9115-b46dece638ff" />
<img width="590" height="195" alt="image" src="https://github.com/user-attachments/assets/b9835932-7938-433c-b1fe-2e8c72d17c2d" />

**HSRP / EtherChannel / STP** : `show standby brief` confirme la répartition de charge entre MLS-1 et MLS-2, `show etherchannel summary` confirme le Port-channel1 en LACP actif sur les 3 liens, et `show spanning-tree root` confirme que le root STP suit bien l'actif HSRP VLAN par VLAN.

***HSRP***

<img width="630" height="145" alt="ah std br" src="https://github.com/user-attachments/assets/0be78c44-59af-46e5-8f99-f1100fac4a65" />
<img width="618" height="145" alt="sh sta br" src="https://github.com/user-attachments/assets/f047ae1e-c7ac-4201-905e-a5fa8b08c35b" />

***EtherChannel***

<img width="650" height="150" alt="sh eth sum" src="https://github.com/user-attachments/assets/7537e4ed-394f-4a9b-bfb8-c253761a8da9" />
<img width="545" height="141" alt="sh eth sum" src="https://github.com/user-attachments/assets/0792cf0d-57a5-4912-96aa-589c27a38d3f" />

***STP***

<img width="644" height="398" alt="sh spa" src="https://github.com/user-attachments/assets/9e1d853b-3eaf-4a14-941d-096eb9926e89" />
<img width="634" height="396" alt="sh spa" src="https://github.com/user-attachments/assets/679f95e7-ead8-4dad-b631-67d5fcb53683" />

**DHCP relay multi-site** : `show ip dhcp binding` sur MLS-1 montre des baux actifs pour des postes de Rabat et de Marrakech, servis depuis les pools centralisés du siège — la preuve que le relais fonctionne de bout en bout.

*MLS-1*

<img width="655" height="401" alt="sh ip dhcp" src="https://github.com/user-attachments/assets/1260550b-707c-4cb2-846b-25aa64415365" />
<img width="657" height="371" alt="sh ip dhcp 1" src="https://github.com/user-attachments/assets/118d1412-5a8d-47d4-bd6f-c5c44fde2eb5" />
<img width="661" height="366" alt="sh ip dhcp2" src="https://github.com/user-attachments/assets/c5f813c5-4b7f-4741-805c-1c05ebdf1cd0" />
<img width="661" height="173" alt="sh ip dhcp bi" src="https://github.com/user-attachments/assets/9fbb3af8-8033-4ec4-b4dd-7161cbd8ce56" />

*MLS-2*

<img width="644" height="399" alt="sh ip dhcp" src="https://github.com/user-attachments/assets/105ba356-74bc-4002-a92a-b26f59073f8b" />
<img width="657" height="376" alt="sh ip dhcp1" src="https://github.com/user-attachments/assets/4bdb02c2-8583-4047-8bb2-689c242b64cc" />
<img width="652" height="369" alt="sh ip dhcp2" src="https://github.com/user-attachments/assets/47c3f801-ae74-4d17-863f-a1af57e9a9b8" />
<img width="672" height="260" alt="sh ip dhcp bin" src="https://github.com/user-attachments/assets/9b4ad613-4ce7-40f6-8f29-2c971d104c15" />

## Résultat

Le lab répond à l'objectif de départ : un siège redondant avec deux hubs DMVPN et deux switchs L3 en haute disponibilité, deux agences connectées en DMVPN Phase 3 avec tunnel spoke-to-spoke chiffré confirmé par capture, et une sortie Internet mutualisée via NAT. Chaque bug rencontré pendant la construction (mismatch ISAKMP entre les deux hubs, routage récursif sur un spoke, mauvaise cible de DHCP relay) a été diagnostiqué par la même méthode : comparer les `show` entre les équipements censés être symétriques, puis vérifier par capture ou par test réel après correction — plutôt que de supposer que la config est bonne parce qu'elle "a l'air" correcte.

## Schéma complet

Le fichier [`lab_topology.svg`](lab_topology.svg) contient le plan d'adressage détaillé (IP, VLANs, liens point-à-point), le schéma de l'overlay DMVPN et la liste des protocoles mis en œuvre.

## Compétences mobilisées

`OSPF multi-area` `EtherChannel/LACP` `Rapid-PVST+` `HSRP` `VLAN/Trunking 802.1Q` `DHCP/DHCP Relay` `NAT/PAT` `DMVPN (mGRE/NHRP)` `IPsec/IKE` `Wireshark` `GNS3` `Troubleshooting CLI avancé`

---
*Lab personnel réalisé en complément du cycle CCNA (Introduction to Networks · Switching, Routing & Wireless Essentials · Enterprise Networking, Security & Automation) et du stage de fin d'études sur architecture DMVPN.*
