# iproute2
```bash
ip n flush all
```

```bash
ip addr add 192.168.1.1 dev eth0
```

`ip` è un tool più moderno: struttura a predicati/verbi/sottocomandi.

Se voglio eliminare l'indirizzo: `ip addr del 192.168.1.1/32 dev eth0`

#Attenzione se la netmask non è specificata usa di default `/32` -> comportamento diverso da `ifconfig` che usa la notazione classful di default.

#Attenzione il comportamento di default non prevede l'accensione a livello fisico dell'interfaccia:
```
ip link set dev eth0 up
```
Il comando `link` permette di lavorare a basso livello, sul livello 2.

Stato:
```
ip addr show dev eth0
ip link show dev eth0
```
I due livelli sono nettamente separati. Il secondo comando mostra l'indirizzo MAC, il primo no.

# Tabella di routing
Sono consultabili con `route -n` o con il più nuovo:
```
ip route
```

Per ogni interfaccia con indirizzo IP viene creata automaticamente una entry con gateway *0.0.0.0* (formato `route`) che è un indirizzo IP non valido, quindi sta ad indicare che la rete è automaticamente raggiungibile. `ip route` ci indica la stessa cosa con la nomenclatura `scope link` (è all'interno di uno scope raggiungibile mediante protocollo h2n); questo comando ci indica anche con quale indirizzo IP sorgente escono i pacchetti dall'interfaccia, utile nel caso in cui **una interfaccia di rete abbia più di un indirizzo IP**. È un tool più nuovo avvezzo all'uso di IPv6, in cui avere indirizzi IP multipli sull'interfaccia è la norma.

Cosa succede se scordo la netmask in ifconfig? viene usata la notazione classful. Verifica con la tabella di routing (`route` mostra dotted notation, mentre `ip route` mostra la notazione CIDR)

# Reti irraggiungibili
#Nota differenza tra *Network unreachable* e *Host unreachable*. Nel primo caso non genero neanche traffico; nel secondo vanno in timeout le richieste ARP.

Errori di livello IP:
- **unknown network** -> non ho default gateway
- **unreachable first hop** -> non riesco a risolvere l'indirizzo IP del default gateway
- **unreachable network** -> contatto il router ma non riesco a raggiungere la rete a cui appartiene l'host che vorrei contattare

# Routing su Linux
Un nodo intermedio scarta di base i pacchetti, se l'indirizzo IP di destinazione non è il proprio. Per cambiare questo comportamento si veda sotto.

Per ottenere lo stato:
```bash
sysctl net.ipv4.ip_forward
```

Per scrivere un valore (alla stregua di scrivere su procfs, non è permanente):
```bash
sysctl -w net.ipv4.ip_forward=1
```

Per rendere l'opzione permanente si modifica il file `/etc/sysctl.conf`; si deve aggiornare la configurazione runtime: `sysctl -p`.

Su H1:
```bash
route add -net 192.168.2.0/24 gw 192.168.1.254
ip route add 192.168.2.0/24 via 192.168.1.254
```

Simmetrico su H2.

## Troubleshooting
Cosa succede se provo a pingare un dispositivo non configurato o non presente sulla rete di destinazione?
- **Host destinazione non configurato:** Finché non configuro il routing su H2 non riceverò mai una risposta se provo a pingarlo da H1. l'ICMP non prevede un timeout sul nodo mittente.
- **Host destinazione non presente**: Il router non riesce a risolvere l'indirizzo MAC. L'ARP discovery fallsice, invia lo stato di errore mediante il protocollo ICMP (grazie a un campo embeddato nel protocollo IP). #Vedi protocolli di segnalazione di IP

#Nota la differenza tra **host unreachable** *from* host sorgente e *from* router. Nel primo caso non riesco a trovare la destinazione a causa di una ARP request scaduta sul MIO dispositivo. Nel secondo caso non riesco a trovare la destinazione a causa di una ARP request inviata DAL ROUTER.

#Nota la differenza tra **network unreachable** *from* host locale e *from* router.
- `Network unreachable`: non conosco una route per raggiungere la rete a partire dal mio host
- `Destination net unreachable`:  non conosco una route per raggiungere la rete sul router. Prova ad esempio ad inserire `route add -net 192.168.4.0/24 gw 192.168.1.254` avendo cura di non collegare la subnet `*.4.0` al router

#Consiglio quando non si riceve nessun errore guarda l'altro capo della rete

## Tabella di routing permanente
Si usano le direttive:
- `pre-up`
- `pre-down` 
- `post-up` o l'alias `up`
- `post-down` o l'alias `down`

Queste direttive indicano quali comandi eseguire nel momento specificato:
```
post-up route add -net 192.168.2.0/24 gw 192.168.1.254
```

## Regole di routing sugli host
Esistono due argomenti passabili a `route`:
- `-net` usata se non esistono regole più specifiche, come sotto
- `-host` permette di specificare la route per un host specifico

`... -net default gw ...` specifica il default gateway
## Default gateway
```bash
route add default gw ADDR/CIDR
```

## Indirizzi /32
Gli indirizzi ip con subnet mask 255.255.255.255 sono assegnati a host isolati, quando non si vogliono assegnare indirizzi contigui a quest'ultimo.

# Real World Access
Il RWA - Real World Access - su Marionnet ci consente di connettere un host a Internet.
Offre DHCP e DNS. Se configurato come default gateway per il dispositivo che vuole uscire su Internet, fornisce connettività.

È considerato un router.

Acquisizione dinamica IP attraverso protocollo DHCP:
```bash
dhclient -i eth0
```

Ora posso lanciare:
```bash
wget --no-check-certificate www.unimore.it
```

#Nota che è un collegamento di livello 2
#Attenzione corrette regole di routing

# Cicli
In presenza di cicli il router che scarta il pacchetto lancia l'errore `Time to live exceeded`.

## IP Cal e IP route get
```bash
ip route get ADDRESS
```

Risponde con la rotta che utilizza per raggiungere l'indirizzo richiesto.