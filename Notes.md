# Notes

## things to install

- flux cli
- kubectl

## VM

MAC: 00:a0:98:5a:1a:51
Password: Schlueth1987

## Flux reconcile

do this for faster adding changes from git repository

``` console
flux reconcile source git cluster -n flux-system
```

## Reboot Talos

``` console
talosctl reboot
```
