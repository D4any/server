# Tailscale (VPN mesh)

## Rôle
Accès admin distant **chiffré** (WireGuard), **sans ouvrir aucun port** sur la box.
Le Pi reste invisible depuis l'Internet public.

## Réseau (tailnet)
| Nœud | IP Tailscale | Auth |
|---|---|---|
| `srv-cyber-infra` (Pi) | `100.80.45.108` | GitHub (D4any) |
| laptop d'admin | `100.66.7.118` | GitHub (D4any) |

- **Expiration de clé DÉSACTIVÉE** sur les 2 nœuds (console admin) → plus de déconnexion automatique. *(C'est l'oubli qui avait causé le lockout précédent.)*
- `tailscaled` **activé au boot** → remonte tout seul après redémarrage / changement de réseau.

## Accès SSH
| Depuis | Commande |
|---|---|
| Réseau local | `ssh pi5` (mDNS `srv-cyber-infra.local`) |
| N'importe où | `ssh pi5-vpn` (via Tailscale `100.80.45.108`) |

## Commandes utiles
```bash
tailscale status        # voir les machines du tailnet
tailscale ip -4         # IP Tailscale locale
sudo tailscale up       # (re)connecter / ré-authentifier
sudo tailscale down     # déconnecter
systemctl status tailscaled
```

## ⚠️ Règle anti-lockout
**Ne JAMAIS restreindre SSH à la seule interface `tailscale0`.** Toujours garder
un accès LAN en parallèle. C'est le point central à respecter quand on fera UFW.
