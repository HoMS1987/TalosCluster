# Notes

## things to install

- flux cli
- kubectl
- talosctl
- sops (Schlüssel `age.agekey`, liegt in KeePassXC)
- helm
- gh (GitHub CLI)
- clustertool.exe (liegt im Repo)

## VM

MAC: 00:a0:98:5a:1a:51
Passwort: siehe KeePassXC

## Important console commands

### Flux reconcile

do this for faster adding changes from git repository

``` console
flux reconcile source git cluster -n flux-system
```

Nur eine App neu abgleichen (holt vorher den Git-Stand):

``` console
flux reconcile kustomization <app> -n flux-system --with-source
```

### Reboot Talos

``` console
talosctl reboot
```

### Get Replication Sources

``` console
kubectl get replicationsource -A
```

### Datenbank-Backups (CNPG)

``` console
kubectl get backups.postgresql.cnpg.io -A
kubectl get scheduledbackups.postgresql.cnpg.io -A
```

### clustertool

``` console
.\clustertool.exe genconfig
.\clustertool.exe decrypt
.\clustertool.exe encrypt
.\clustertool.exe checkcrypt
```

Vor jedem Commit verschlüsseln und mit `checkcrypt` prüfen.

## Backup Schedule

Cluster-Zeiten alle in UTC ohne Zeitzone. MESZ = UTC + 2 h, MEZ = UTC + 1 h. Dauer = typische Laufzeit (Stand 2026-10-05).

| UTC | MESZ | App | Aufgabe | Zeitplan | Dauer |
|---|---|---|---|---|---|
| 00:00 | 02:00 | longhorn | snapshot-delete | `0 0 * * *` | – |
| 00:05 | 02:05 | nextcloud | cnpg | `0 5 0 * * *` | 23 s |
| 00:20 | 02:20 | longhorn | snapshot-cleanup | `20 0 * * *` | – |
| 00:40 | 02:40 | longhorn | trim | `40 0 * * *` | – |
| 01:00 | 03:00 | actualserver | data | `0 1 * * *` | 52 s |
| 01:10 | 03:10 | nextcloud | config | `10 1 * * *` | 54 s |
| 01:20 | 03:20 | nextcloud | html | `20 1 * * *` | 1:10 min |
| 01:30 | 03:30 | ddns-updater | data | `30 1 * * *` | 50 s |
| 01:40 | 03:40 | freshrss | config | `40 1 * * *` | 52 s |
| 01:45 | 03:45 | meshcentral | cnpg | `0 45 1 * * *` | 6 s |
| 01:50 | 03:50 | meshcentral | data | `50 1 * * *` | 47 s |
| 02:00 | 04:00 | meshcentral | files | `0 2 * * *` | 51 s |
| 02:10 | 04:10 | meshcentral | web | `10 2 * * *` | 1:19 min |
| 02:20 | 04:20 | meshcentral | backups | `20 2 * * *` | 1:22 min |
| 02:30 | 04:30 | plex | config | `30 2 * * *` | 6 min, 30 min einplanen |
| 02:45 | 04:45 | paperless-ngx | export (CronJob, document_exporter nach TrueNAS) | `45 2 * * *` | erster Lauf länger, danach nur Änderungen |
| 03:30 | 05:30 | rustdesk | data | `30 3 * * *` | 50 s |
| 03:50 | 05:50 | kitchenowl | data | `50 3 * * *` | 49 s |
| 03:55 | 05:55 | kitchenowl | cnpg | `0 55 3 * * *` | 3 s |
| 04:00 | 06:00 | wg-easy | config | `0 4 * * *` | 53 s |
| 04:15 | 06:15 | immich | cnpg | `0 15 4 * * *` | 47 s |
| 04:20 | 06:20 | immich | profile | `20 4 * * *` | 52 s |
| 04:45 | 06:45 | paperless-ngx | cnpg | `0 45 4 * * *` | – |

Ohne Backup: blocky, grafana, plex-auto-languages

### TrueNAS

Zeiten in TrueNAS-Ortszeit (Europe/Berlin)

| Zeit | Aufgabe |
|---|---|
| 01:00 | Snapshot `MAIN/paperless/media`, Aufbewahrung 4 Wochen |
| 07:00 | Cloud Sync `MAIN/paperless/export` => R2 `paperless-export` (Copy, verschlüsselt), nach allen Cluster-Backups |
