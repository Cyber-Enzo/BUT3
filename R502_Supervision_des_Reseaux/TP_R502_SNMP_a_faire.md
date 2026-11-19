# TP R502 – Supervision SNMP : ce qu'il reste à faire

Groupe **R502-CYB-GR29** – VLAN **1329**

---

## 1. Plan d'adressage (ligne GR29)

| Équipement | Adresse |
|---|---|
| Switch HP 2610 | 192.168.200.129 |
| PC de gauche (eth1, sonde SNMP) | 192.168.200.141 |
| Routeur Cisco – E0 (fe0/0, côté switch) | 192.168.200.153 |
| Routeur Cisco – E1 (fe0/1, côté outside) | 172.17.0.189 |
| Route par défaut du routeur | 172.17.0.254 |
| Serveur Centreon (VM) | 192.168.200.10 |
| Serveur Windows 2022 (VM) | 192.168.200.11 |
| Serveur Linux (VM, agent SNMP + FTP) | 192.168.200.12 |

Comptes utiles :
- Serveur Linux : `user` / `bonjour`
- Windows : administrateur / `Bonjour0 ;`
- Centreon (SSH) : `root` / `centreon`
- Centreon (web, http://192.168.200.10) : `admin` / `Centreon!123`
- Proxmox : https://172.19.11.60:8006
- Communautés SNMP : `reso` (lecture seule), `admin` (lecture/écriture)

---

## 2. Branchements à refaire à chaque séance

- **PC de gauche** : `eth0` → Internet de l'IUT (uniquement pour installer des paquets), `eth1` → **port 1** du switch HP.
- **Switch HP** :
  - port **9** → **Table.3**
  - port **10** → routeur, port **fe0/0**
- **Routeur** : port **fe0/1** → **Table.1**
- **Table.1** → switch de la baie, **VLAN 1700**
- **Table.3** → switch central de la salle, **VLAN 1329, port 17**

Schéma résumé :

```
PC gauche eth1 ── port 1 [Switch HP] port 10 ── fe0/0 [Routeur] fe0/1 ── Table.1 ── VLAN 1700
                           port 9 ── Table.3 ── switch central VLAN 1329 (port 17) ── VMs Proxmox
```

Rappel : le port 9 du switch est celui qui mène au réseau central (c'est l'index d'interface utilisé dans `ifAdminStatus.9` / `ifOperStatus.9`).

---

## 3. Vérifications rapides en début de séance

- [ ] Ping switch : `ping 192.168.200.129`
- [ ] Ping routeur : `ping 192.168.200.153`
- [ ] Ping des 3 VMs : `.10`, `.11`, `.12`
- [ ] Accès Internet depuis le réseau (PAT sur le routeur, E1 = outside)
- [ ] SNMP OK : `snmpget -v1 -c reso 192.168.200.12 SNMPv2-MIB::sysDescr.0`
- [ ] Interface 9 du switch **réactivée** (`ifAdminStatus.9 = up(1)`)

> **Point d'attention** : lors d'un test, `ping 192.168.200.12` a renvoyé « Destination Host Unreachable » (depuis 192.168.200.141) et `ifOperStatus.9` était `down(2)`. C'est cohérent avec l'interface 9 désactivée pour l'exercice `snmpset`. Penser à la remettre `up` avant de continuer, et le noter dans le compte rendu.

---

## 4. Compte rendu complet à rédiger

Le compte rendu doit couvrir **toutes** les sections « À faire » du sujet, avec commandes, résultats (captures) et explications.

### 4.1 Mise en place du réseau
- [ ] Documenter la configuration de chaque équipement : switch HP (reset, IP, VLAN), routeur Cisco (IP E0/E1, route par défaut vers 172.17.0.254, PAT avec E1 en outside), PC de gauche (IP eth1), 3 VMs (IP), pare-feu Windows (règle ping).
- [ ] Valider le réseau : résultats des `ping` (switch, routeur, VMs, entre les deux stations, Internet).
- [ ] Capture de la connexion FTP au serveur Linux (`ftp 192.168.200.12`, compte `user`).

### 4.2 Paquets logiciels
- [ ] Sur le serveur Linux : `dpkg --get-selections | grep snmpd` → expliquer le résultat (statut `install`).
- [ ] Sur le PC de gauche : `dpkg --get-selections | grep snmp` → expliquer, installer `snmp` et `snmp-mibs-downloader` si absents (ou `dpkg -i *.deb` depuis ftp://148.60.210.231/_RESEAU/SNMP).

### 4.3 SNMP sur le commutateur
- [ ] Expliquer `snmp-server contact groupeX` et `snmp-server location salle-reseaux`.
- [ ] Expliquer `snmp-server community reso operator restricted` et `snmp-server community admin operator unrestricted`.

### 4.4 SNMP sur le serveur Linux
- [ ] Éditer `/etc/snmp/snmpd.conf` : `rocommunity reso`, `rwcommunity admin`, commenter `agentAddress udp:127.0.0.1:161`, décommenter `agentAddress udp:161,udp6:[::1]:161`.
- [ ] `/etc/init.d/snmpd restart`
- [ ] `netstat -lnptu` → analyser le résultat (snmpd en écoute sur UDP 161).

### 4.5 Supervision du commutateur (depuis le PC de gauche)
- [ ] Explorer le groupe `system` avec `snmpget` (`-c reso -v1 192.168.200.129 1.3.6.1.2.1.1.5.0`, `SNMPv2-MIB::sysContact.0`, etc.) et **expliquer la syntaxe** (`-v1`, `-c`, adresse, OID/nom, index `.0`).
- [ ] Capture Wireshark d'un `snmpget` et analyse (UDP 161, version, communauté en clair, PDU GetRequest/GetResponse, OID, valeur).
- [ ] `snmpset` de `sysLocation.0` (`s "salle-reseaux"`), noter la commande et le résultat.
- [ ] `snmpwalk -c reso -v1 192.168.200.129 system` → identifier la variable portant le nom du switch (`sysName`).
- [ ] Changer le nom du switch avec `snmpset` (`sysName.0`), noter commande et effet sur la configuration.
- [ ] `snmpwalk -c reso -v1 192.168.200.129 1.3.6.1.2.1.2.2 | more` → commenter, repérer les variables d'état (`ifAdminStatus`, `ifOperStatus`).
- [ ] Vérifier l'état des 2 interfaces utilisées (ports 1 et 9, ou 9 et 10 selon le câblage) avec `snmpget` sur `ifOperStatus.<index>`.
- [ ] Désactiver puis réactiver l'interface vers le réseau central : `snmpset -c admin -v1 192.168.200.129 IF-MIB::ifAdminStatus.9 i down` puis `i up`. Noter les commandes (déjà testé : `ifAdminStatus.9` / `ifOperStatus.9` = down(2)).

### 4.6 Supervision du serveur Linux
- [ ] `snmpwalk -c reso -v1 192.168.200.12 1.3.6.1.2.1.2.2.1 | more` → commenter, repérer `ifInOctets` / `ifOutOctets`.
- [ ] Calculer les débits entrant/sortant de l'interface Ethernet (deux relevés espacés d'un intervalle connu : Δoctets × 8 / Δt).
- [ ] `snmpget -v1 -c reso 192.168.200.12 SNMPv2-MIB::sysDescr.0` → résultat déjà obtenu :
  `Linux srvlinux-r502-1 6.12.43+deb13-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.12.43-1 (2025-08-27) x86_64`

### 4.7 Traps SNMP
- [ ] Ajouter dans `/etc/snmp/snmptrapd.conf` : `authCommunity log,execute,net admin` et **expliquer** (`man snmptrapd.conf` : actions autorisées log/execute/net pour la communauté `admin`).
- [ ] Lancer `snmptrapd` sur le serveur Linux.
- [ ] Sur le switch : `snmp-server enable traps snmp` et `snmp-server host 192.168.200.12 traps admin`.
- [ ] Désactiver une interface (`shutdown`), trouver où arrivent les traps (logs syslog du serveur), capturer avec Wireshark et interpréter (Trap, OID linkDown, ifIndex…).
- [ ] Réactiver l'interface et vérifier la réception du trap linkUp.

### 4.8 Centreon
- [ ] Configurer l'IP 192.168.200.10, vérifier l'accès web, vérifier que le poller tourne.
- [ ] Ajouter les hôtes (Centreon, Linux, Windows, switch, routeur si pertinent) avec leurs modèles, création des services associés.
- [ ] Exporter la configuration du poller (Configuration > Pollers) après chaque modification.
- [ ] **6.5a** Supervision simple visible / non visible (ping / ICMP) : expliquer la démarche et le protocole.
- [ ] **6.5b** Supervision avancée en SNMP (CPU, RAM, swap, disque, trafic) ; tester avec `stress` sur le serveur Linux et du trafic réseau vers le PC de gauche.
- [ ] **6.6a** Paramètres pertinents selon l'équipement (serveur : partitions, CPU, RAM ; commutateur/routeur : état et trafic des interfaces, etc.).
- [ ] **6.6b** Stratégie d'états OK / Warning / Critical : définir les seuils, puis les tester et montrer le changement d'état.

---

## 5. Nouveaux contrôles à ajouter dans Centreon (macros / services)

Objectif : étendre la supervision au-delà de ping/CPU/RAM. Pour chacun, créer une commande + un modèle de service (avec **macros** pour les seuils et paramètres), l'affecter à l'hôte concerné, exporter la configuration, puis **tester** et capturer l'état OK → Warning → Critical.

### 5.1 Validité des certificats
- Contrôler la date d'expiration d'un certificat TLS (par ex. service HTTPS d'un hôte).
- Pistes : plugin Nagios `check_http --ssl -C <jours_warning>,<jours_critical>` ou plugin Centreon HTTP/TLS équivalent.
- Macros à prévoir : `HOSTADDRESS`, port (443), `WARNINGDAYS` (ex. 30), `CRITICALDAYS` (ex. 7).
- Test : utiliser un certificat auto-signé à courte validité (`openssl req -x509 -days 5 …`) pour provoquer Warning/Critical.

### 5.2 Stress serveur
- Contrôles CPU, RAM, swap, charge (modèles Linux SNMP déjà fournis).
- Macros : seuils `WARNING` / `CRITICAL` pour chaque indicateur (ex. CPU 70 % / 90 %).
- Test : `stress --cpu 2 --timeout 120`, `stress --vm 1 --vm-bytes 512M --timeout 120` sur le serveur Linux, et observer le passage d'état et les graphiques (Monitoring > Performances).

### 5.3 Exploitation des logs
- Surveiller un fichier de log (ex. `/var/log/auth.log`, `/var/log/syslog`) à la recherche de motifs (échecs SSH, traps reçus par `snmptrapd`, etc.).
- Pistes : `check_logfiles` / plugin de recherche de motifs via SNMP (extend) ou SSH, ou traps SNMP traités par Centreon.
- Macros : chemin du fichier, motif (regex) Warning, motif Critical.
- Test : générer des événements (tentatives de connexion SSH échouées, `logger "message test"`).

### 5.4 Détection de scan nmap
- Détecter qu'un scan de ports est en cours contre un hôte supervisé (défense).
- Pistes : règles `iptables` qui journalisent (`-j LOG`) les rafales de connexions vers des ports fermés, ou outil comme `portsentry`/`fail2ban` ; puis service Centreon qui analyse ces logs (voir 5.3) ou compteur de connexions.
- Macros : seuil de tentatives sur une période, motif de log.
- Test : lancer `nmap -sS 192.168.200.12` depuis le PC de gauche, vérifier l'alerte Centreon, capturer Wireshark.

> Remarque : dans Centreon, les **macros** sont les variables (`$_SERVICEXXX$`, `$_HOSTXXX$`) utilisées par les commandes ; elles se renseignent dans les modèles de service/hôte et permettent de réutiliser la même commande avec des seuils différents.

---

## 6. Rappels pratiques

- Éteindre/rallumer les VMs depuis Proxmox ; vérifier que Centreon démarre bien (`systemctl restart cbd centengine gorgoned mariadb`).
- Ne pas oublier : `eth0` Internet seulement pour installer, puis le débrancher et reconnecter `eth1`.
- Faire des **captures d'écran lisibles** (commande + résultat) pour chaque « À faire », elles serviront directement dans le compte rendu.
- Coquille dans le sujet : l'exemple `snmpset` utilise `192.169.200.XXX`, l'adresse correcte est en `192.168.200.XXX`.
