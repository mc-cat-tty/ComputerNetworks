# Storia
IPv6 world day, l'anno successivo adozione (eg. implementazione in Linux kernel).

L'RFC di riferimento è ora 8200

# Dettagli
IPv6 non è un *drop-in replacement* per IPv4. Serve una transizione, in quanto non sono interoperabili.
## Vantaggi
- huge addressing space
- minimizza il processing di routing
- particolare attenzione ai dispositivi mobili e roaming
- security by design -> la lenta transizione ha permesso il porting di IPSec
- abbastanza indirizzi perché tutti i dispositivi abbiano un IP pubblico

_Fanalino di coda_

## Notazione
Dimensione: 128 bit, in notazione esadecimale. Ogni coppia di byte divisa da ":"
Compressione: rimuovere leading zeros, sostituire la sequenza di zeri più lunga con "::"

### Undefined address
```
::/128
```

Viene assegnato agli host prima che acquisiscano un indirizzo IPv6

### Loopback address
```
::1/128
```

#Nota in IPv4 è una subnet, qui un singolo indirizzo

### Global
```
2000::/3
```

### Link-local
> Hanno scope interno alla LAN

```
fe80::/10
```

Tutti gli indirizzi all'interno di questo spazio di indirizzamento sono usati da interfacce locali alle LAN. Per uscire sui Internet servono quindi sia un indirizzo link-local che globale per le interfacce che vogliono uscire.

### Private addresses
Esistono, ma tipicamente non vengono utilizzati

## Scope
>Gli indirizzi unicast possono avere scope link-local o globale. Lo scope globale li rende routabili in Internet.

Vengono inoltre rimossi gli indirizzi broadcast. Rimangono multicast e unicast.

## Subnetting
I primi 8 byte sono detti *network prefix*, gli ultimi 8 sono di *interface ID*.

Esistono subnet mask consigliate. RIPE usa tipicamente spazi /32. I clienti finali si prendono solitamente spazi /56.

Partizionando lo spazio in subnet /64, ottengo 256 possibili subnet da 2^64 host l'una.
## Gerarchia
IANA -> RIR -> LIR -> End user

Come in IPv4 si dice che un prefisso è delegato al cliente finale dall'ISP

# Header
Header fisso a 40 byte.

Campi:
- version [4b] -> contiene 6
- traffic class [8b] -> priorità e controllo di congestione
- flow label [20b] -> 
- payload length [16b] -> lunghezza del payload, escluso l'header stesso
- next header [8b] -> ?
- hop limit [8b] -> equivalente del TTL (Time To Live)
- src addr [128b]
- dst addr [128b]
- no checksum -> rimosso in quanto già integrato in TCP e UDP (necessario)

La frammentazione non è più inclusa nell'header IP: vedi dopo.

Daisy-chained headers.

## Frammentazione
>I router non fanno frammentazione dei pacchetti IPv6. Inviano un pacchetto ICMPv6 too big al mittente, che reinvierà il pacchetto con una dimensione appropriata.

Si delega al mittente il compito di frammentare il pacchetto.

## Path MTU discovery
Ricevendo progressivamente _packet too big_ dai router scopro il path MTU.

## Broadcast
Svantaggi individuati: broadcast storm, risvegliare tutti i nodi, ecc.

Multicast "all nodes" address:
```
ff02::1/128
```

Multicast "all routers" address:
```
ff02::2/128
```
# Neighbour Discovery Protocol
Non esiste ARP, rimpiazzato con NDP, che fa uso di ICMPv6 e usa multicast.

Utilizzi:
- risoluzione di indirizzi link-layer
- trovare router neigh
- adv public ip change
- change route

## Neigh solicitation and advertisement
Scenario: un host H1 vuole scoprire il MAC di un altro, conoscendo il suo IPv6

H1 crea il **solicited-node multicast address**: #Vedi regole
H1 crea il **solicited-node multicast MAC address** #Vedi 

H2 risponde con un pacchetto di advertisement.

## Router solicitation and advertisement
Tipi 133 e 134

Come fa un host a conoscere il suo prefisso? ricordando che ha un indirizzo pubblico

# Autoconfigurazione
Ogni host ha bisogno di un indirizzo link-local e un indirizzo globale.

1. creazione dell'indirizzo link-local: prefisso `fe80` e interface ID generato dal MAC address
2. link-local address è nello stato *candidate*, potrebbero esistere duplicati.
3. tramite neigh solicitation con dst IPv6 auto-assegnato si scopre se un host nella rete lo possiede già. Se nessuno risponde l'indirizzo diventa definitivamente quello dell'host.
4. come generare la il giusto prefisso di rete? si usa router solicitation da host a router
5. il router risponde con un router advertisement che fornisce network prefix, info sul suo indirizzo e può forzare l'interface id dell'host
6. Ora serve creare un global unicast address (vedi SLAAC - StateLess Address AutoConfiguration) che, come in IPv4, può essere assegnato staticamente dall'amministratore di rete.

## Generazione di interface ID
- Opaque IID: generazione randomica dell'interface ID -> risolve problemi di profilazione
- Generazione a partire dal MAC address
- DHCPv6

## Note
Un interfaccia ha molteplici indirizzi:
- link local
- più di un indirizzo globale (ricevuto da DHCP, calcolato con SLAAC)

# DNS, URL et al
## URL
Hanno forma: `https://[fe80:xxxx:xxxx::]:8080/resource.html`

Il wrap nelle parentesi quadre serve per evitare ambiguità con i ":" del numero di porta.
## DNS
Il campo AAAA memorizza l'indirizzo IPv6 associato ad un host

## Transizione
I due stack sono ininteroperabili. Si usa la tecnica del **dual stack**

ISP delegano a un cliente un prefisso IPv6 e un indirizzo IPv4.

Una soluzione per l'interoperabilità tra core network dell'ISP e reti finali (domestiche, aziendali, accademiche, ecc.) è l'incapsulamento di un pacchetto IPvX nel payload di un pacchetto IPvY.

## Happy eyeballs
Definisce come gli OS si devono comportare nel decidere quando usare IPv4 o IPv6.

# Configurazione
```
auto eth0
iface eth0 inet6 auto  # auto = configurazione automatica indirizzo
iface eth0 inet6 static
	address 2001::1/56
```

#Nota multicast listener report message -> host si mette in ascolto su indirizzi multicast per ricevere l'advertisement da parte di un neigh

#Nota DAD - Duplicate Address Detection

```
ping6 2001::1
```

#Nota `netcat` non supporta IPv6:
```
ncat -6vlp 8080  # H2
ncat -6 2001::2  # H1
```

Ping su indirizzi link-local:
```
ping6 fe80::xxxx%eth0
```
#Nota è necessario specificare l'interfaccia di destinazione perchè di default il pacchetto esce con indirizzo globale.

Se in `/etc/network/interfaces` non si specifica un indirizzo link-local, ma solamente uno globale, il primo viene assegnato automaticamente con l'autoconfigurazione.

Forwarding IPv6:
```
sysctl -w net.ipv6.conf.all.forwarding=1
```

# Approfondimenti
IPv6 broker tunneling

Un servizio supporta IPv6?
```
dig AAAA +short github.com
```

# Note
mattia.trabutto@unimore.it