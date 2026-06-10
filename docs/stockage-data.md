# Stockage — carte `/data`

## Rôle
Séparer l'OS (microSD 32 Go, `/`) des données lourdes qui grossissent (carte 128 Go, `/data`).
Si l'OS est réinstallé, `/data` survit.

## Matériel
- Carte 128 Go, adaptateur **USB 3.0** (port **bleu** du Pi)
- Filesystem **ext4**, label **`DATA`**
- UUID : `5b314913-f0f9-4aad-92a3-4b5382c0536a`
- Reformatée à neuf le 2026-05-31

## Montage persistant (`/etc/fstab`)
```
UUID=5b314913-f0f9-4aad-92a3-4b5382c0536a /data ext4 defaults,noatime,nodev,nosuid,nofail,x-systemd.device-timeout=10 0 2
```
| Option | Rôle |
|---|---|
| `noatime` | pas d'écriture de l'heure d'accès → préserve la carte |
| `nodev`, `nosuid` | durcissement (bloque devices et binaires setuid sur la carte) |
| `nofail` + `x-systemd.device-timeout=10` | **le Pi démarre même si la carte est absente/morte** (anti-lockout) |
| `0 2` | vérif fsck après la partition racine |

Point de montage : `/data`, propriété `d4any:d4any`.
Backup de l'ancien fstab : `/etc/fstab.bak.2026-05-31`.

## Commandes utiles
```bash
findmnt /data        # voir le montage
df -h /data          # espace libre
sudo mount /data     # monter (si démonté)
sudo umount /data    # démonter
sudo mount -a        # tester toutes les entrées fstab
```

## Organisation prévue (à créer à la phase suivante)
```
/data
├── docker/    → data-root Docker (images, volumes, conteneurs)
├── apps/      → docker-compose + configs des services
└── backups/
```
