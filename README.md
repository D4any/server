# srv-cyber-infra — Homelab (Raspberry Pi 5)

> Résumé de l'état actuel et notes importantes. Dernière mise à jour : **2026-06-25**.

---

## 📌 En une phrase
Raspberry Pi 5 réinstallé **proprement de zéro** le 2026-05-31 (après un lockout total),
accessible de façon **résiliente** par `ssh pi5`. Base saine, hardening à venir.

---

## ✅ État actuel (ce qui est fait)
- **OS neuf** : Raspberry Pi OS Lite 64-bit (Debian Trixie, kernel 6.12).
- **Hostname** : `srv-cyber-infra`.
- **Utilisateur** : `d4any` (sudo).
- **SSH actif**, avec **2 accès indépendants** :
  - 🔑 clé SSH (méthode principale),
  - 🔒 mot de passe (secours console).
- **Adressage stable via mDNS** : joignable par `srv-cyber-infra.local`, donc
  **insensible aux changements d'IP DHCP et même de box**. Résolution `.local`
  activée sur le laptop (`libnss-mdns`).
- Connexion en Ethernet sur la box actuelle.

---

## 🔌 Comment se connecter (depuis le laptop)

| Élément | Valeur |
|---|---|
| **Commande (LAN)** | `ssh pi5` (ou `ssh srv-cyber-infra.local`) |
| **Commande (à distance)** | `ssh pi5-vpn` (via Tailscale `100.80.45.108`) |
| Nom réseau | `srv-cyber-infra.local` (mDNS — suit l'IP automatiquement) |
| Utilisateur | `d4any` |
| Clé privée | `~/.ssh/id_ed25519_pi5` (sur le laptop) |
| Mot de passe | celui choisi lors de la réinstall (secours) |
| Alias SSH | défini dans `~/.ssh/config` (bloc `Host srv-cyber-infra pi5`) |

---

## ⚠️ Notes importantes à ne pas oublier
1. 🔑 **SAUVEGARDER la clé privée** `~/.ssh/id_ed25519_pi5` ailleurs
   (gestionnaire de mots de passe + clé USB). C'est ta sécurité n°1 contre un re-lockout.
2. 🧠 **Cause du lockout précédent** : SSH avait été restreint à la **seule interface
   Tailscale** + clé perdue → plus aucun accès quand Tailscale est tombé.
   **Ne jamais refaire ça.**
3. 🌐 Pas d'accès admin au routeur actuel (mot de passe inconnu) → on s'appuie sur
   le mDNS plutôt que sur une réservation DHCP.

---

## 🛠️ Ce qui reste à faire (quand tu veux, rien d'urgent)
- [x] ✅ Clé privée `id_ed25519_pi5` sauvegardée (2026-05-31).
- [x] ✅ Carte 128 Go `/data` reformatée en ext4 propre (2026-05-31). Label `DATA`,
  **nouveau UUID `5b314913-f0f9-4aad-92a3-4b5382c0536a`** (à utiliser dans le futur `/etc/fstab` du Pi).
- [x] ✅ `/data` monté sur le Pi (fstab, `noatime,nodev,nosuid,nofail`) → voir [docs/stockage-data.md](docs/stockage-data.md).
- [x] ✅ **Tailscale** (expiration de clé désactivée) → [docs/tailscale.md](docs/tailscale.md)
- [x] ✅ **ACL Tailscale — micro-segmentation** (2026-06-25, syntaxe `grants`) :
  tailnet passé du « tout ouvert » au **moindre privilège** — admin → le Pi sur tout,
  famille → DNS (53) uniquement, **zéro device-to-device**. Serveur = `tag:server-paris`,
  gens = identité. Template anonymisé dans [`infra/tailscale/`](infra/tailscale/)
  (vrais emails **uniquement** dans la console) → voir [docs/tailscale.md](docs/tailscale.md).
- [x] ✅ **UFW** (deny in / allow out, SSH LAN + Tailscale) → [docs/ufw.md](docs/ufw.md)
- [x] ✅ Arborescence `/data/{docker,apps,backups}` créée (2026-06-05).
- [x] ✅ **Docker** installé (moteur + plugin compose, dépôt officiel Trixie),
  `d4any` dans le groupe `docker` (2026-06-05).
- [x] ✅ **Pi-hole** (DNS filtrant anti-pubs) en conteneur Docker, testé OK
  → voir [docs/pihole.md](docs/pihole.md) (2026-06-05).
- [x] ✅ **Décision exposition Docker/LAN** (2026-06-10) : les ports publiés par Docker
  contournent UFW (chaîne `DOCKER` avant les règles UFW dans FORWARD) → exposition LAN
  du Pi-hole **assumée et documentée** → voir [docs/pihole.md](docs/pihole.md).
- [ ] (dette assumée) durcir avec la chaîne **`DOCKER-USER`** (filtrage admin des ports
  publiés par Docker — la méthode « prod »).
- [x] ✅ **Reverse proxy nginx + TLS** (2026-06-10) : seul point d'entrée web (443),
  vrai cert Let's Encrypt via `tailscale cert`, Pi-hole ne publie plus que le DNS
  → voir [docs/nginx.md](docs/nginx.md). Composes versionnés dans [`infra/`](infra/).
- [ ] **Renouvellement auto du cert TLS** (expire 2026-09-08) — exercice tout trouvé
  pour l'étape « automatisation ».
- [ ] **Services** suivants, étape par étape :
  **monitoring** (prochaine étape), détection (CrowdSec), etc.
- [ ] (option) brancher Docker `data-root` sur `/data/docker` (images encore sur la SD).

---

## 🛡️ Règles anti-lockout (à respecter pour la suite)
1. **Toujours 2 accès indépendants** avant d'activer une règle restrictive.
2. **Ne jamais limiter SSH à la seule interface Tailscale** — garder une voie LAN.
3. **Tester un firewall en gardant une session de secours ouverte.**
4. **Tailscale : désactiver l'expiration de clé** du nœud serveur (admin console).
5. **Garder la clé privée sauvegardée** + un mot de passe console connu.

---

## 📚 Docs détaillées (par sujet, dans `docs/`)
- [stockage-data.md](docs/stockage-data.md) — carte `/data` : format, montage fstab, organisation
- [tailscale.md](docs/tailscale.md) — VPN mesh : IPs du tailnet, accès SSH, commandes
- [ufw.md](docs/ufw.md) — pare-feu : politique, règles actives, méthode anti-lockout
- [pihole.md](docs/pihole.md) — DNS filtrant : déploiement Docker, accès, gestion, sécurité
- [nginx.md](docs/nginx.md) — reverse proxy : terminaison TLS, routage, ajout d'un service

Les `docker-compose` des services sont versionnés dans [`infra/`](infra/)
(source de vérité — le Pi exécute une copie).

---

## 🧰 Annexe — comment l'accès a été rétabli (référence)
Utile si ça se reproduit :
- ⚠️ **`rpi-imager` en version snap n'écrit PAS la personnalisation OS** sur la carte
  (hostname/user/SSH), même en confirmant « appliquer ». Vérifié 2 fois.
- **Contournement fiable** (sur une image **fraîche jamais démarrée**, partition de boot
  `bootfs` en FAT, éditable sans sudo) :
  - fichier vide **`ssh`** → active SSH à chaque boot (service `sshswitch`) ;
  - **`userconf.txt`** = `d4any:$6$<hash>` (hash via `openssl passwd -6`) → crée l'utilisateur.
- **Trouver le Pi sur le réseau sans nmap** : ping-sweep du `/24` puis `ip neigh` ;
  test infaillible = débrancher l'Ethernet et voir quelle IP disparaît.
- **mDNS** : sur le Pi `avahi-daemon` diffuse `srv-cyber-infra.local` ; sur le laptop,
  `sudo apt install libnss-mdns` pour résoudre les `.local`.
