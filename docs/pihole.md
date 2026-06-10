# Pi-hole — DNS filtrant (anti-pubs/trackers)

> Déployé le **2026-06-05** en conteneur Docker. Bloque pubs & trackers au niveau DNS.

## En une phrase
Pi-hole tourne dans Docker sur le Pi et joue le rôle de **serveur DNS** : toute
requête qui correspond à sa liste de blocage (pub/tracker) reçoit une réponse
« nulle » (`0.0.0.0`/NXDOMAIN), donc la ressource ne se charge jamais.

## Où c'est installé
- Stack : `/data/docker/pihole/`
  - `docker-compose.yml` — la recette (versionnable, pas de secret dedans).
  - `.env` — contient `PIHOLE_PASSWORD` (mot de passe admin). **Accès `-rw-------` (d4any seul).**
  - `etc-pihole/` — données persistantes (config + listes). Survit à la destruction du conteneur.
- Méthode choisie : **Docker Compose** (vs natif / `docker run`) → lisible, reproductible, propre.

## Réglages clés du compose
- `FTLCONF_dns_listeningMode: "ALL"` — **indispensable en Docker** : sinon Pi-hole
  prend les requêtes (qui arrivent par le réseau interne Docker) pour « étrangères »
  et les ignore. Sans ça, Pi-hole semble mort.
- `ports 53:53 (tcp+udp)` — le DNS uniquement. **Depuis le 2026-06-10, le port 80
  n'est plus publié** : l'admin web passe par le reverse proxy nginx (HTTPS, port 443)
  via le réseau Docker interne `proxy` → voir `docs/nginx.md`.
- `volumes ./etc-pihole:/etc/pihole` — persistance hors du conteneur.
- `FTLCONF_database_maxDBdays: "30"` — rétention de l'historique des requêtes = 30 jours
  (défaut 91). NB : Pi-hole borne par **durée**, pas par taille (pas de "X Go max").
  Limite la base + meilleure vie privée. La base vit sur `/data` (109 Go libres) → aucun
  risque de remplissage. Logs plats (`/var/log/pihole/`) gérés par `logrotate`.

## Accès
| Quoi | Valeur |
|---|---|
| Interface admin (HTTPS, via nginx) | `https://srv-cyber-infra.tail46bedb.ts.net/admin` |
| Mot de passe | dans `/data/docker/pihole/.env` |

(Depuis le 2026-06-10 : plus d'accès HTTP direct — l'admin passe par le reverse
proxy, donc nécessite d'être sur le tailnet. Le DNS :53, lui, reste publié.)

## Gestion (depuis `/data/docker/pihole/`)
- État : `docker compose ps`
- Logs : `docker compose logs -f`
- Arrêter : `docker compose down`  /  Démarrer : `docker compose up -d`
- Mettre à jour : `docker compose pull && docker compose up -d`

## Comment s'en servir au quotidien
**Règle d'or (cf. README) : un SEUL serveur DNS = le Pi-hole. JAMAIS de DNS public
(8.8.8.8…) en secondaire** → sinon round-robin/parallèle et les pubs fuient. Le repli
est toujours un *interrupteur manuel* instantané, jamais un "auto-fallback".

### Méthode ACTIVE : Tailscale global (depuis 2026-06-05)
Console admin Tailscale → **DNS** → nameserver global = `100.80.45.108` + **"Override
local DNS"**. → tous les appareils du tailnet utilisent Pi-hole **partout, quel que soit
le Wi-Fi**, automatiquement (aucun réglage par réseau). Vérifié OK : `doubleclick.net`
→ `0.0.0.0`/`::` via le résolveur **système** (sans `@IP`).
- **Secours si le Pi tombe** :
  - `tailscale set --accept-dns=false` → garde le VPN, coupe juste le DNS Pi-hole (chirurgical).
  - `tailscale down` → coupe tout le VPN, DNS inclus (gros bouton rouge).
  - réactiver : `tailscale set --accept-dns=true` / `tailscale up`.
- ⚠️ Compromis : centralisé → si le Pi tombe, **tous** les appareils perdent le DNS
  jusqu'au basculement manuel.

### Plan B : scripts nmcli (par réseau, sans Tailscale)
Sur le laptop d'admin (NetworkManager + systemd-resolved) :
- `scripts/pihole-dns-on.sh` — force le DNS sur Pi-hole (ignore le DNS du routeur, anti-fuite).
- `scripts/pihole-dns-off.sh` — remet le DNS auto du routeur.
Limite : réglage **par profil de connexion** (à refaire sur chaque nouveau Wi-Fi).

## Test de validation (fait le 2026-06-05)
```
dig +short @100.80.45.108 example.com      # -> vraie IP (DNS OK)
dig +short @100.80.45.108 doubleclick.net  # -> 0.0.0.0 (bloqué)
```

## 🔓 Exposition réseau — décision documentée (2026-06-10)
- **Constat (le « pourquoi »)** : quand un port est publié (`ports:` du compose), Docker
  insère ses propres règles iptables (chaîne `DOCKER`). Le trafic vers un conteneur est
  **NATé** et traverse la chaîne **FORWARD** — pas la chaîne INPUT où UFW filtre. Les
  règles Docker sont évaluées **avant** celles d'UFW → les ports 53/80 du Pi-hole sont
  joignables depuis le LAN **même si UFW dit "deny incoming"**. Le trafic ne viole pas
  UFW, il le **contourne**. (Vérifiable : `sudo iptables -L DOCKER -n -v`.)
- **Décision — option A** : exposition LAN **assumée et volontaire**. Raison : garder la
  porte ouverte à un futur usage « DNS de la maison » (box → Pi-hole pour les appareils
  hors tailnet : invités, TV…). Une exposition *accidentelle* est une faille ; la même
  exposition *documentée* est un choix d'architecture.
- **Dette technique assumée (option C, plus tard)** : filtrer via la chaîne
  **`DOCKER-USER`** — chaîne laissée exprès par Docker pour les règles admin, évaluée
  avant les siennes. C'est la méthode « prod » propre.
- NB : le test DNS via `100.80.45.108` passe par `allow in on tailscale0` (règle UFW
  légitime) — ce chemin-là, lui, est bien filtré par UFW.

## ⚠️ À durcir (prochaine session)
- **Docker data-root** : prévu sur `/data/docker` (cf mémoire) mais **pas encore fait** —
  les images vivent encore sur la carte SD (`/var/lib/docker`). Seules les données Pi-hole
  (volume) sont sur `/data`. À rebrancher via `/etc/docker/daemon.json` si besoin.
- Envisager un reverse proxy + TLS pour l'admin (HTTP en clair pour l'instant).
