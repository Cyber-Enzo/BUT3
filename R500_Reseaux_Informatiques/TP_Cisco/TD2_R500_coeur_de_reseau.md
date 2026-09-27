# TD2 R500 – Mise en place d'un cœur de réseau

BUT3 parcours Cyber – Packet Tracer

## Topologie

| Équipement | Modèle | Rôle |
|---|---|---|
| SW1 | Cisco 3560 (Multilayer Switch) | Cœur de réseau, root STP, routage inter-VLAN |
| SW2 | Cisco 2950-24 | Switch d'accès (gauche) |
| SW3 | Cisco 2950-24 | Switch d'accès (droite) |
| Admin | PC | Poste d'administration (VLAN 99) |
| Serv1 | Server-PT | Serveur DHCP |
| ST11, ST12, ST21, ST22 | PC | Stations utilisateurs |

### Câblage

| Lien | Ports |
|---|---|
| SW1 ↔ SW2 | SW1 Fa0/1 ↔ SW2 Fa0/24 |
| SW1 ↔ SW3 | SW1 Fa0/2 ↔ SW3 Fa0/24 |
| SW2 ↔ SW3 | SW2 Fa0/23 ↔ SW3 Fa0/23 |
| Admin ↔ SW1 | Admin Fa0 ↔ SW1 Fa0/3 (Ethernet) + câble console (RS232) |
| Serv1 ↔ SW1 | Serv1 Fa0 ↔ SW1 Fa0/5 |
| ST11 / ST12 | SW2 Fa0/1 / SW2 Fa0/2 |
| ST21 / ST22 | SW3 Fa0/1 / SW3 Fa0/2 |

### Plan d'adressage

| VLAN | Nom | Réseau | Passerelle (SW1) |
|---|---|---|---|
| 99 | admin | 192.168.99.0/27 | – (management) |
| 100 | user1 | 192.168.100.0/24 | 192.168.100.254 |
| 200 | user2 | 192.168.200.0/24 | 192.168.200.254 |

| Équipement | Adresse |
|---|---|
| Admin | 192.168.99.1/27 (statique) |
| SW1 (Vlan99) | 192.168.99.2/27 |
| SW2 (Vlan99) | 192.168.99.3/27 |
| SW3 (Vlan99) | 192.168.99.4/27 |
| Serv1 | 192.168.100.50/24, passerelle 192.168.100.254 |
| Stations | DHCP |

---

## Niveau essentiel

### 1. Spanning Tree – état initial

Commande sur les 3 switchs :

```
show spanning-tree
```

Constat : sans configuration, toutes les priorités valent 32768, donc le root est élu sur la MAC la plus basse. Chez nous c'était **SW2 (le switch de gauche)** qui était root, alors qu'on veut que ce soit SW1, le cœur de réseau.

### 2. Forcer SW1 comme root et bloquer Fa0/23 de SW3

**SW1 :**
```
conf t
spanning-tree vlan 1 priority 4096
end
```

**SW2** (meilleur candidat que SW3 sur le lien SW2↔SW3) :
```
conf t
spanning-tree vlan 1 priority 8192
end
```

Vérification (`show spanning-tree vlan 1`) :

- SW1 : root (priorité 4097 = 4096 + sys-id-ext 1, MAC 0030.A331.EC80)
- SW2 : priorité 8193, root port Fa0/24, Fa0/23 en **Desg FWD**
- SW3 : root port Fa0/24, **Fa0/23 en Altn BLK** ✅

> **Remarque :** un premier `show spanning-tree` sur SW3 juste après le changement montrait encore l'ancien état (Fa0/23 en Root, Fa0/24 en BLK). Il faut laisser le temps au STP de reconverger (≈ 30 à 50 s, passage par listening puis learning) avant de relever l'état.

### 3. Création des VLANs (SW1, SW2 et SW3)

```
conf t
vlan 99
 name admin
vlan 100
 name user1
vlan 200
 name user2
end
show vlan brief
```

### 4. Trunks 802.1Q

**SW1** (3560 : il faut préciser l'encapsulation avant de passer en trunk) :
```
conf t
interface range fa0/1 - 2
 switchport trunk encapsulation dot1q
 switchport mode trunk
end
```

**SW2 et SW3 :**
```
conf t
interface range fa0/23 - 24
 switchport mode trunk
end
```

Vérification :
```
show interfaces trunk
```
Les VLANs 1, 99, 100 et 200 sont autorisés et en forwarding sur les trunks.

### 5. Répartition de charge STP par VLAN (PVST+)

Objectif : Fa0/23 de SW2 bloqué pour le VLAN 100, Fa0/23 de SW3 bloqué pour le VLAN 200.

**SW1** (root pour tous les VLANs) :
```
conf t
spanning-tree vlan 1,99,100,200 priority 4096
end
```

**SW2** (prioritaire pour les VLANs 1, 99, 200) :
```
conf t
spanning-tree vlan 1,99,200 priority 8192
end
```

**SW3** (prioritaire pour le VLAN 100) :
```
conf t
spanning-tree vlan 100 priority 8192
end
```

> **Problème rencontré :** sur SW2 j'avais d'abord tapé `spanning-tree vlan 1,99,100,200 priority 8192`, donc SW2 gagnait aussi sur le VLAN 100. Correction :
> ```
> conf t
> no spanning-tree vlan 100 priority
> end
> ```
> SW2 repasse à la priorité par défaut (32868 = 32768 + 100) pour le VLAN 100.

Résultat final vérifié avec `show spanning-tree vlan 100` et `show spanning-tree vlan 200` :

| VLAN | SW2 (priorité) | SW3 (priorité) | Port bloqué |
|---|---|---|---|
| 1, 99, 200 | 8192 | 32768 | **Fa0/23 de SW3** |
| 100 | 32768 | 8192 | **Fa0/23 de SW2** |

Root = SW1 sur tous les VLANs. Les deux chemins entre SW2 et SW3 sont utilisés selon le VLAN.

### 6. Installation des stations

- Placement de ST11, ST12, ST21, ST22 et câblage en câble droit (tous les liens au vert).
- Desktop → IP Configuration → **DHCP**.

Résultat : **pas d'adresse IP**, aucun serveur DHCP n'est encore actif. Packet Tracer n'a pas généré d'APIPA à ce moment-là (champ vide), contrairement à un vrai Windows.

### 7. Affectation des ports d'accès aux VLANs

**SW2 et SW3 :**
```
conf t
interface fa0/1
 switchport mode access
 switchport access vlan 100
interface fa0/2
 switchport mode access
 switchport access vlan 200
end
show vlan brief
```

Après ça, les stations ont pris une adresse **APIPA (169.254.x.x/16)**.

> **Problèmes rencontrés :**
> - Avec l'outil enveloppe (Add Simple PDU), Packet Tracer affichait « *ST12 has no functional ports* ». J'ai fait les tests depuis Desktop → Command Prompt à la place.
> - Le ping entre stations en APIPA ne passait pas (ex. ST11 → ST21 en 169.254.131.106). Pour savoir si c'était un vrai problème réseau, j'ai vérifié SW1 (`show vlan brief`, `show interfaces trunk`) : tout était bon.
> - Test avec des **IP statiques** dans le même réseau : le ping passe. La commutation (VLAN, trunks, STP) fonctionne donc, c'est l'APIPA qui est mal simulée par Packet Tracer. Les stations ont ensuite été remises en DHCP.

---

## Niveau fonctionnel

### 1. Accès console

- Câble **console** entre Admin et SW1 (en plus du lien Ethernet sur Fa0/3).
- Admin → Desktop → **Terminal** → paramètres par défaut (9600 bauds) → Connect.
- On arrive sur le CLI de SW1, exactement comme dans l'onglet CLI du switch.

### 2. Telnet (vty 0 à 5)

Depuis le terminal Admin, sur **SW1** :
```
enable
conf t
line vty 0 5
 password Telnet
 login
end
```

La même config a aussi été appliquée sur SW2 et SW3.

À ce stade, Telnet n'est pas utilisable : ni le switch ni Admin n'ont d'adresse dans un réseau commun.

### 3. VLAN 99 d'administration

**SW1** (port d'Admin + interface de management) :
```
conf t
interface fa0/3
 switchport mode access
 switchport access vlan 99
interface vlan 99
 ip address 192.168.99.2 255.255.255.224
 no shutdown
end
```

**SW2 :**
```
conf t
interface vlan 99
 ip address 192.168.99.3 255.255.255.224
 no shutdown
end
```

**SW3 :**
```
conf t
interface vlan 99
 ip address 192.168.99.4 255.255.255.224
 no shutdown
end
```

**Admin** (Desktop → IP Configuration → Static) : 192.168.99.1 / 255.255.255.224

Tests depuis Admin : `ping 192.168.99.2`, `ping 192.168.99.3`, `ping 192.168.99.4` → OK (premier paquet parfois perdu à cause de l'ARP).

### 4. Accès Telnet et mot de passe enable

```
telnet 192.168.99.2
```

Résultat : mot de passe `Telnet` demandé, accès en mode utilisateur (`Switch>`), mais `enable` renvoie :
```
% No password set.
```
Pas d'accès au mode privilégié à distance tant qu'aucun mot de passe enable n'est défini.

**SW1 :**
```
conf t
enable password bonjour
end
```

Nouveau test : `telnet 192.168.99.2` → `Telnet` → `enable` → `bonjour` → `Switch#`. Accès complet ✅

### 5. Serveur DHCP

**Serv1** (IP statique) :
```
IP : 192.168.100.50
Masque : 255.255.255.0
Passerelle : 192.168.100.254
```

**Pool VLAN 100** (Services → DHCP, pool « serverPool ») :
```
Default Gateway : 192.168.100.254
DNS Server : 192.168.100.50
Start IP Address : 192.168.100.10
Subnet Mask : 255.255.255.0
Maximum Number of Users : 10
Service : On
```

> **Problème :** ST11 et ST21 n'obtenaient pas d'adresse. Serv1 était sur Fa0/5 de SW1, resté dans le **VLAN 1** par défaut, alors que les stations sont dans le VLAN 100. Le DHCPDISCOVER est un broadcast et ne sort pas de son VLAN.

**SW1 :**
```
conf t
interface fa0/5
 switchport mode access
 switchport access vlan 100
end
```

Après ça, ST11 et ST21 récupèrent une adresse en 192.168.100.10–19 (la première tentative a affiché « DHCP request failed », un simple renouvellement a suffi). Ping ST11 → ST21 OK.

ST12 et ST22 (VLAN 200) n'obtiennent **aucune adresse** : le broadcast du VLAN 200 n'atteint jamais le serveur qui est dans le VLAN 100.

### 6. Routage inter-VLAN et relais DHCP

**SW1 :**
```
conf t
ip routing
interface vlan 100
 ip address 192.168.100.254 255.255.255.0
 no shutdown
interface vlan 200
 ip address 192.168.200.254 255.255.255.0
 no shutdown
 ip helper-address 192.168.100.50
end
```

Vérifications :
```
show ip interface brief
show ip route
```
Vlan99, Vlan100 et Vlan200 en up/up, et les 3 réseaux apparaissent en routes connectées (C).

**Second pool sur Serv1** (RIMS2) :
```
Pool Name : RIMS2
Default Gateway : 192.168.200.254
DNS Server : 192.168.100.50
Start IP Address : 192.168.200.10
Subnet Mask : 255.255.255.0
Maximum Number of Users : 10
```

> **Problèmes rencontrés :**
> 1. Le DNS du pool RIMS2 avait été saisi en **192.168.200.50** (adresse qui n'existe pas). Corrigé en 192.168.100.50.
> 2. ST12 n'avait toujours pas d'IP. Vérifications faites :
>    - `show interfaces fa0/2 switchport` sur SW2 → bien en access VLAN 200
>    - `show ip route` sur SW1 → routes OK
>    - Mode Simulation filtré sur DHCP : la requête partait bien de ST12 et était **relayée par SW1 jusqu'à Server1**. Le relais fonctionnait, c'est la réponse qui ne revenait pas.
>    - Au passage, un ancien test ICMP (Server1 → ST12) traînait dans la liste des PDU et polluait la simulation : il a fallu le supprimer et faire *Reset Simulation*.
> 3. **Cause réelle : Serv1 n'avait pas de passerelle par défaut.** Le serveur doit répondre en unicast à l'agent relais (giaddr = 192.168.200.254), qui est dans un autre réseau. Sans passerelle, il ne sait pas comment le joindre. Pour le VLAN 100 ça marchait car les clients sont dans le même réseau que le serveur.
>
> Correction sur Serv1 : **Default Gateway = 192.168.100.254**.

Résultat : ST12 et ST22 obtiennent une adresse en 192.168.200.10–19 ✅

### Fonctionnement du relais DHCP

1. Le client envoie un DHCPDISCOVER en broadcast dans son VLAN.
2. SW1 le reçoit sur l'interface Vlan200 et, grâce à `ip helper-address`, le renvoie en **unicast** vers 192.168.100.50 en mettant son adresse (192.168.200.254) dans le champ **giaddr**.
3. Le serveur choisit le pool qui correspond au giaddr (RIMS2) et répond en unicast à SW1.
4. SW1 retransmet la réponse au client dans le VLAN 200. Même chose pour DHCPREQUEST et DHCPACK.
