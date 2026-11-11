# SteamOS dans une VM Proxmox VE

🇫🇷 Français · [🇬🇧 English](README.md)

Tutoriel pas à pas pour installer SteamOS 3 (l'OS du Steam Deck) dans une machine virtuelle Proxmox VE, sans carte graphique dédiée, avec un bureau KDE Plasma utilisable et un accès SSH.

> **Avertissement.** Valve ne prend pas en charge SteamOS en machine virtuelle. La méthode vient de guides communautaires (voir [Sources](#sources)), adaptés ici à Proxmox. Elle décrit une installation réellement effectuée le 10 novembre 2026 ; elle peut cesser de fonctionner avec d'autres versions.

## Résultat attendu

- SteamOS démarre en **mode Bureau** (KDE Plasma) dans la console Proxmox.
- La VM s'administre en **SSH** depuis le réseau local.
- Le pare-feu est restreint au LAN, et les réglages survivent aux mises à jour.

**Limite importante :** le **mode Jeu** est inutilisable. Il exige le GPU AMD du Steam Deck et n'affiche qu'un écran noir dans une VM sans carte graphique. Cette VM sert donc à explorer et tester SteamOS, pas à jouer.

## Environnement testé

| Élément | Version |
| --- | --- |
| Hyperviseur | Proxmox VE 9.2 |
| Hôte | Intel NUC, Core i7-1165G7 |
| Image de récupération | `steamdeck-oobe-repair-20260707.10-3.8.14.img.bz2` |
| SteamOS après mise à jour | 3.9.5 |

## Prérequis

| Élément | Valeur utilisée | Remarque |
| --- | --- | --- |
| Image | Image de récupération du Steam Deck (`.img.bz2`) | Ce n'est pas une ISO. À télécharger sur la [page officielle de Valve](https://store.steampowered.com/steamos/download/?ver=steamdeck&snr=) |
| Firmware | UEFI (OVMF), Secure Boot désactivé | Le BIOS classique ne démarre pas SteamOS |
| Machine | q35, CPU `host`, 2 cœurs | |
| Mémoire | 8 Go | Les mises à jour peuvent échouer en dessous |
| Disque | 64 Go en provisionnement fin | Environ 15 à 20 Go réellement occupés |
| Agent invité | Aucun (`--agent 0`) | |
| Espace temporaire | Environ 8 Go dans `/root` de l'hôte | Pour l'image décompressée, supprimée à la fin |

## Conventions

- Les commandes **sur l'hôte Proxmox** se lancent en `root`.
- Les commandes **dans SteamOS** se lancent avec l'utilisateur `deck`.
- Toutes les commandes sont en syntaxe **bash**, le shell par défaut de Proxmox et de SteamOS.
- Les exemples utilisent ces valeurs, à adapter à votre installation :

| Valeur | Exemple utilisé |
| --- | --- |
| Identifiant de la VM | `216` |
| Stockage | `local-lvm` (groupe de volumes `pve`) |
| Pont réseau | `vmbr0` |
| Réseau local | `192.168.1.0/24` |

Avec ces valeurs, Proxmox nomme les disques ainsi : `vm-216-disk-0` (disque EFI), `vm-216-disk-1` (image de récupération importée), `vm-216-disk-2` (disque cible de SteamOS). Vérifiez les noms réels avec `qm config 216`.

## Installation

### 1. Télécharger et décompresser l'image

Sur l'hôte, dans `/root` (pas dans le dossier des ISO). Copiez le lien de téléchargement depuis la page de Valve :

```bash
cd /root
wget '<lien de Valve>'
bunzip2 -k steamdeck-oobe-repair-20260707.10-3.8.14.img.bz2
```

### 2. Créer la VM et importer l'image

```bash
qm create 216 --name steamos --ostype l26 --machine q35 --bios ovmf \
  --efidisk0 local-lvm:1,efitype=4m,pre-enrolled-keys=0 \
  --sockets 1 --cores 2 --cpu host --memory 8192 \
  --scsihw virtio-scsi-single --net0 virtio,bridge=vmbr0 --agent 0 && \
qm disk import 216 /root/steamdeck-oobe-repair-20260707.10-3.8.14.img local-lvm
```

L'option `pre-enrolled-keys=0` désactive le Secure Boot. L'image importée apparaît comme disque `unused0` (ici `vm-216-disk-1`).

### 3. Brancher l'image et présenter le disque cible en NVMe

Le script d'installation de Valve cherche le disque `/dev/nvme0n1`. Or Proxmox ne propose pas de disque NVMe dans son interface : on le déclare donc temporairement avec l'option `args`, qui passe des arguments bruts à QEMU.

```bash
qm set 216 --virtio1 local-lvm:vm-216-disk-1 --boot order=virtio1 && \
pvesm alloc local-lvm 216 vm-216-disk-2 64G && \
lvchange -ay pve/vm-216-disk-2 && \
qm set 216 --args '-drive file=/dev/pve/vm-216-disk-2,if=none,id=nvm,format=raw,cache=none -device nvme,drive=nvm,serial=badbeef' && \
qm start 216
```

La VM démarre sur l'image de récupération. Ouvrez sa console dans l'interface Proxmox.

### 4. Installer SteamOS

Sur le bureau de récupération :

1. Si vous n'utilisez pas un clavier QWERTY américain, ajoutez d'abord votre disposition (Configuration du système > Clavier). Vous en aurez besoin pour saisir un mot de passe à l'étape suivante.
2. Lancez **« Wipe Devices & Install SteamOS »**, puis **« Proceed »**.
3. À la fin, répondez **« Cancel »** à la proposition de redémarrage.

### 5. Activer SSH avant le premier démarrage

Sans cette étape, la VM démarre en mode Jeu sur un écran noir, sans aucun moyen d'y accéder.

SteamOS possède deux jeux de partitions système, A et B, chacun avec sa propre configuration. Il faut activer SSH dans les deux.

Dans le terminal du bureau de récupération, ouvrez un chroot sur le **jeu B** avec le script de Valve :

```bash
sudo ~/tools/repair_device.sh chroot
```

Puis, dans le chroot, définissez un mot de passe pour `deck` et activez SSH :

```bash
steamos-readonly disable
passwd deck
systemctl enable sshd
steamos-readonly enable
exit
```

Faites de même sur le **jeu A** :

```bash
sudo steamos-chroot --disk /dev/nvme0n1 --partset A --no-overlay
```

```bash
steamos-readonly disable
passwd deck
systemctl enable sshd
steamos-readonly enable
exit
```

Enfin, éteignez la VM :

```bash
sudo systemctl poweroff
```

### 6. Retirer le NVMe et rattacher le disque en virtio

Sur l'hôte, une fois la VM éteinte. La première commande interrompt la suite si la VM tourne encore :

```bash
qm status 216 | grep -q stopped && \
qm set 216 --delete args && \
qm set 216 --virtio0 local-lvm:vm-216-disk-2,discard=on,iothread=1 --boot order=virtio0 && \
qm set 216 --delete virtio1 && \
qm start 216
```

**L'écran de la console reste noir : c'est normal** (mode Jeu). Il faut maintenant trouver l'adresse IP de la VM. Relevez son adresse MAC avec `qm config 216 | grep net0`, puis balayez le réseau depuis l'hôte :

```bash
(for i in $(seq 2 254); do ping -c1 -W1 192.168.1.$i >/dev/null & done; wait)
ip neigh | grep -i <adresse MAC de la VM>
```

Connectez-vous ensuite :

```bash
ssh deck@<ip>
```

### 7. Basculer durablement sur le bureau

En SSH :

```bash
steamosctl set-default-login-mode desktop && \
steamosctl set-default-desktop-session plasma.desktop
sudo reboot
```

Après le redémarrage, le bureau KDE Plasma s'affiche dans la console Proxmox.

> **Ne cliquez pas sur l'icône « Return to Gaming Mode » du bureau** : elle remet le mode Jeu par défaut. Si cela arrive, refaites cette étape en SSH.

### 8. Restreindre le pare-feu au réseau local

Par défaut, SteamOS ouvre SSH et les ports 1024 à 65535 à tout le monde. Les commandes suivantes créent une zone `lan` limitée à votre réseau, y autorisent SSH, puis ferment la zone publique. Adaptez `192.168.1.0/24`.

```bash
sudo firewall-cmd --permanent --new-zone=lan && \
sudo firewall-cmd --permanent --zone=lan --add-source=192.168.1.0/24 && \
sudo firewall-cmd --permanent --zone=lan --add-service=ssh && \
sudo firewall-cmd --permanent --zone=public --remove-service=ssh && \
sudo firewall-cmd --permanent --zone=public --remove-port=1024-65535/tcp --remove-port=1024-65535/udp && \
sudo firewall-cmd --reload
```

Le retrait du service et celui des ports doivent rester deux commandes séparées : cette version de firewalld refuse de les combiner.

### 9. Régler le clavier et le fuseau horaire

Exemple pour un clavier français et l'heure de Paris, à adapter :

```bash
sudo localectl set-x11-keymap fr pc105 oss_nodeadkeys && \
sudo timedatectl set-timezone Europe/Paris
```

### 10. Faire survivre ces réglages aux mises à jour

**À faire avant la première mise à jour.** SteamOS réécrit `/etc` à chaque mise à jour et ne conserve que les fichiers inscrits dans une liste. Le dossier `/etc/atomic-update.conf.d/`, lui, est conservé : on y dépose donc sa propre liste.

```bash
printf '%s\n' /etc/firewalld/zones/lan.xml /etc/firewalld/zones/public.xml /etc/vconsole.conf /etc/X11/xorg.conf.d/00-keyboard.conf /etc/localtime | sudo tee /etc/atomic-update.conf.d/homelab-keep.conf
```

Le système fournit un exemple de ce format dans `/etc/atomic-update.conf.d/example-additional-keep-list.conf`.

> Lors de l'installation d'origine, cette liste a été créée après la première mise à jour : la zone `lan` et le clavier ont été perdus, et les étapes 8 et 9 ont dû être refaites.

### 11. Mettre à jour SteamOS

La mise à jour est atomique : elle s'installe sur l'autre jeu de partitions (A/B), puis le système redémarre dessus.

```bash
sudo steamos-update check && sudo steamos-update && sudo reboot
```

### 12. Nettoyer et prendre un snapshot

Sur l'hôte. L'arrêt propre n'a pas fonctionné lors de cette installation (voir [Dépannage](#dépannage)) ; il a fallu forcer avec `qm stop` :

```bash
qm stop 216 && \
qm disk unlink 216 --idlist unused0 --force && \
qm snapshot 216 steamos-propre
```

`qm disk unlink ... --force` supprime définitivement l'image de récupération, devenue `unused0` à l'étape 6. Supprimez enfin les fichiers téléchargés :

```bash
rm -v /root/steamdeck-oobe-repair-20260707.10-3.8.14.img /root/steamdeck-oobe-repair-20260707.10-3.8.14.img.bz2
```

## Dépannage

| Symptôme | Cause | Solution |
| --- | --- | --- |
| Écran noir après l'installation | La session `gamescope` (mode Jeu) est lancée par SDDM | Activer SSH dans le chroot (étape 5), puis basculer avec `steamosctl` (étape 7) |
| `switch-to-desktop-mode` échoue avec « TransactionIsDestructive » | Le service utilisateur `galileo-mura-setup` (calibrage de l'écran OLED du Deck) avait un démarrage en attente | Le mode par défaut était déjà enregistré : un redémarrage suffit |
| `firewall-cmd` refuse `--remove-service` et `--remove-port` ensemble | Limite de cette version de firewalld | Deux commandes séparées (étape 8) |
| Zone `lan` et clavier perdus après `steamos-update` | `/etc` est réécrit à chaque mise à jour, sauf les fichiers listés | Créer la liste de conservation (étape 10), puis refaire les étapes 8 et 9 |
| `qm shutdown` expire, la VM tourne toujours | Cause non élucidée : l'ACPI n'est pas pris en compte, et `sudo systemctl poweroff` lancé en SSH s'est bloqué après la coupure du réseau | `qm stop` en dernier recours. Au démarrage suivant, lire `journalctl -b -1 -n 50` |
| Retour au mode Jeu après un clic sur « Return to Gaming Mode » | L'icône remet le mode Jeu par défaut | Refaire l'étape 7 en SSH |

## Points non vérifiés

- **Liste de conservation.** Il reste à confirmer, lors d'une prochaine mise à jour, qu'elle préserve bien le pare-feu, le clavier et le fuseau horaire.
- **Arrêt de la VM.** La cause de l'échec de `qm shutdown` et de `systemctl poweroff` n'est pas identifiée.

## Sources

- [Igalia : Running the Steam Deck's OS in a virtual machine using QEMU](https://blogs.igalia.com/berto/2022/07/05/running-the-steam-decks-os-in-a-virtual-machine-using-qemu/) (mis à jour en juillet 2025 pour l'image 3.7.7)
- [MusaBase : How to virtualize SteamOS](https://www.musabase.com/2025/04/how-to-virtualize-steamos-test-its.html)
- [GamingOnLinux : HOWTO SteamOS 3 on a VM](https://www.gamingonlinux.com/forum/topic/5200/)
- [Reddit r/SteamOS : installation sur RX 9060 XT / 9070 XT](https://www.reddit.com/r/SteamOS/comments/1o2bk9b/how_to_install_steamos_on_amd_rx_9070xt_9060xt/) (astuce `repair_device.sh chroot`)
- [Page de téléchargement SteamOS de Valve](https://store.steampowered.com/steamos/download/?ver=steamdeck&snr=)
- Sur le système lui-même : `/usr/bin/steamos-session-select` et `/etc/atomic-update.conf.d/example-additional-keep-list.conf`
