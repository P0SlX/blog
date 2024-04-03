# Remplacement de la SFR Box 7 Plus
Cet article parle d'un remplacement d'une SFR Box 7 Plus par un ONT7-SFU de chez Altice avec un PC portable x86 derrière et un routeur Xiaomi configuré en "Dump AP" pour le sans-fil.
## Pourquoi ?
La box marche bien alors pourquoi vouloir la remplacer ?
**La flexibilité.**
Avoir le contrôle total sur son réseau local permet de faire quasiment ce qu'on veut. Ça serait cool si la box pouvait aussi servir de NAS en branchant quelques HDD/SSD, ça serait cool aussi de pouvoir se connecter à son réseau local de partout et quelque soit l'appareil, non ?
C'est justement mon cas, je vais détailler le processus de communication avec l'ONT et la mise en place de la "box" remplaçante ainsi que son AP (access point).
## L'Importance de l'ONT Externe
Tout d'abord, l'état du réseau actuel est ainsi : SFR -> Fibre -> Box 7.
Pas d'ONT ? Si, interne.
Le plus gros problème vient de là. Je n'ai aucun PC/SBC (single board computer) qui permet de brancher une fibre.
En ma possession, j'ai un ancien PC portable avec un i3-5005U (2c/4t) et 8Go de RAM sans écran que j'ai récupéré d'un ami et un Xiaomi Redmi AC2100.

Je me suis renseigné sur les ONT SFR externes (utilisé dans la Box 7 non Plus) et je suis tombé sur l'ONT7-SFU d'Altice. Petit bémol, c'est loué par SFR et aucun moyen de convaincre le service client de "downgrade" ma box contre l'ONT + box...
## OwN iT
Et si des gens en vendaient sur Ebay ? Eh bah oui, pour seulement 20 euros en plus !
Après réception, je devais trouver un moyen de communiquer avec lui. Sur ce que j'ai pu voir, il supporte principalement du telnet et SSH.
Maintenant, il fallait trouver l'utilisateur et le mot de passe...
Je savais que la box communiquait avec l'ONT pour récupérer les informations sur l'état de la fibre, la température etc. (accessible via l'API de la box), donc la box doit avoir les creds.
Après quelques recherches, j'ai trouvé un dump d'un ancien firmware de la Box 7 et dans les fichiers de la box, il y avait bien les creds qu'utilise la box pour communiquer en telnet avec l'ONT.
Je ne publierais pas les creds ici pour des raisons évidentes.

Pour se connecter en telnet, il faut se brancher directement à l'arrière de l'ONT et se mettre une IP fixe sur 192.168.4.0 (255.255.255.0). J'ai pris l'IP 192.168.4.100 et lancé `arp -a` pour voir où se trouve l'ONT sur le sous-réseau.

Une réponse attire mon attention sur 192.168.4.254, déjà par l'adresse IP mais aussi car c'est la seule réponse qui diffère des autres sur le réseau.

J'essaie de me connecter avec les identifiants et bingo, *I'm in* ! Du premier coup !
Ok, première fois que j'utilise du telnet alors je lance quelques commandes pour me familiariser avec.
`show gpon slid` permet d'avoir le "mot de passe fibre" (mentionné ainsi sur l'interface web de la Box 7)

Et... C'est vierge, littéralement.
Il y a deux formats d'affichage du slid, hex et ascii. En hex, ça donne : 0x20202020202020 soit des espaces en ascii. On dirait un ONT tombé du camion :)

Bon, on va tenter de le réécrire avec le mien. La commande `set gpon slid ascii <mon_slid>` me permet de faire ça et après un reboot, mon slid est bien affiché correctement !
Aucune autre commande `set` n'existe alors j'imagine qu'on peut tester ça sur ma fibre ?

Je branche tout ça, les LED passent par toutes les couleurs et deux d'entre elles bloquent sur du orange pendant environ une minute... Et d'un coup, l'ONT s'éteint.

Je le laisse un petit moment, on sait jamais s'il revient à la vie et... J'ai bien fait car environ une minute après il se rallume et cette fois-ci, toutes les LED sont vertes, ouf.
## Remplacement de la Box
Cette étape, je l'avais déjà faite auparavant chez mes parents, alors j'ai pu rapidement mettre ça en place.
J'ai naturellement été vers OpenWRT car je suis très familier avec, mais aussi car c'est l'un des meilleurs systèmes pour ce genre de chose. Aussi, mon AP sera sous ce système, ça rajoute un point pour l'utiliser.

Après avoir flashé l'OS sur une clé USB (USB 2.0 suffira comme le système se charge entièrement en RAM pendant le boot), j'ai aussi inséré deux SSD SATA de 1To pour les mettre en RAID 1 (avec `mdadm`) et faire un partage SMB et DLNA/UPNP.

Le PC portable a, naturellement, qu'un port RJ45 alors un adaptateur USB3.0/RJ45 permettra d'en rajouter un autre pour faire le LAN.

Voici ma config réseau (`/etc/config/network`)
```
config interface 'loopback'
        option device 'lo'
        option proto 'static'
        option ipaddr '127.0.0.1'
        option netmask '255.0.0.0'

config globals 'globals'
        option ula_prefix 'fd1b:8136:589b::/48'
        option packet_steering '1'

config device
        option name 'br-lan'
        option type 'bridge'
        list ports 'eth0'

config interface 'lan'
        option device 'br-lan'
        option proto 'static'
        option ipaddr '192.168.1.1'
        option netmask '255.255.255.0'
        option ip6assign '60'
        list dns '1.1.1.1'
        list dns '1.0.0.1'

config interface 'wan'
        option device 'eth1'
        option proto 'dhcp'
        option vendorid 'neufbox_NB6V-XXXXXXXXXXXXX'
        option peerdns '0'
        list dns '1.1.1.1'
        list dns '1.0.0.1'
```

Tout d'abord, on active `packet_steering` pour distribuer la charge logiciellement sur plusieurs cœurs car en pleine charge, un seul cœur n

'est pas assez rapide pour du WAN - LAN.
Ensuite, on assigne les ports RJ45 aux interfaces afin de pouvoir SSH sur la machine facilement car jusqu'à présent, je virtualisais l'OS avec QEMU en utilisant la clé USB comme disque (merci QEMU de pouvoir faire ça !) ce qui faisait que les modifications étaient appliquées directement dessus. Ça m'a notamment permis la configuration et l'installation de plusieurs paquets.
Enfin, le plus important je dirais : `vendorid`. Il permet de spoof le... vendorid... pour faire passer la nouvelle box, pour une box SFR auprès de l'ONT. D'après mes tests, `neufbox` dans le nom suffit pour qu'il accepte la box mais j'ai préféré respecter le même pattern que l'ancienne box avait.

Je boote le PC portable et après une minute, j'ai une IP sur le WAN ! Super, j'ai officiellement accès à internet et je peux `ping` avec succès.
## Dump AP
Que serait un réseau domestique sans Wi-Fi. Cette partie était assez simple que j'avais déjà flashé mon Xiaomi sous OpenWRT avec [l'exploit web](https://openwrt.org/toh/xiaomi/xiaomi_redmi_router_ac2100#method_aweb_exploit). J'ai dû reset le routeur histoire de repartir sur une base fraiche pour ensuite désactiver le DHCP car on délègue ça au PC portable maintenant.
Mais d'ailleurs, pourquoi "Dump AP" ?
Simplement car le routeur va juste récupérer le flux sans fil et le transmettre au PC, il ne fait rien niveau réseau, pas de firewall, pas de WAN-LAN, pas de transfert vers un autre sous-réseau, rien.
Son but est uniquement de diffuser un AP en 5 et 2.4GHz, le reste du travail est délégué à la box. Très pratique sachant qu'en WAN - LAN, le Xiaomi plafonne à environ 300Mbits et avec ce setup je peux saturer le débit sans soucis.

# Conclusion
Très fun. J'ai beaucoup appris sur le fonctionnement d'un ONT plus que sur une box grâce à ce que j'avais déjà fait avec ce système, mais ça n'en reste pas moins excitant de voir des LED vertes s'allumer et un `ping` qui arrive à passer !
Je tiens à remercier les forums de lafibre.info qui m'ont beaucoup aidé pour trouver ce que je voulais même si beaucoup des informations essentielles étaient "cachées".
J'ai fait le choix de ne pas faire le tour du setup SMB/DLNA car je trouve ça hors sujet car ça ne diffère pas vraiment d'un setup traditionnel, l'environnement en lui-même change mais ça reste globalement la même chose. Idem pour Docker.

Il ne me reste qu'à appeler SFR pour leur demander de me passer en IPv4 Full Stack et plus en CGNAT (car pénurie d'IPv4) car je pense que c'est pour ça que mes ports ne sont pas joignables depuis l'extérieur.

Merci à vous d'avoir pris du temps pour lire mon aventure.
