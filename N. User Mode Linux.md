# Introduzione
> È un sistema di virtualizzazione che consente di eseguire il kernel Linux come un processo applicativo, creando un ambiente isolato, protetto e controllabile/ispezionabile. Permette di eseguire più sistemi operativi virtuali Linux-based su un unico host come se fossero normali processi (all'interno dello **user space**): un kernel Linux guest, per essere avviato come applicativo all'interno di un altro kernel Linux host, deve essere compilato per l'architettura _um_

#Vedi https://user-mode-linux.sourceforge.net

UML nace nel '99 per mano di Jeff Dike. È parte del kernel a partire dalla versione 2.6

## Funzionamento
UML aggiunge un livello software tra il processo lanciato in ambiente virtualizzato e il kernel realmente a contatto con l'hardware. Espone risorse virtuali, rimanendo un normale processo applicativo per il kernel host.

È interessante come sia un kernel **full-featured**: sono stati cambiati nella sorgente del kernel pochi aspetti architecture-dependent

## Risorse
UML si presenta con diversi eseguibili: linux, uml_moo (merge di file COW - Copy On Write), uml_mkcow, uml_switch (per la gestione della rete virtuale) e uml_console (per la gestione delle macchine dall'esterno)

È infine necessario procurarsi l'immagine di un fs funzionante.

#Vedi **VDE** - Virtual Distributed Ethernet - che è costruito su UML

## Parametri di avvio linux
Per visualizzare i parametri di boot del kernel:
```bash
cat /proc/cmdline
# Oppure
sysctl -a
```

Per lanciare il kernel con opzioni:
```bash
linux [opzioni-kernel] [opzioni-init]
```

### Disco
```
ubd<N>=[<cow_file>,]<image_file>
```

Con N=0 si indica il fs root. È necessario almeno un disco per avviare un host.

#Nota UBD - Userspace Block Device

### Interfaccia
```
eth<N>=daemon,,unix,<socket.ctl>
# Oppure
eth<N>=vde,<socket_dir>[,[<mac_addr>][,<port_number>]]
```

Dove:
- `vde`: usiamo lo stack vde per comunicare con `vde_switch`
- `daemon`: comunichiamo con `uml_switch`

Per collegare eth0 alla porta 5 dello switch vde in /tmp/s1: `eth0=vde,/tmp/sw1,,5`
### Misc
- `-mem=<MB>`
- `-root=<root_partition>`
- `-devfs=mount` abilita il `dev` filesystem

#Attenzione non sono permessi spazi tra i sottoparametri. Inoltre, terminare gracefully il processo UML con `halt` o `shutdown`

## VDE Switch
Per avviare il VDE switch: `vde_switch -s <socket_dir>`
Per far sì che l'host di prima si connetta allo switch1: `vde_switch -s /tmp/sw1`

### Parametri
Opzioni:
- `-x, -hub` disabilita il motore di switching
- `-f <path>` specifica un file di configurazione
- `-d` background, da usare con `-M`
- `-M <path>` crea un socket con cui connettersi allo switch tramite `vdeterm`

# Creazione di un disco
```bash
dd if=/dev/zero of=fs.ext4 bs=1 count=1 seek=300M
```

- bs - block size - ulteriormente declinato in ibs e obs (input block size e output block size)
- count - copia un solo byte
- seek - skip di 300MB dall'inizio del file di output

Viene creato un file sparso con dimensione ~300MB, visualizzabile con `ls -lsh fs.ext4`. I dati realmente scritti sul disco sono grandi un solo Byte: `ls -sh fs.ext4` o con `du -h fs.ext4`

```bash
mkfs.ext4 fs.ext4  # Crea il filesystem
```

## Utility
```bash
lsblk
mount
blkid
```

# Creazione di una rete L2
Per testare la connettività a livello H2N, senza necessità del livello superiore:
```bash
arping -0Bi eth0
```
Per inviare richieste di risoluzione di 255.255.255.255

```bash
arping -0i eth0 1.1.1.1
```
Per inviare richieste di risoluzione di 1.1.1.1

In entrambi i casi `0` significa che IP SRC di questi messaggi (`tell ...`) è 0.0.0.0

# Copy On Write
>Come facciamo a disegnare i baffi alla Gioconda senza andare in prigione? li disegnamo su un telo e lo posizioniamo davanti. Jeff Dike

UBD = COW file + backing file

Il backing file viene usato in modalità read-only, le differenze memorizzate sul file COW.
Il vantaggio è che un'immagine può essere condivisa tra più sistemi virtuali; inoltre la parte più corposa dell'immagine, ovvero quella comune, non spreca spazio aggiuntivo sul disco host.

I file COW sono **scattered**, ovvero sparsi.

#Attenzione il backing file deve essere più vecchio del file COW

A livello concettuale, le letture avvengono sul segmento più nuovo tra i due file, mentre le scritture avvengono sempre sul file COW.

`uml_mkcow` crea una nuova immagine COW
`uml_moo` crea una nuova immagine partendo da file COW e backing: `uml_moo <file.cow> <new_img.ext4>`. Di fatto esegue una funzione di sincronizzazione e applicazione delle modifiche tra i file.

Il file COW referenzia il file di backing, non è necessario specificarlo.

## Backup
Quando si creano archivi contenente file immagine e file COW è necessario mantenere la consistenza tra i timestamp dei due file.

Per preservare le date di accesso:
- `-p`preserve o `-a` archive con il comando `cp`
- `--atime-preserve` con il comando `tar`

Per preservare la sparsità dei file:
- `--sparse=always` con `cp`
- `-S` con `tar`

