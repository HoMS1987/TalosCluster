# Notes

## things to install

- flux cli
- kubectl

## VM

MAC: 00:a0:98:5a:1a:51
Password: Schlueth1987

## Important console commands

### Flux reconcile

do this for faster adding changes from git repository

``` console
flux reconcile source git cluster -n flux-system
```

### Reboot Talos

``` console
talosctl reboot
```

### Get Replication Sources

``` console
kubectl get replicationsource -A
```

## Backup Schedule

actualserver
- data 0 1 * * * (täglich 1:00)

ddns-updater
- data 30 1 * * * (täglich 1:30)

freshrss
- config 40 1 * * * (täglich 1:40)

meshcentral
- cnpg "0 15 0 * * *" (???)
- data 50 1 * * * (täglich 1:50)
- files 0 2 * * * (täglich 2:00)
- web 10 2 * * * (täglich 2:10)
- backups 20 2 * * * (täglich 2:20)

Nextcloud
- cnpg "0 5 0 * * *" (???)
- config 10 1 * * * (täglich 1:10)
- html 20 1 * * * (täglich 1:20)

plex
- config 30 2 * * * (täglich 2:30) => für Dauer 30min einplanen

rustdesk
- data 30 3 * * * (täglich 3:30)
