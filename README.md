# VPN Site-à-Site IPsec avec pfSense sur Proxmox

Interconnexion de deux sites via un tunnel VPN IPsec.

**Message clé : "Le tunnel IPsec chiffre le trafic entre deux sites distants à travers un réseau non fiable (Internet)."**

---

## Architecture du lab

```
                        ┌─────────────────────────────────────┐
                        │        vmbr98 - WAN simulé          │
                        │         172.16.0.0/24               │
                        └──────────┬────────────────┬─────────┘
                                   │                │
                            172.16.0.1        172.16.0.2
                                   │                │
                  ┌────────────────┴────┐     ┌─────┴────────────────┐
                  │    pfSense A        │     │     pfSense B        │
                  │                     │     │                      │
                  │ WAN: 192.168.42.254 │     │ LAN: 192.168.2.1     │
                  │ LAN: 10.0.0.1       │     └───────┬──────────────┘
                  │ DMZ: 10.10.10.1     │             │
                  └──┬──────────────────┘             │
                     │                                │
        ┌────────────┴────────────┐       ┌───────────┴───────────┐
        │  vmbr2 - LAN A          │       │  vmbr11 - LAN B       │
        │  10.0.0.0/16            │       │  192.168.2.0/24       │
        └────────────┬────────────┘       └───────────┬───────────┘
                     │                                │
                 [Win11]                         [Client-B]
                 10.0.0.10                       192.168.2.10
```

| Élément | Bridge | Réseau | Rôle |
|---|---|---|---|
| WAN pfSense A | vmbr1 | 192.168.42.0/24 | Accès Internet réel |
| LAN A | vmbr2 | 10.0.0.0/16 | Réseau interne Site A |
| DMZ | vmbr99 | 10.10.10.0/24 | Zone DMZ (TP précédent) |
| **WAN simulé** | **vmbr98** | **172.16.0.0/24** | **Lien "Internet" entre les 2 sites** |
| **LAN B** | **vmbr11** | **192.168.2.0/24** | **Réseau interne Site B** |

> **⚠️ Pourquoi 192.168.2.0/24 et pas 10.0.2.0/24 ?** Le LAN A utilise le réseau 10.0.0.0/16, ce qui inclut toutes les adresses de 10.0.0.0 à 10.0.255.255. Si on utilisait 10.0.2.0/24 pour le LAN B, pfSense A considérerait que ces adresses font partie de son propre réseau local et essaierait de les joindre directement via ARP au lieu de les envoyer dans le tunnel IPsec. En utilisant 192.168.2.0/24, on évite tout chevauchement de sous-réseau.

| Machine | Type | Bridges | IP | Rôle |
|---|---|---|---|---|
| pfSense A | VM (existante) | vmbr1, vmbr2, vmbr99, **vmbr98** | 192.168.42.254 / 10.0.0.1 / 10.10.10.1 / **172.16.0.1** | Firewall Site A |
| pfSense B | **VM (nouvelle)** | **vmbr98, vmbr11** | **172.16.0.2 / 192.168.2.1** | **Firewall Site B** |
| Win11 | VM (existante) | vmbr2 | 10.0.0.10 | Client Site A |
| **Client-B** | **LXC (nouveau)** | **vmbr11** | **192.168.2.10** | **Client Site B** |

---

## Qu'est-ce qu'un VPN Site-à-Site IPsec ?

Un VPN **site-à-site** relie deux réseaux distants de manière permanente à travers Internet, comme s'ils étaient sur le même câble. Contrairement au VPN d'accès distant (comme le TP RADIUS), ce n'est pas un utilisateur qui se connecte, c'est un **réseau entier** qui se connecte à un autre réseau.

**IPsec** (Internet Protocol Security) est le protocole qui assure :

| Fonction | Protocole IPsec | Rôle |
|---|---|---|
| Négociation | **IKE** (Internet Key Exchange) | Les deux pare-feux s'authentifient et négocient les clés de chiffrement |
| Chiffrement | **ESP** (Encapsulating Security Payload) | Chiffre les données qui transitent dans le tunnel |
| Intégrité | **AH** (Authentication Header) | Garantit que les données n'ont pas été modifiées en transit |

**Les deux phases d'IPsec :**

- **Phase 1 (IKE)** : Les deux pfSense s'authentifient mutuellement (via une clé pré-partagée ou un certificat) et créent un canal sécurisé pour négocier.
- **Phase 2 (ESP)** : Dans ce canal sécurisé, ils négocient les paramètres du tunnel de données (quels réseaux, quel chiffrement), puis le tunnel s'ouvre.

```
pfSense A                                              pfSense B
172.16.0.1                                             172.16.0.2
     │                                                      │
     │── Phase 1 : "Qui es-tu ?" ─────────────────────────>│
     │<── "Je suis B, voici ma preuve" ────────────────────│
     │── "OK, voici les paramètres de chiffrement" ───────>│
     │<── "Accepté, canal sécurisé établi" ────────────────│
     │                                                      │
     │── Phase 2 : "Je veux connecter 10.0.0.0/16" ──────>│
     │<── "Moi 192.168.2.0/24, voici les clés ESP" ──────────│
     │                                                      │
     │══════════ Tunnel IPsec actif ══════════════════════│
     │   10.0.0.0/16 <──────────────────> 192.168.2.0/24     │
     │   (tout le trafic est chiffré)                       │
```

---

## Étape 1 — Créer les bridges Proxmox

### 1.1 Créer vmbr98 (WAN simulé)

Dans Proxmox : nœud → **Système** → **Réseau** → **Créer** → **Linux Bridge** :

| Champ | Valeur |
|---|---|
| Nom | vmbr98 |
| Adresse IPv4/CIDR | (laisser vide) |
| Commentaire | WAN simule VPN |

Cliquez **Créer**.

### 1.2 Créer vmbr11 (LAN B)

Même procédure :

| Champ | Valeur |
|---|---|
| Nom | vmbr11 |
| Adresse IPv4/CIDR | (laisser vide) |
| Commentaire | LAN Site B |

Cliquez **Créer** → **Appliquer la configuration**.

### 1.3 Vérification

```bash
ip link show vmbr98
ip link show vmbr11
```

Les deux bridges doivent apparaître.

---

## Étape 2 — Ajouter l'interface WAN simulé à pfSense A

### 2.1 Ajouter une carte réseau

Dans Proxmox, sélectionnez la VM pfSense A → **Matériel** → **Ajouter** → **Carte réseau** :

| Champ | Valeur |
|---|---|
| Bridge | vmbr98 |
| Modèle | VirtIO (paravirtualized) |

Cliquez **Ajouter** → **Redémarrez pfSense A**.

### 2.2 Assigner l'interface dans pfSense A

Connectez-vous à l'interface web de pfSense A (https://10.0.0.1).

1. **Interfaces** → **Assignments** → vous verrez une nouvelle interface disponible.
2. Cliquez **+ Add** → **Save**.
3. Cliquez sur la nouvelle interface (probablement **OPT2**).
4. Configurez :

| Champ | Valeur |
|---|---|
| Activer | ✅ Cocher « Enable interface » |
| Description | **VPNWAN** |
| Type de configuration IPv4 | Static IPv4 |
| Adresse IPv4 | 172.16.0.1 |
| Masque | /24 |

5. Cliquez **Save** → **Apply Changes**.

### 2.3 Règle pare-feu temporaire sur VPNWAN

Pour que pfSense B puisse joindre pfSense A sur le lien WAN simulé :

1. **Firewall** → **Rules** → onglet **VPNWAN**.
2. Cliquez **⬆ Add** :

| Champ | Valeur |
|---|---|
| Action | Pass |
| Interface | VPNWAN |
| Protocole | Any |
| Source | any |
| Destination | any |
| Description | Temporaire - Autoriser tout VPNWAN |

3. Cliquez **Save** → **Apply Changes**.

> **💡 Note :** On affinera cette règle plus tard pour n'autoriser que le trafic IPsec (UDP 500, UDP 4500 et ESP). Pour l'instant, on laisse tout passer pour faciliter la mise en place.

---

## Étape 3 — Créer pfSense B

### 3.1 Télécharger l'ISO pfSense

Si vous ne l'avez plus, téléchargez l'ISO de pfSense Community Edition et uploadez-la dans Proxmox (stockage local → ISO Images).

### 3.2 Créer la VM

Dans Proxmox → **Créer VM** :

| Onglet | Champ | Valeur |
|---|---|---|
| Général | VM ID | 300 (ou autre libre) |
| Général | Nom | pfSense-B |
| OS | ISO Image | pfSense (votre ISO) |
| Système | Type | Other |
| Disque | Taille | 8 Go |
| CPU | Cœurs | 1 |
| Mémoire | RAM | 1024 Mo (1 Go) |
| Réseau | Bridge | **vmbr98** |
| Réseau | Modèle | VirtIO |

Cliquez **Terminer** (ne démarrez pas encore).

### 3.3 Ajouter la deuxième carte réseau (LAN B)

Sélectionnez la VM pfSense-B → **Matériel** → **Ajouter** → **Carte réseau** :

| Champ | Valeur |
|---|---|
| Bridge | **vmbr11** |
| Modèle | VirtIO |

### 3.4 Installer pfSense B

1. Démarrez la VM → suivez l'assistant d'installation pfSense.
2. Acceptez les options par défaut → **Install** → **OK** → **Reboot**.
3. Au redémarrage, pfSense vous demande d'assigner les interfaces :
   - **WAN** : `vtnet0` (vmbr98)
   - **LAN** : `vtnet1` (vmbr11)

> **💡 Astuce :** Si vous n'êtes pas sûr quelle interface est laquelle, notez les adresses MAC dans Proxmox (Matériel → chaque carte réseau) et comparez avec ce que pfSense affiche.

### 3.5 Configurer les IPs via la console

Dans la console pfSense B, utilisez le menu :

**Option 2 — Set interface(s) IP address :**

**WAN :**

| Champ | Valeur |
|---|---|
| Interface | WAN |
| IPv4 | 172.16.0.2 |
| Masque | 24 |
| Passerelle | 172.16.0.1 |

**LAN :**

| Champ | Valeur |
|---|---|
| Interface | LAN |
| IPv4 | 192.168.2.1 |
| Masque | 24 |

> **⚠️ Important :** Quand pfSense demande « Do you want to revert to HTTP as the webConfigurator protocol? », répondez **y** (oui). Sinon vous devrez utiliser HTTPS avec un certificat auto-signé, ce qui complique l'accès à l'interface web.

### 3.6 Vérification

Depuis la console pfSense B :

**Option 7 — Ping host :**

```
172.16.0.1
```

pfSense B doit pouvoir pinguer pfSense A sur le lien WAN simulé. Si ça ne marche pas, vérifiez la règle pare-feu temporaire sur VPNWAN (étape 2.3).

---

## Étape 4 — Créer Client-B (LXC)

### 4.1 Créer le conteneur

Dans Proxmox → **Créer CT** :

| Onglet | Champ | Valeur |
|---|---|---|
| Général | CT ID | 201 (ou autre libre) |
| Général | Hostname | Client-B |
| Général | Mot de passe | Un mot de passe de votre choix |
| Template | Template | debian-13-standard (ou ubuntu-24.04) |
| Disque | Taille | 4 Go |
| CPU | Cœurs | 1 |
| Mémoire | RAM | 256 Mo |
| Mémoire | Swap | 256 Mo |
| Réseau | Bridge | **vmbr11** |
| Réseau | IPv4 | **Static** |
| Réseau | Adresse IPv4/CIDR | **192.168.2.10/24** |
| Réseau | Passerelle | **192.168.2.1** |
| DNS | Serveur DNS | 8.8.8.8 |

Cliquez **Terminer** → **Démarrer**.

### 4.2 Règle pare-feu sur pfSense B (LAN)

Accédez à l'interface web de pfSense B depuis Client-B : ouvrez un navigateur ou utilisez `curl` :

```bash
curl -k http://192.168.2.1
```

Ou bien configurez l'interface web via la console pfSense B.

Par défaut, pfSense autorise tout depuis le LAN. Vérifiez dans **Firewall** → **Rules** → **LAN** qu'il y a bien une règle « Default allow LAN to any ».

### 4.3 Vérification

Depuis Client-B :

```bash
ping 192.168.2.1
```

Le ping vers la passerelle pfSense B doit fonctionner.

```bash
ping 172.16.0.1
```

Le ping vers pfSense A via le WAN simulé doit aussi fonctionner (si la règle VPNWAN est en place).

> **⚠️ À ce stade, Client-B ne peut PAS pinguer 10.0.0.1 ou les machines du LAN A.** C'est normal : il n'y a pas encore de tunnel IPsec. C'est exactement ce qu'on va configurer.

---

## Étape 5 — Configurer le tunnel IPsec

### 5.1 Phase 1 sur pfSense A

Connectez-vous à pfSense A (https://10.0.0.1).

1. **VPN** → **IPsec** → onglet **Tunnels**.
2. Cliquez **+ Add P1** (ajouter Phase 1).
3. Renseignez :

**General Information :**

| Champ | Valeur |
|---|---|
| Key Exchange version | IKEv2 |
| Internet Protocol | IPv4 |
| Interface | **VPNWAN** |
| Remote Gateway | **172.16.0.2** |
| Description | VPN vers Site B |

**Phase 1 Proposal (Authentication) :**

| Champ | Valeur |
|---|---|
| Authentication Method | Mutual PSK (Pre-Shared Key) |
| My identifier | My IP address |
| Peer identifier | Peer IP address |
| Pre-Shared Key | **MonSuperVPN2025!** |

> **⚠️ Notez bien la clé pré-partagée.** Elle doit être exactement identique sur les deux pfSense. C'est le "mot de passe" qui permet aux deux pare-feux de s'authentifier mutuellement.

**Phase 1 Proposal (Encryption Algorithm) :**

| Champ | Valeur |
|---|---|
| Algorithm | AES | 256 bits |
| Hash | SHA256 |
| DH Group | 14 (2048 bit) |
| Lifetime | 28800 |

4. Cliquez **Save**.

### 5.2 Phase 2 sur pfSense A

1. Sous la Phase 1 que vous venez de créer, cliquez **+ Show Phase 2 Entries** → **+ Add P2**.
2. Renseignez :

**General Information :**

| Champ | Valeur |
|---|---|
| Mode | Tunnel IPv4 |
| Local Network | Type: **Network** / Adresse: **10.0.0.0** / Masque: **16** |
| Remote Network | Type: **Network** / Adresse: **192.168.2.0** / Masque: **24** |
| Description | LAN A vers LAN B |

> **⚠️ Important :** N'utilisez pas « LAN subnets » pour le Local Network. pfSense peut mal interpréter le masque (ex: /24 au lieu de /16). Indiquez explicitement le réseau et le masque pour éviter les surprises. Le réseau distant est en 192.168.2.0/24 — un réseau volontairement choisi en dehors de la plage 10.0.0.0/16 pour éviter les conflits de routage.

**Phase 2 Proposal (SA/Key Exchange) :**

| Champ | Valeur |
|---|---|
| Protocol | ESP |
| Encryption Algorithms | AES | 256 bits |
| Hash Algorithms | SHA256 |
| PFS key group | 14 (2048 bit) |
| Lifetime | 3600 |

3. Cliquez **Save**.

### 5.3 Activer IPsec sur pfSense A

1. Retournez dans **VPN** → **IPsec** → onglet **Tunnels**.
2. Cochez **☑ Enable IPsec** (en haut de la page).
3. Cliquez **Save** → **Apply Changes**.

### 5.4 Phase 1 sur pfSense B

Connectez-vous à pfSense B (http://192.168.2.1).

1. **VPN** → **IPsec** → onglet **Tunnels**.
2. Cliquez **+ Add P1**.
3. Renseignez :

**General Information :**

| Champ | Valeur |
|---|---|
| Key Exchange version | IKEv2 |
| Internet Protocol | IPv4 |
| Interface | **WAN** |
| Remote Gateway | **172.16.0.1** |
| Description | VPN vers Site A |

**Phase 1 Proposal (Authentication) :**

| Champ | Valeur |
|---|---|
| Authentication Method | Mutual PSK |
| My identifier | My IP address |
| Peer identifier | Peer IP address |
| Pre-Shared Key | **MonSuperVPN2025!** |

> **⚠️ La clé doit être EXACTEMENT la même** que sur pfSense A. Une seule différence de caractère et le tunnel ne s'établira jamais.

**Phase 1 Proposal (Encryption Algorithm) :**

| Champ | Valeur |
|---|---|
| Algorithm | AES | 256 bits |
| Hash | SHA256 |
| DH Group | 14 (2048 bit) |
| Lifetime | 28800 |

4. Cliquez **Save**.

### 5.5 Phase 2 sur pfSense B

1. Cliquez **+ Show Phase 2 Entries** → **+ Add P2**.
2. Renseignez :

**General Information :**

| Champ | Valeur |
|---|---|
| Mode | Tunnel IPv4 |
| Local Network | Type: **Network** / Adresse: **192.168.2.0** / Masque: **24** |
| Remote Network | Type: **Network** / Adresse: **10.0.0.0** / Masque: **16** |
| Description | LAN B vers LAN A |

> **Attention au masque :** Le réseau de Site A est en /16 (10.0.0.0/16), pas en /24. Le masque doit correspondre exactement à ce qui est configuré côté A. Le réseau local de Site B (192.168.2.0/24) a été choisi en dehors de la plage 10.0.0.0/16 pour éviter les conflits de routage avec le LAN A.

**Phase 2 Proposal :** Mêmes paramètres que pfSense A :

| Champ | Valeur |
|---|---|
| Protocol | ESP |
| Encryption Algorithms | AES | 256 bits |
| Hash Algorithms | SHA256 |
| PFS key group | 14 (2048 bit) |
| Lifetime | 3600 |

3. Cliquez **Save**.

### 5.6 Activer IPsec sur pfSense B

1. Cochez **☑ Enable IPsec**.
2. Cliquez **Save** → **Apply Changes**.

---

## Étape 6 — Règles de pare-feu pour IPsec

Le tunnel est configuré mais pfSense bloque par défaut le trafic qui sort du tunnel. Il faut autoriser le trafic IPsec sur les deux pfSense.

### 6.1 Règles sur pfSense A

**Interface VPNWAN** (autoriser le trafic IPsec entrant) :

Si vous avez encore la règle temporaire « Autoriser tout », elle couvre déjà IPsec. Sinon, créez :

1. **Firewall** → **Rules** → onglet **VPNWAN** → **⬆ Add** :

| Champ | Valeur |
|---|---|
| Action | Pass |
| Protocole | UDP |
| Source | any |
| Destination | VPNWAN address |
| Destination Port Range | From: **500** / To: **500** |
| Description | IKE (IPsec Phase 1) |

2. Ajoutez une deuxième règle :

| Champ | Valeur |
|---|---|
| Action | Pass |
| Protocole | UDP |
| Source | any |
| Destination | VPNWAN address |
| Destination Port Range | From: **4500** / To: **4500** |
| Description | NAT-T (IPsec NAT Traversal) |

3. Ajoutez une troisième règle :

| Champ | Valeur |
|---|---|
| Action | Pass |
| Protocole | ESP |
| Source | any |
| Destination | VPNWAN address |
| Description | ESP (IPsec données) |

**Interface IPsec** (autoriser le trafic dans le tunnel) :

1. **Firewall** → **Rules** → onglet **IPsec** → **⬆ Add** :

| Champ | Valeur |
|---|---|
| Action | Pass |
| Interface | IPsec |
| Protocole | Any |
| Source | Type: **Network** / Adresse: **192.168.2.0** / Masque: **24** |
| Destination | LAN subnets |
| Description | Autoriser LAN B vers LAN A via VPN |

2. Cliquez **Save** → **Apply Changes**.

### 6.2 Règles sur pfSense B

**Interface WAN** (autoriser le trafic IPsec entrant) :

Par défaut, pfSense B est en mode console. Si la règle « Block private networks » est cochée sur le WAN, **décochez-la** :

1. **Interfaces** → **WAN** → décochez **Block private networks and loopback addresses**.
2. Cliquez **Save** → **Apply Changes**.

> **⚠️ Important :** Notre WAN simulé utilise 172.16.0.0/24 qui est une plage privée. Si pfSense bloque les réseaux privés sur le WAN (option cochée par défaut), tout le trafic IPsec sera bloqué.

Ensuite ajoutez les règles sur l'interface **WAN** de pfSense B :

**Règle 1 — IKE :**

1. **Firewall** → **Rules** → onglet **WAN** → **⬆ Add** :

| Champ | Valeur |
|---|---|
| Action | Pass |
| Protocole | UDP |
| Source | any |
| Destination | WAN address |
| Destination Port Range | From: **500** / To: **500** |
| Description | IKE (IPsec Phase 1) |

2. Cliquez **Save**.

**Règle 2 — NAT-T :**

1. **⬆ Add** :

| Champ | Valeur |
|---|---|
| Action | Pass |
| Protocole | UDP |
| Source | any |
| Destination | WAN address |
| Destination Port Range | From: **4500** / To: **4500** |
| Description | NAT-T (IPsec NAT Traversal) |

2. Cliquez **Save**.

**Règle 3 — ESP :**

1. **⬆ Add** :

| Champ | Valeur |
|---|---|
| Action | Pass |
| Protocole | ESP |
| Source | any |
| Destination | WAN address |
| Description | ESP (IPsec données) |

2. Cliquez **Save** → **Apply Changes**.

**Interface IPsec** :

1. **Firewall** → **Rules** → onglet **IPsec** → **⬆ Add** :

| Champ | Valeur |
|---|---|
| Action | Pass |
| Interface | IPsec |
| Protocole | Any |
| Source | Type: **Network** / Adresse: **10.0.0.0** / Masque: **16** |
| Destination | LAN subnets |
| Description | Autoriser LAN A vers LAN B via VPN |

2. Cliquez **Save** → **Apply Changes**.

---

## Étape 7 — Tests de validation

### Test 1 — Vérifier l'état du tunnel

**Sur pfSense A :** **Status** → **IPsec** → onglet **Overview**.

Vous devez voir le tunnel avec le statut **ESTABLISHED** (Phase 1) et **INSTALLED** (Phase 2).

Si vous voyez **CONNECTING** ou rien du tout, le tunnel n'est pas monté. Vérifiez :

- La clé pré-partagée est identique des deux côtés
- Les paramètres Phase 1 et Phase 2 sont identiques
- Les règles pare-feu autorisent UDP 500, 4500 et ESP
- pfSense B ne bloque pas les réseaux privés sur le WAN

### Test 2 — Client-B vers LAN A ✅

Depuis Client-B (192.168.2.10) :

```bash
ping 10.0.0.1
```

**Résultat attendu :** Le ping fonctionne. Les paquets traversent le tunnel IPsec chiffré entre les deux sites.

```bash
traceroute 10.0.0.1
```

Le traceroute devrait montrer un saut direct vers 10.0.0.1 (le trafic passe dans le tunnel, pas par le routage normal).

### Test 3 — Win11 (LAN A) vers Client-B (LAN B) ✅

Depuis Win11 (10.0.0.10) :

```
ping 192.168.2.10
```

**Résultat attendu :** Le ping fonctionne dans les deux sens.

### Test 4 — Vérifier le chiffrement

Sur pfSense A, allez dans **Status** → **IPsec** → onglet **SPDs** (Security Policy Database).

Vous devez voir :

| Source | Destination | Protocol |
|---|---|---|
| 10.0.0.0/16 | 192.168.2.0/24 | ESP (chiffré) |
| 192.168.2.0/24 | 10.0.0.0/16 | ESP (chiffré) |

Cela confirme que tout le trafic entre les deux LANs est **chiffré par ESP**.

### Test 5 — Sans tunnel, pas de communication ❌

Pour prouver que le tunnel est indispensable :

1. Sur pfSense A, allez dans **Status** → **IPsec** → cliquez **Disconnect** sur le tunnel.
2. Depuis Client-B :

```bash
ping 10.0.0.1
```

**Résultat attendu :** Timeout. Sans le tunnel IPsec, les deux réseaux ne peuvent pas communiquer car ils sont sur des sous-réseaux différents sans route directe.

3. **Reconnectez** le tunnel → le ping refonctionne.

---

## Résumé du flux

```
Client-B                                                 Win11
192.168.2.10                                             10.0.0.10
     │                                                       │
     │                                                       │
     │── ping 10.0.0.10 ─────>│                              │
     │                         │                              │
     │                  [pfSense B]                           │
     │                  192.168.2.1                           │
     │                  Route: vers tunnel IPsec              │
     │                         │                              │
     │                ══ Tunnel IPsec chiffré (ESP) ══        │
     │                  WAN simulé: 172.16.0.0/24             │
     │                         │                              │
     │                  [pfSense A]                           │
     │                  10.0.0.1                              │
     │                  Déchiffre → route vers LAN A          │
     │                         │                              │
     │                         │── paquet déchiffré ────────> │
     │                                                       │
     │ <────────────── réponse (même chemin inverse) ──────── │
```

---

## Dépannage

### Le tunnel ne s'établit pas (CONNECTING)

| Vérification | Action |
|---|---|
| La clé PSK est-elle identique ? | Comparez lettre par lettre sur les deux pfSense |
| Les paramètres Phase 1 sont-ils identiques ? | AES-256, SHA256, DH14, IKEv2 sur les deux |
| pfSense B bloque-t-il les réseaux privés ? | Interfaces → WAN → décocher « Block private networks » |
| Les règles pare-feu autorisent-elles IPsec ? | UDP 500, UDP 4500, ESP sur VPNWAN (A) et WAN (B) |
| Les IPs sont-elles correctes ? | A: Remote = 172.16.0.2 / B: Remote = 172.16.0.1 |

### Le tunnel est ESTABLISHED mais pas de trafic

| Vérification | Action |
|---|---|
| Phase 2 est-elle INSTALLED ? | Status → IPsec → vérifiez Phase 2 |
| Les réseaux Phase 2 sont-ils corrects ? | A: Local = 10.0.0.0/16, Remote = 192.168.2.0/24 / B: Local = 192.168.2.0/24, Remote = 10.0.0.0/16 |
| Les règles IPsec existent-elles ? | Firewall → Rules → IPsec → une règle Pass doit exister |
| Le routage est-il correct ? | Depuis Client-B : `traceroute 10.0.0.1` pour voir le chemin |

### Le ping fonctionne dans un sens mais pas l'autre

| Vérification | Action |
|---|---|
| Les règles IPsec existent des deux côtés ? | Vérifiez l'onglet IPsec dans Firewall → Rules sur les DEUX pfSense |
| Les réseaux Phase 2 sont-ils symétriques ? | Le Local de A = le Remote de B et inversement |

### Erreurs courantes dans les logs

Allez dans **Status** → **System Logs** → onglet **IPsec** pour voir les logs détaillés.

| Message | Cause | Solution |
|---|---|---|
| NO_PROPOSAL_CHOSEN | Paramètres Phase 1/2 différents | Alignez les algorithmes des deux côtés |
| AUTHENTICATION_FAILED | Clé PSK différente | Corrigez la clé |
| INVALID_KE_PAYLOAD | Groupe DH différent | Mettez le même DH group (14) |
| TS_UNACCEPTABLE | Réseaux Phase 2 incorrects | Vérifiez les réseaux local/remote |

### Chevauchement de sous-réseau (piège classique)

Si le LAN A est en 10.0.0.0/16 et le LAN B en 10.0.x.0/24, le tunnel s'établit (ESTABLISHED + INSTALLED) mais **aucun trafic ne passe**. C'est parce que pfSense A considère que les adresses 10.0.x.x font partie de son propre réseau local et tente de les joindre via ARP au lieu de les envoyer dans le tunnel.

**Symptômes :**

- Phase 1 ESTABLISHED, Phase 2 INSTALLED
- Packets-In > 0 mais Packets-Out = 0 sur un des deux pfSense
- Packet capture sur enc0 montre les requests mais aucune reply
- Aucun log dans le firewall

**Solution :** Utilisez un réseau qui ne chevauche pas le LAN de l'autre site. C'est pourquoi notre LAN B utilise 192.168.2.0/24 au lieu de 10.0.2.0/24.****
