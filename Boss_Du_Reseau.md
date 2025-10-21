
# TP Réseau — Accès externe & firewall Linux

## Objectifs

- Apprendre à exposer un service d’une VM vers l’extérieur (port forwarding / bridged).

- Mettre en place un pare-feu Linux qui filtre et journalise le trafic.

- Comprendre la notion de DMZ et de routage entre hôte, VM et Internet.

## Phase 1 — Préparer une VM accessible depuis l’extérieur

### Situation de départ

- Une VM linux, un serveur web (python par exemple).
Le but : Rendre accessible ce service à son binôme via le LAN de l'école

### En ordre
- 1 : Ouvrir la machine virtuelle du serveur Ubuntu

- 2 : Définir le mode réseau sur NAT dans VirtualBox

- 3 : Définissez une règle de redirection de port entre l'hôte et la machine virtuelle en mode NAT.

Hôte (votre ordinateur) : localhost:8080

VM (Ubuntu) : port 8000

- 4 : Ecrire dans la machine ```` python3 -m http.server 8000````

- 5 : Tapez localhost:8080 dans votre navigateur depuis votre propre ordinateur



- [localhost](localhost.png)




##  Phase 2 — Ajouter un pare-feu / routeur Linux en frontal

### Nouvelle architecture

- [Machine hôte] → [VM Firewall/Routeur] → [VM Serveur web]

### En ordre

- 1 : Créez une deuxième VM → configurez-la sur « PARE-FEU / ROUTEUR ».

Adaptateur 1 → NAT (connexion Internet)

Adaptateur 2 → Réseau interne (intnet) → Réseau local dédié au serveur web

- 2 : Ajoutez l'adaptateur 2 (Internet) à la première VM de la même manière.

Cette VM possède désormais deux branches :

NAT (pour l'accès externe à des fins de test)

Réseau interne (réseau 192.168.100.x)

- 3 : Attribuer une adresse IP interne à la machine virtuelle du pare-feu.
````
sudo nano /etc/netplan/02-intnet.yaml

network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s8:
      dhcp4: no
      addresses: [192.168.100.1/24]

sudo netplan apply
````


enp0s8 → 192.168.100.1/24 → Définir comme passerelle LAN

- 4 : Attribuez l'adresse IP interne à la première machine virtuelle.
````
sudo nano /etc/netplan/02-intnet.yaml

network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s8:
      dhcp4: no
      addresses: [192.168.100.10/24]
      routes:
        - to: 0.0.0.0/0
          via: 192.168.100.1

sudo netplan apply
````

enp0s8 → 192.168.100.10/24

Passerelle → 192.168.100.1

- 5 : Ouvrir la redirection IP dans la machine virtuelle du pare-feu → en faire un routeur
````
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
````

- 6 : Ajouter la règle NAT MASQUERADE dans la machine virtuelle du pare-feu → Fournir l'accès du LAN à Internet
````
sudo iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE
````

- 7 : Test avec ping 8.8.8.8 depuis la VM du serveur Web → le trafic passe par le pare-feu pour aller sur Internet


- [ping](ping.png)

## Phase 3 — Analyse & durcissement

### En ordre

- 1 : Appliquer la politique de sécurité par défaut sur la machine virtuelle du pare-feu

````
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT
````
- 2 : Bloquer automatiquement tout trafic non identifié

- 3 : Autoriser uniquement le trafic HTTP (port 8000 vers le serveur Web)

````
iptables -t nat -A PREROUTING -i enp0s3 -p tcp --dport 8080 -j DNAT --to-destination 192.168.100.10:8000

iptables -A FORWARD -m state --state ESTABLISHED,RELATED -j ACCEPT

iptables -A FORWARD -i enp0s3 -o enp0s8 -p tcp --dport 8000 -m state --state NEW -j ACCEPT
````

- 4 : Le site Web s'ouvre avec succès depuis la machine hôte → le trafic passe par le pare-feu

[localhost](localhost.png)

- 5 : Bloquer et enregistrer le trafic ICMP (ping) à des fins de sécurité
````
iptables -A FORWARD -p icmp -j LOG --log-prefix "FW-ICMP-DROP: "`

iptables -A FORWARD -p icmp -j DROP
````

- 6 : Test ping 8.8.8.8 → BLOQUÉ

[pingloss](pingloss.png)

- 7 : Les connexions SSH sont également bloquées pour des raisons de sécurité.
````
iptables -A FORWARD -p tcp --dport 22 -j LOG --log-prefix "FW-SSH-BLOCK: "

iptables -A FORWARD -p tcp --dport 22 -j REJECT
````

- 8 : Avec les compteurs et les journaux iptables, on observe que le pare-feu filtre activement le trafic.

## Phase 4 — Observation et rapport (BONUS)

### En ordre

- 1 : Architecture réseau
````
HOST (browser)
   ↓ NAT Port Forwarding (HostPort:8080 → Firewall:8080)
FIREWALL VM
   - WAN (enp0s3 / NAT)
   - LAN (enp0s8 → 192.168.100.1/24)
   ↓ Internal Network (intnet)
WEB SERVER VM
   - enp0s8 → 192.168.100.10/24
   - Gateway → 192.168.100.1 (Firewall)
````

- 2 : Statut d'accès
````
| HTTP (TCP 8000 → NAT:8080) | ✅ Ouvert | consulté via l'hôte du site |

| Ping (ICMP) | ❌ Bloqué | ABANDON + enregistré par le pare-feu |

| SSH (TCP 22) | ❌ Bloqué | REJET + enregistré par le pare-feu |
````
- 3 : Test de connexion

````http://localhost:8080````

[localhost](localhost.png)

La page Web s'est ouverte avec succès → le trafic a été redirigé via le pare-feu.

- 4 : Preuve de journal

Surveillance des journaux sur le pare-feu :

````sudo journalctl -k -f | grep FW- ````

[logs](logs.png)

Les connexions Ping et SSH sont activement bloquées et enregistrées par le pare-feu

## Résultat final attendu
### Un mini-système fonctionnel et observable :

- 5 : Hebergement d'un service réel,
Service accessible à d’autres,
Protection du service et log du trafic en n’utilisant que du Linux et VirtualBox.


Yarkin ONER