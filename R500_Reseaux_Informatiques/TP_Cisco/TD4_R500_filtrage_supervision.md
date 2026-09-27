# TD4 R500 – Filtrage et outils de supervision

BUT3 parcours Cyber – Packet Tracer
Base : configuration finale du TD3.

## Topologie

Même maquette que le TD3, avec en plus :

| Équipement | Modèle | Rôle |
|---|---|---|
| Serv-adm | Server-PT | Serveur de logs (Syslog), VLAN 99, sur SW1 Fa0/10 |

### Rappel de l'adressage utilisé

| Équipement | Adresse |
|---|---|
| Admin | 192.168.99.1/27, passerelle 192.168.99.2 |
| SW1 | Vlan99 192.168.99.2, Vlan300 192.168.30.1 |
| SW2 / SW3 | Vlan99 192.168.99.3 / 192.168.99.4 |
| Serv-adm | 192.168.99.10/27, passerelle 192.168.99.2 |
| R1 | Fa0/0 192.168.30.2, Fa0/1 172.31.16.190, Fa1/0 172.31.16.33 |
| RFW | Fa0/0 172.31.16.34, Fa0/1 123.2.2.2 |
| Serv-DMZ | 172.31.16.185/29 |
| INTERNET | 123.1.1.1/8 |

---

## Niveau essentiel

Reprise du fichier Packet Tracer du TD3 (enregistré sous un nouveau nom pour garder le TD3 intact).

Vérifications :
- ST11 → `ping 123.1.1.1` ✅
- ST12 → `ping 172.31.16.185` ✅, `ping 123.1.1.1` ❌ (bloqué, normal)
- Admin → `telnet 192.168.99.2` ✅

---

## Niveau fonctionnel

### 1. NAT vers la DMZ (validation)

Déjà configuré au TD3 (niveau complémentaire) sur RFW :
```
ip nat inside source static tcp 172.31.16.185 80 123.2.2.2 80
ip nat inside source static tcp 172.31.16.185 21 123.2.2.2 21
```

Test depuis INTERNET : Web Browser → `http://123.2.2.2` → page de Serv-DMZ ✅

### 2. Filtrage particulier : VLAN 200 → DMZ en HTTP uniquement

Au TD3, le VLAN 200 avait accès à tout sur la DMZ. On remplace la ligne DMZ de FILTRE-V200 par une ligne qui n'autorise que le TCP 80 vers Serv-DMZ.

Packet Tracer n'affiche pas les numéros de ligne des ACL, donc l'ACL est supprimée puis recréée. La règle ST22 du TD3 est conservée.

**SW1 :**
```
conf t
no ip access-list extended FILTRE-V200
ip access-list extended FILTRE-V200
 permit udp any any eq bootps
 deny ip host 192.168.200.20 172.31.16.184 0.0.0.7
 permit ip 192.168.200.0 0.0.0.255 192.168.50.0 0.0.0.255
 permit tcp 192.168.200.0 0.0.0.255 host 172.31.16.185 eq www
 deny ip any any
exit
interface vlan 200
 ip access-group FILTRE-V200 in
end
show access-lists FILTRE-V200
```

Seule la 4e ligne change : `permit ip ... 172.31.16.184 0.0.0.7` (tout vers la DMZ) devient `permit tcp ... host 172.31.16.185 eq www` (web uniquement, vers le serveur uniquement).

| Test depuis ST12 | Résultat |
|---|---|
| Web Browser → `http://172.31.16.185` | ✅ |
| `ping 172.31.16.185` | ❌ bloqué |
| `ftp 172.31.16.185` | ❌ bloqué |
| `ping 192.168.50.1` (Intranet) | ✅ |

### 3. Service NTP

#### Serveur INTERNET

Services → **NTP : On** (HTTP toujours On).

#### RFW

```
conf t
ntp server 123.1.1.1
end
```

RFW est directement dans le réseau 123.0.0.0/8.

#### R1

```
conf t
ntp server 123.1.1.1
end
```

R1 envoie ses requêtes NTP avec l'adresse de Fa1/0 (172.31.16.33). Or le NAT de RFW ne traduisait que 192.168.0.0/16 : la requête sortait donc avec une adresse privée non traduite. Il faut ajouter le lien R1↔RFW à l'ACL du NAT.

**RFW :**
```
conf t
access-list 1 permit 172.31.16.32 0.0.0.3
end
show access-lists 1
```

> **Problème rencontré :** la ligne avait d'abord été tapée **sur R1** au lieu de RFW, et avec une mauvaise adresse (172.31.16.21). Nettoyage sur R1 :
> ```
> conf t
> no access-list 1
> end
> ```
> Puis ajout de la bonne ligne sur RFW.

Vérification sur RFW et R1 :
```
show ntp associations
show ntp status
show clock
```
Résultat : `Clock is synchronized, stratum 2, reference is 123.1.1.1` ✅

La synchro prend du temps (une requête toutes les 64 s) : utiliser l'avance rapide ⏩ de Packet Tracer.

#### SW1

Premier essai avec `ntp server 123.1.1.1` : SW1 arrive à pinger 123.1.1.1, mais reste en `unsynchronized` avec `reach 0`, même après avance rapide.

> **Problème rencontré – limite de Packet Tracer avec le PAT :**
>
> `show ip nat translations` sur RFW :
> ```
> udp 123.2.2.2:1024   192.168.30.1:123   123.1.1.1:123   123.1.1.1:123
> udp 123.2.2.2:123    172.31.16.33:123   123.1.1.1:123   123.1.1.1:123
> ```
> Le port 123 de l'adresse publique était déjà pris par R1, donc le PAT a donné le port 1024 à SW1. Le serveur NTP de Packet Tracer répond toujours vers le port 123 : la réponse part vers R1 et SW1 ne reçoit jamais rien. La config est correcte, c'est la simulation qui ne gère pas ce cas.
>
> **Solution : hiérarchie NTP.** Les switchs se synchronisent sur R1, qui est lui-même synchronisé sur le serveur Internet (INTERNET → RFW et R1 → switchs). C'est la pratique normale en entreprise : seuls un ou deux équipements vont chercher l'heure à l'extérieur, et le reste du réseau se synchronise sur eux.

**SW1 :**
```
conf t
no ntp server 123.1.1.1
ntp server 192.168.30.2
end
```

#### SW2 et SW3

Ce sont des switchs de niveau 2 : leur seule adresse IP est celle de management (VLAN 99). Pour joindre R1, qui est dans un autre réseau, il leur faut une passerelle, SW1.

**SW2 et SW3 :**
```
conf t
ip default-gateway 192.168.99.2
ntp server 192.168.30.2
end
```

Sur un switch L2 on utilise `ip default-gateway` et pas `ip route`, car il ne fait pas de routage.

Le trafic de SW2/SW3 vers R1 passe par l'interface Vlan99 de SW1, et FILTRE-V99 autorise déjà 192.168.30.0/30.

> **Remarque :** avant de passer à la hiérarchie NTP, FILTRE-V99 avait été recréée avec une ligne autorisant le NTP vers le serveur Internet :
> ```
> conf t
> no ip access-list extended FILTRE-V99
> ip access-list extended FILTRE-V99
>  permit ip 192.168.99.0 0.0.0.31 192.168.99.0 0.0.0.31
>  permit ip 192.168.99.0 0.0.0.31 192.168.30.0 0.0.0.3
>  permit ip 192.168.99.0 0.0.0.31 172.31.16.32 0.0.0.3
>  permit ip 192.168.99.0 0.0.0.31 host 172.31.16.190
>  permit udp 192.168.99.0 0.0.0.31 host 123.1.1.1 eq 123
>  deny ip any any
> exit
> interface vlan 99
>  ip access-group FILTRE-V99 in
> end
> ```
> Avec la synchro sur R1, la ligne `permit udp ... host 123.1.1.1 eq 123` n'est plus utilisée. Elle a été laissée, elle ne gêne pas.

Vérification sur SW1, SW2, SW3 :
```
show ntp status
show clock
```
Les trois switchs sont synchronisés ✅

### 4. Gestion des logs

#### Installation de Serv-adm

- Server-PT renommé **Serv-adm**, câble droit sur **SW1 Fa0/10**

**SW1 :**
```
conf t
interface fa0/10
 switchport mode access
 switchport access vlan 99
end
```

**Serv-adm** (Desktop → IP Configuration) :
```
IP : 192.168.99.10
Masque : 255.255.255.224
Passerelle : 192.168.99.2
```

Vérifications depuis Serv-adm :
- VLAN 99 : `ping 192.168.99.1` (Admin), `.2` (SW1), `.3` (SW2), `.4` (SW3) ✅
- Tous les actifs : `ping 192.168.30.2` (R1), `ping 172.31.16.34` (RFW) ✅

#### Serveur Syslog

Serv-adm → Services → **SYSLOG : On**

#### Envoi des logs (SW1, SW2, SW3)

```
conf t
service timestamps log datetime msec
logging 192.168.99.10
end
```

- `logging 192.168.99.10` : envoie les logs au serveur Syslog
- `service timestamps log datetime msec` : horodate les logs avec la vraie date et l'heure (synchronisées par NTP), au lieu du temps écoulé depuis le démarrage

#### Tests

**SW1 – coupure de Fa0/1 :**
```
conf t
interface fa0/1
 shutdown
end
```
Logs visibles sur Serv-adm (`%LINK-5-CHANGED`, `%LINEPROTO-5-UPDOWN`) ✅

```
conf t
interface fa0/1
 no shutdown
end
```

**SW2 – coupure de Fa0/23 :**
```
conf t
interface fa0/23
 shutdown
end
```
Logs visibles sur Serv-adm ✅

```
conf t
interface fa0/23
 no shutdown
end
```

### 5. Gestion SNMP

**R1 et SW3 :**
```
conf t
snmp-server community Entreprise ro
end
```

`ro` = lecture seule. Le nom de communauté est sensible à la casse.

#### Interrogation depuis Admin

Admin → Desktop → **MIB Browser** → Advanced :
```
Address : 192.168.30.2 (R1) puis 192.168.99.4 (SW3)
Port : 161
Read Community : Entreprise
SNMP Version : v2
```

| Information | OID | Opération |
|---|---|---|
| Nom (sysName) | .1.3.6.1.2.1.1.5.0 | Get |
| Nom des interfaces (ifDescr) | .1.3.6.1.2.1.2.2.1.2 | Get Bulk |
| Type des interfaces (ifType) | .1.3.6.1.2.1.2.2.1.3 | Get Bulk |

Admin joint R1 parce que FILTRE-V99 autorise 192.168.30.0/30, et SW3 est dans son propre réseau.

Constats :
- Mêmes OID sur les deux équipements (MIB-II standard), valeurs différentes.
- sysName renvoie le hostname de chaque équipement.
- Ports Ethernet physiques : type **ethernetCsmacd (6)** sur les deux. Interfaces VLAN du switch : type virtuel (**propVirtual (53)**).
- Le dernier chiffre de l'OID est l'**ifIndex**, qui relie le nom et le type d'une même interface.

### 6. Sauvegarde

Sur SW1, SW2, SW3, R1 et RFW :
```
copy running-config startup-config
```
Et enregistrement du fichier Packet Tracer (Ctrl+S).
