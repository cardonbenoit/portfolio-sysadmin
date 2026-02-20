# RUNBOOK - journald - conserver les logs après un reboot

*Contexte*  

Sur RHEL 10, par défaut le paramètre Storage=auto : tant que /var/log/journal/ n’existe pas,
journald conserve les logs en volatile sous /run/log/journal/, et ils disparaissent au reboot.

*Objectifs*
- Garder les logs de journald de manière permanente (indépendamment des reboots)
- Paramétrer une gestion de la place des logs sur le disque
- Ne pas perdre les logs du runtime

*Infra*
- OS: **RHEL 10.1 (Coughlan)**  `cat /etc/redhat-release`
- **systemd: 257-13.el10**      `systemctl --version | head -n1`
- **/var/log FS: XFS**          `findmnt -T /var/log -no FSTYPE`

## Rappel sur journald

### Quel est le service pour journald ? 
```bash
sudo systemctl list-unit-files --type=service --no-legend --no-pager | grep -Fi journald
```
> renvoie uniquement des services statiques

### Qui déclenche/lance le service static systemd-journald ? 
```
sudo systemctl show -p TriggeredBy --value systemd-journald.service | xargs -n 1 echo
```
```text
systemd-journald-audit.socket
systemd-journald-dev-log.socket
systemd-journald.socket
```
> systemd-journald.service est déclenché par _socket activation_ 

### Unit files et Drop-in installés ?
```bash
sudo systemd-analyze cat-config systemd/journald.conf
```
```text
# /usr/lib/systemd/journald.conf
#  This file is part of systemd.
#
# Entries in this file show the compile time defaults. Local configuration
# should be created by either modifying this file (or a copy of it placed in
# /etc/ if the original file is shipped in /usr/), or by creating "drop-ins" in
# the /etc/systemd/journald.conf.d/ directory. The latter is generally
# recommended. Defaults can be restored by simply deleting the main
# configuration file and all drop-ins located in /etc/.
```

## Rendre les logs permanents

### 1ere condition : Création du fichier de conf
```bash
sudo mkdir -p /etc/systemd/journald.conf.d/
cat <<'EOF' | sudo tee /etc/systemd/journald.conf.d/00-persistent.conf
[Journal]             # Section obligatoire, case-sensitive
Storage=persistent    # Rendre les logs persistants
SystemMaxUse=500M     # Limiter la taille des logs à 500 Mo
SystemKeepFree=100M   # S'assurer qu'il reste au moins 100 Mo sur le FS qui héberge les logs
EOF

sudo chmod 644 /etc/systemd/journald.conf.d/00-persistent.conf
```

### 2eme condition : Créer le dossier de log, journald ne le fait pas seul
> voir man 8 systemd-journald.service
```bash
sudo mkdir -p /var/log/journal
sudo chgrp systemd-journal /var/log/journal
sudo chmod 2755 /var/log/journal              # Bit SGID : tout ce qui est créé hérite du groupe du répertoire
sudo systemd-tmpfiles --create --prefix /var/log/journal 
sudo restorecon -RFv /var/log/journal # par anticipation
```

> systemd-tmpfiles applique la règle tmpfiles pour ce chemin.
> `systemd-tmpfiles --cat-config | grep -nE '/var/log/journal'`
> Ici la règle contient `+C` (NoCoW - No Copy-On-Write) utile surtout sur des FS CoW (ex: btrfs).
> Dans notre cas (/ en XFS), `+C` n’apporte rien, mais la commande est sans risque.

### Redémarrage du service
```bash
sudo systemctl restart systemd-journald
sudo systemctl status systemd-journald --no-pager
```

### Vérifier la configuration effective appliquée
```bash
sudo systemd-analyze cat-config systemd/journald.conf | sed -n '/^\[Journal\]/,/^\s*$/p'
```


## Conservation des logs du runtime actuel 
`sudo journalctl --flush`
> on flush les logs du runtime en persistant
> Contrôler en comparant
```bash
tree /run/log/journal /var/log/journal 2>/dev/null
```
> Les logs sont stockés dans /var/log/journal/$(cat /run/machine-id)
> Petit aparté: => Bien supprimer le machine-id sur une VM *template* avant clonage par un `truncate -s 0 /etc/machine-id` 

## Avant un reboot, forcer un sync pour ne pas perdre les buffers en RAM
`sudo journalctl --sync`

## Contrôle du fonctionnement
```bash
logger -t 'TAG-TEST' "test persistant journald $(date)"
sudo journalctl -t TAG-TEST --no-pager # On vérifie
sudo journalctl --sync
sudo reboot
sudo journalctl -t TAG-TEST --no-pager # On doit le retrouver
```

## Surveillance du volume des logs dans le temps
```bash
sudo journalctl --disk-usage
sudo du -sh /run/log/journal /var/log/journal 2>/dev/null
```

## ROLLBACK - Revenir en logs runtime
```bash
sudo mv /etc/systemd/journald.conf.d/00-persistent.conf /etc/systemd/journald.conf.d/00-persistent.conf.disabled
# Attention - Cela ne suffit pas, car tant que /var/log/journal existe, Storage=auto peut rester en mode persistant.
sudo rm -rf /var/log/journal
sudo systemctl restart systemd-journald
```
