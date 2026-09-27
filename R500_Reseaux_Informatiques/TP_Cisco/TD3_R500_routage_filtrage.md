# TD3 R500 – Plan de routage et filtrage simple

BUT3 parcours Cyber – Packet Tracer
Base : configuration finale du TD2.

## Topologie

| Équipement | Modèle | Rôle |
|---|---|---|
| SW1, SW2, SW3 | 3560 / 2950 | Cœur de réseau du TD2 |
| R1 | Routeur 2811 + module NM-1FE-TX | Accès DMZ et Internet |
| RFW | Routeur 2811 | Routeur « pare-feu » en bordure, NAT |
| Serv1 | Server-PT | Intranet + DHCP |
| Serv-DMZ | Server-PT | Web (HTTP) + FTP en DMZ |
| INTERNET | Server-PT | Simule Internet (serveur web) |

### Câblage

| Lien | Câble |
|---|---|
| R1 Fa0/0 ↔ SW1 Fa0/4 | droit |
| R1 Fa0/1 ↔ Serv-DMZ Fa0 | croisé |
| R1 Fa1/0 ↔ RFW Fa0/0 | croisé |
| RFW Fa0/1 ↔ INTERNET Fa0 | laissé tel quel (lien vert) |

### Plan d'adressage

| Réseau | Adresse | Équipements |
|---|---|---|
| VLAN 99 (admin) | 192.168.99.0/27 | Admin .1, SW1 .2, SW2 .3, SW3 .4 |
| VLAN 100 (user1) | 192.168.100.0/24 | SW1 .254, DHCP .10–.19 |
| VLAN 200 (user2) | 192.168.200.0/24 | SW1 .254, DHCP .10–.19 |
| VLAN 50 (Intranet) | 192.168.50.0/24 | SW1 .254, Serv1 .1 |
| VLAN 300 (SW1↔R1) | 192.168.30.0/30 | SW1 .1, R1 Fa0/0 .2 |
| DMZ | 172.31.16.184/29 | Serv-DMZ .185, R1 Fa0/1 .190 |
| Lien R1↔RFW | 172.31.16.32/30 | R1 Fa1/0 .33, RFW Fa0/0 .34 |
| Internet | 123.0.0.0/8 | RFW Fa0/1 123.2.2.2, INTERNET 123.1.1.1 |

> L'adressage des VLANs 50 et 300 n'est pas imposé par le sujet, c'est un choix personnel :
> - 3e octet calqué sur le numéro de VLAN quand c'est possible. Pour le VLAN 300, 192.168.300.0 est impossible (un octet ne dépasse pas 255), donc 192.168.30.0.
> - Un /30 pour le VLAN 300, car c'est un lien point à point avec seulement 2 équipements.

Calculs DMZ et lien R1↔RFW :
- 172.31.16.184/29 : .184 = réseau, .185 à .190 utilisables, .191 = broadcast. Serv-DMZ = première (.185), R1 = dernière (.190).
- 172.31.16.32/30 : .32 = réseau, .33 et .34 utilisables, .35 = broadcast. R1 = première (.33), RFW = .34.

---

## Niveau essentiel

### 1. Reprise du TD2

Ouverture de la sauvegarde Packet Tracer du TD2 et vérification : DHCP OK sur les stations, ping ST11 → ST21 OK, ping Admin → 192.168.99.2 OK.

### 2. Préparation de R1

Le 2811 n'a que Fa0/0 et Fa0/1 de base. Pour avoir Fa1/0 :
1. Onglet Physical → éteindre le routeur
2. Glisser un module **NM-1FE-TX** dans un slot libre
3. Rallumer

### 3. Câblage

> **Problèmes rencontrés :**
> - Le lien R1 ↔ RFW avait été fait en câble droit : refait en **croisé** (routeur ↔ routeur).
> - Tous les voyants des routeurs étaient rouges au départ : c'est normal, les interfaces d'un routeur sont en `shutdown` par défaut. Elles passent au vert après `no shutdown`.
> - Pour RFW ↔ INTERNET, le schéma du sujet montre un trait plein, mais Internet y est dessiné comme un nuage alors qu'on le simule avec un serveur. Le lien est passé up/up avec le câble posé, donc il a été gardé.

### 4. Configuration de R1

```
enable
conf t
hostname R1
interface fa0/1
 ip address 172.31.16.190 255.255.255.248
 no shutdown
interface fa1/0
 ip address 172.31.16.33 255.255.255.252
 no shutdown
end
show ip interface brief
```

> **Problème :** une première tentative avait mis `ip address 172.31.16.184 255.255.0.0` (adresse réseau + mauvais masque), retirée avec `no ip address`. Le `show ip interface brief` confirme qu'il ne reste que .190 sur Fa0/1 et .33 sur Fa1/0.

Juste après, Fa1/0 était en up/down : normal, RFW Fa0/0 n'était pas encore configuré en face.

### 5. Configuration de RFW

```
enable
conf t
hostname RFW
interface fa0/0
 ip address 172.31.16.34 255.255.255.252
 no shutdown
interface fa0/1
 ip address 123.2.2.2 255.0.0.0
 no shutdown
end
show ip interface brief
```

Fa0/0 et Fa0/1 en up/up, et R1 Fa1/0 passe aussi en up/up.

### 6. Serveurs

**Serv-DMZ** (Desktop → IP Configuration) :
```
IP : 172.31.16.185
Masque : 255.255.255.248
Passerelle : 172.31.16.190
```
Services : HTTP **On**, FTP **On**, DHCP **Off**, le reste Off.

La passerelle est indispensable pour que le serveur puisse répondre à des machines hors de son réseau (même problème qu'avec le DHCP au TD2).

**INTERNET :**
```
IP : 123.1.1.1
Masque : 255.0.0.0
```
Services : HTTP **On**, DHCP **Off**.

### 7. Vérifications

- R1 : `ping 172.31.16.185` et `ping 172.31.16.34` → OK
- RFW : `ping 123.1.1.1` → OK

---

## Niveau fonctionnel

### 1. VLANs 50 et 300 sur SW1

```
conf t
vlan 50
 name INTRANET
vlan 300
 name R1
exit
interface fa0/5
 switchport mode access
 switchport access vlan 50
interface fa0/4
 switchport mode access
 switchport access vlan 300
end
show vlan brief
```

Fa0/5 (Serv1) dans le VLAN 50, Fa0/4 (R1) dans le VLAN 300.

### 2. Intranet : Serv1 dans le VLAN 50

**SW1 :**
```
conf t
interface vlan 50
 ip address 192.168.50.254 255.255.255.0
 no shutdown
end
```

**Serv1 :**
```
IP : 192.168.50.1
Masque : 255.255.255.0
Passerelle : 192.168.50.254
```

Test depuis SW1 : `ping 192.168.50.1` → OK

### 3. Remise en route du DHCP

Serv1 a changé de VLAN et d'adresse, donc les deux VLANs utilisateurs passent maintenant par le relais.

**SW1 :**
```
conf t
interface vlan 100
 ip helper-address 192.168.50.1
interface vlan 200
 no ip helper-address 192.168.100.50
 ip helper-address 192.168.50.1
end
```

Dans les pools DHCP de Serv1, le DNS 192.168.100.50 (qui n'existe plus) a été remplacé par **192.168.50.1**.

> **Problème rencontré :** le pool par défaut **serverPool** de Packet Tracer est lié au réseau de l'interface du serveur. Quand Serv1 est passé en 192.168.50.1, son Start IP s'est **recalé tout seul sur 192.168.50.0**, et ce pool ne pouvait plus servir le VLAN 100. Il ne peut pas être supprimé (bouton Remove grisé).
>
> Solution : création d'un nouveau pool avec le bouton **Add** (et pas Save, qui aurait écrasé serverPool) :
> ```
> Pool Name : RIMS-1
> Default Gateway : 192.168.100.254
> DNS Server : 192.168.50.1
> Start IP Address : 192.168.100.10
> Subnet Mask : 255.255.255.0
> Maximum Number of Users : 10
> ```
> serverPool est laissé tel quel : il ne répondrait qu'à un client du VLAN 50, et il n'y en a aucun. Le serveur choisit le pool grâce au giaddr du relais.

Résultat : ST11 et ST12 récupèrent bien une adresse dans leur réseau.

### 4. Lien SW1 ↔ R1 (VLAN 300)

**SW1 :**
```
conf t
interface vlan 300
 ip address 192.168.30.1 255.255.255.252
 no shutdown
end
```

**R1 :**
```
conf t
interface fa0/0
 ip address 192.168.30.2 255.255.255.252
 no shutdown
end
```

Test depuis SW1 : `ping 192.168.30.2` → OK

### 5. Routage vers la DMZ

Le routage a été fait avant le filtrage, pour pouvoir tester les restrictions ensuite.

**SW1** (route par défaut vers R1) :
```
conf t
ip route 0.0.0.0 0.0.0.0 192.168.30.2
end
```

**R1** (retour vers le LAN) :
```
conf t
ip route 192.168.99.0 255.255.255.224 192.168.30.1
ip route 192.168.100.0 255.255.255.0 192.168.30.1
ip route 192.168.200.0 255.255.255.0 192.168.30.1
ip route 192.168.50.0 255.255.255.0 192.168.30.1
end
```

La DMZ est directement connectée à R1, pas besoin de route.

Tests depuis ST11 et ST12 : `ping 172.31.16.185` et Web Browser → `http://172.31.16.185` → OK

### 6. Accès Internet : routage + NAT sur RFW

**R1** (route par défaut vers RFW) :
```
conf t
ip route 0.0.0.0 0.0.0.0 172.31.16.34
end
```

**RFW** (retour vers le LAN et la DMZ) :
```
conf t
ip route 192.168.99.0 255.255.255.224 172.31.16.33
ip route 192.168.100.0 255.255.255.0 172.31.16.33
ip route 192.168.200.0 255.255.255.0 172.31.16.33
ip route 192.168.50.0 255.255.255.0 172.31.16.33
ip route 192.168.30.0 255.255.255.252 172.31.16.33
ip route 172.31.16.184 255.255.255.248 172.31.16.33
end
```

**RFW – NAT (PAT / overload) :**
```
conf t
interface fa0/0
 ip nat inside
interface fa0/1
 ip nat outside
exit
access-list 1 permit 192.168.0.0 0.0.255.255
ip nat inside source list 1 interface fa0/1 overload
end
```

- `ip nat inside` / `outside` : côté privé et côté public
- `access-list 1` : adresses autorisées à être traduites (tout le LAN en 192.168.x.x)
- `overload` : tout le monde partage l'adresse publique 123.2.2.2, les connexions sont distinguées par le port source

Tests depuis ST11 : `ping 123.1.1.1` et Web Browser → `http://123.1.1.1` → OK

Sur RFW :
```
show ip nat translations
```
L'adresse de ST11 apparaît traduite en 123.2.2.2.

### 7. Filtrage inter-VLAN (ACL sur SW1)

ACL étendues appliquées **en entrée** sur les interfaces VLAN de SW1, pour filtrer au plus près de la source. Une ACL est lue de haut en bas et s'arrête à la première ligne qui correspond, l'ordre des lignes compte donc.

#### VLAN 200 : Intranet et DMZ uniquement

```
conf t
ip access-list extended FILTRE-V200
 permit udp any any eq bootps
 permit ip 192.168.200.0 0.0.0.255 192.168.50.0 0.0.0.255
 permit ip 192.168.200.0 0.0.0.255 172.31.16.184 0.0.0.7
 deny ip any any
exit
interface vlan 200
 ip access-group FILTRE-V200 in
end
```

La ligne `bootps` (UDP 67) est indispensable, sinon les requêtes DHCP sont bloquées avant d'être relayées.

| Test depuis ST12 | Résultat |
|---|---|
| ping 192.168.50.1 (Intranet) | ✅ |
| ping 172.31.16.185 (DMZ) | ✅ |
| ping 123.1.1.1 (Internet) | ❌ bloqué |
| ping ST11 | ❌ bloqué |
| DHCP | ✅ toujours OK |

#### VLAN 100 : Intranet, DMZ et Internet

```
conf t
ip access-list extended FILTRE-V100
 permit udp any any eq bootps
 deny ip 192.168.100.0 0.0.0.255 192.168.200.0 0.0.0.255
 deny ip 192.168.100.0 0.0.0.255 192.168.99.0 0.0.0.31
 permit ip 192.168.100.0 0.0.0.255 any
exit
interface vlan 100
 ip access-group FILTRE-V100 in
end
```

| Test depuis ST11 | Résultat |
|---|---|
| ping 192.168.50.1 (Intranet) | ✅ |
| ping 172.31.16.185 (DMZ) | ✅ |
| ping 123.1.1.1 (Internet) | ✅ |
| ping ST12 | ❌ bloqué |
| ping 192.168.99.1 (Admin) | ❌ bloqué |
| DHCP | ✅ toujours OK |

#### VLAN 99 : équipements uniquement

> **Problème :** Admin n'avait pas de passerelle depuis le TD2 (il ne parlait qu'aux switchs de son réseau). Pour joindre les routeurs : Admin → **Default Gateway = 192.168.99.2**.

```
conf t
ip access-list extended FILTRE-V99
 permit ip 192.168.99.0 0.0.0.31 192.168.99.0 0.0.0.31
 permit ip 192.168.99.0 0.0.0.31 192.168.30.0 0.0.0.3
 permit ip 192.168.99.0 0.0.0.31 172.31.16.32 0.0.0.3
 permit ip 192.168.99.0 0.0.0.31 host 172.31.16.190
 deny ip any any
exit
interface vlan 99
 ip access-group FILTRE-V99 in
end
```

- 192.168.99.0/27 : les switchs
- 192.168.30.0/30 : R1 Fa0/0
- 172.31.16.32/30 : R1 Fa1/0 et RFW Fa0/0
- host 172.31.16.190 : R1 Fa0/1 (l'interface seulement, pas le serveur DMZ)

| Test depuis Admin | Résultat |
|---|---|
| ping 192.168.99.3 (SW2) | ✅ |
| ping 192.168.30.2 (R1) | ✅ |
| ping 172.31.16.34 (RFW) | ✅ |
| ping 192.168.50.1 (Intranet) | ❌ bloqué |
| ping 172.31.16.185 (DMZ) | ❌ bloqué |
| ping 123.1.1.1 (Internet) | ❌ bloqué |
| telnet 192.168.99.2 | ✅ |

---

## Niveau complémentaire

### 1. Accès à la DMZ depuis Internet (NAT statique par port)

Une seule adresse publique (123.2.2.2), déjà utilisée par le PAT du LAN. On fait donc une redirection de ports : ce qui arrive sur 123.2.2.2 en TCP 80 et 21 part vers Serv-DMZ.

**RFW :**
```
conf t
ip nat inside source static tcp 172.31.16.185 80 123.2.2.2 80
ip nat inside source static tcp 172.31.16.185 21 123.2.2.2 21
end
show ip nat translations
```

Pour être sûr que la page affichée vient bien de Serv-DMZ (même page par défaut que le serveur INTERNET), l'index.html de Serv-DMZ a été modifié (Services → HTTP) pour afficher « Serveur DMZ ».

Tests depuis INTERNET :
- Web Browser → `http://123.2.2.2` → page « Serveur DMZ » ✅
- Command Prompt → `ftp 123.2.2.2`, identifiants Packet Tracer par défaut `cisco` / `cisco` ✅

Depuis Internet on ne voit que 123.2.2.2, jamais l'adresse privée 172.31.16.185, et seuls les ports 80 et 21 sont exposés.

### 2. Captures en mode Simulation

- **ST11 → Intranet** : filtres ARP, TCP, HTTP, Web Browser → `http://192.168.50.1`. Trajet ST11 → SW2 → trunk (VLAN 100) → SW1 (routage Vlan100 → Vlan50) → Serv1. TCP : SYN / SYN-ACK / ACK, port source > 1024, port destination 80, puis GET HTTP, réponse, FIN.
- **ST12 → Internet** : Web Browser → `http://123.1.1.1`. Le TCP SYN est bloqué sur SW1 par FILTRE-V200, la connexion ne s'établit pas (comportement voulu).
- **Comparaison ST11 → Internet** : au passage de RFW, l'adresse source 192.168.100.x est traduite en 123.2.2.2.

### 3. Évolution : ST22 limité à l'Intranet

ST22 est dans le VLAN 200, qui a accès à l'Intranet et à la DMZ. Il faut lui retirer la DMZ sans toucher à ST12.

ST22 était en DHCP, donc avec une adresse qui peut changer. Pour pouvoir le cibler dans une ACL, il lui faut une **IP fixe hors de la plage DHCP** (le pool distribue .10 à .19).

**ST22** (Desktop → IP Configuration → Static) :
```
IP : 192.168.200.20
Masque : 255.255.255.0
Passerelle : 192.168.200.254
DNS : 192.168.50.1
```

La règle pour ST22 doit être placée **avant** la ligne qui autorise la DMZ au VLAN 200, sinon elle ne sera jamais lue.

> **Problème rencontré :** `show access-lists FILTRE-V200` dans Packet Tracer n'affiche pas les numéros de séquence, donc impossible d'insérer proprement une ligne au milieu. L'ACL a été supprimée puis recréée dans le bon ordre.

**SW1 :**
```
conf t
no ip access-list extended FILTRE-V200
ip access-list extended FILTRE-V200
 permit udp any any eq bootps
 deny ip host 192.168.200.20 172.31.16.184 0.0.0.7
 permit ip 192.168.200.0 0.0.0.255 192.168.50.0 0.0.0.255
 permit ip 192.168.200.0 0.0.0.255 172.31.16.184 0.0.0.7
 deny ip any any
exit
interface vlan 200
 ip access-group FILTRE-V200 in
end
show access-lists FILTRE-V200
```

`ip access-group` est réappliqué par sécurité après la suppression et la recréation de l'ACL.

ST22 n'avait déjà pas accès à Internet ni aux autres VLANs (grâce au `deny ip any any` final). Avec la nouvelle ligne, il ne lui reste plus que l'Intranet.

| Test | Résultat |
|---|---|
| ST22 → ping 192.168.50.1 (Intranet) | ✅ |
| ST22 → ping 172.31.16.185 (DMZ) | ❌ bloqué |
| ST12 → ping 172.31.16.185 (DMZ) | ✅ toujours OK |

### 4. Sauvegarde

Sur chaque switch et routeur :
```
copy running-config startup-config
```
Plus l'enregistrement du fichier Packet Tracer (Ctrl+S).
