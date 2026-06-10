# nginx — Reverse proxy + TLS (point d'entrée unique)

> Déployé le **2026-06-10** en conteneur Docker. Termine le TLS et route vers
> les services internes. Source de vérité : `infra/nginx/` dans ce repo.

## En une phrase
nginx est le **seul service exposé** (port 443) : il déchiffre le HTTPS
(*terminaison TLS*) puis relaie en interne vers les conteneurs, qui ne
publient plus aucun port web.

## Architecture
```
tailnet ──443/HTTPS──▶ [nginx] ──réseau docker "proxy"──▶ pihole:80
                        seul port                      ──▶ (services futurs)
                        web exposé
```
- **Avant** : chaque service publiait son port (Pi-hole :80, etc.) → surface
  d'attaque qui grossit à chaque service, et chaque port publié par Docker
  contourne UFW (cf. `docs/pihole.md`).
- **Après** : les services vivent sur le réseau interne `proxy`
  (`docker network create proxy`), sans `ports:`. Un conteneur non publié ne
  perce **aucun** trou dans le firewall → ajouter un service = **0 changement UFW**.

## Où c'est installé
- Pi : `/data/docker/nginx/` — compose + `conf.d/` + `certs/`.
- Repo : `infra/nginx/` (compose + conf versionnés, **pas les certs**).
  Toute modif se fait dans le repo puis est copiée sur le Pi (IaC).

## TLS — certificat Tailscale
- Nom : `srv-cyber-infra.tail46bedb.ts.net` (vrai cert **Let's Encrypt**,
  émis via `tailscale cert`, prérequis : HTTPS Certificates activé dans la
  console admin Tailscale). NB assumé : le nom de machine est publié dans le
  journal public des certificats (Certificate Transparency).
- Émission (sur le Pi, root) :
  ```bash
  sudo tailscale cert \
    --cert-file /data/docker/nginx/certs/srv-cyber-infra.tail46bedb.ts.net.crt \
    --key-file  /data/docker/nginx/certs/srv-cyber-infra.tail46bedb.ts.net.key \
    srv-cyber-infra.tail46bedb.ts.net
  ```
- ⚠️ **Validité 90 jours** (expire le **2026-09-08**) — renouvellement encore
  **manuel** (relancer la commande + `docker compose restart nginx`).
  À automatiser à l'étape « automatisation » de la roadmap (cron/systemd-timer).

## Routage : par chemin, pas par sous-domaine
Contrainte : Tailscale n'émet des certs **que pour le nom exact de la machine**
(pas de wildcard, donc pas de `pihole.xxx.ts.net`). → routage **par chemin** :
| URL | Service |
|---|---|
| `https://srv-cyber-infra.tail46bedb.ts.net/admin` | Pi-hole (admin) |
| *(futur)* `/grafana`, … | monitoring, … |

## Ajouter un service derrière le proxy (recette)
1. Dans le compose du service : `networks: [default, proxy]`, **aucun** `ports:`.
2. Dans `infra/nginx/conf.d/` : un bloc `location /monservice/ { proxy_pass http://<conteneur>:<port>/; }`.
3. Copier sur le Pi, `docker compose up -d` (service) + `docker compose restart nginx`.

## Gestion (depuis `/data/docker/nginx/`)
- État : `docker compose ps` · Logs : `docker compose logs -f`
- Tester une conf avant de recharger : `docker compose exec nginx nginx -t`
- Recharger après modif de conf : `docker compose restart nginx`

## Méthode appliquée (anti-lockout, généralisée)
Le port 80 du Pi-hole n'a été retiré **qu'après** vérification complète de la
nouvelle voie HTTPS (HTTP 302 du Pi-hole à travers nginx + cert valide).
Même principe que pour un firewall : **on ne ferme l'ancien chemin qu'une fois
le nouveau prouvé** — jamais les deux changements dans le même mouvement.

## Test de validation (fait le 2026-06-10)
```
curl -sI https://srv-cyber-infra.tail46bedb.ts.net/admin/   # → 302 /admin/login (Pi-hole via nginx)
openssl s_client -connect srv-cyber-infra.tail46bedb.ts.net:443  # → issuer Let's Encrypt
curl -sI http://<ip-du-pi>/admin/                           # → ne répond plus (port fermé)
dig +short @<ip-tailscale-pi> doubleclick.net               # → 0.0.0.0 (DNS intact)
```
