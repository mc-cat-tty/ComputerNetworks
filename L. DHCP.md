# Introduzione
Dynamic Host Configuration Protocol
https://datatracker.ietf.org/doc/html/rfc2131
# Obiettivi
1. implementare policy di rete da parte dell'amministratore
2. configurazione automatica dei dispositivi
3. scalabilità: deve funzionare bene per reti che scalano in dimensioni. Ma anche permettere la replicazione.

## Gestione degli indirizzi
Data una o più subnet al server DHCP:
- **assegnazione manuale**: il binding IP-host è contenuto del database.
- **assegnazione dinamica**: l'assegnazione delle policy può cambiare. Il server decide autonomamente come distribuire gli indirizzi.

# Protocollo
## Porte
#Attenzione abbiamo detto che all'apertura di una connessione TCP la porta in uso sul client è solitamente una porta *alta*, libera, non well-known. Per DHCP non è così.

Il **server** rimane in ascolto sulla porta **UDP 67**.
Il **client** rimane in ascolto sulla porta **UDP 68**.

## Pacchetto
```
+--------------------------+
| op | htype | hlen | hops |
|           xid            |
|           ...            |
|          ciaddr          |
|          yiaddr          |
|          siaddr          |
|           ...            |
|         chadddr          |
|           ...            |
|    options (variabile)   |
+--------------------------+
```

- op - operation code: 1 request, 2 reply
- xid - transaction ID, scelto casualmente dal client e inserito dal server nelle risposte
- ciaddr - client IP address. Usato in RENEW, BOUND e BINDING.
- yiaddr - your (client) IP address. Usato dal server per proporre un indirizzo al client.
- siaddr - (next) server IP address to use in bootstrap
- chaddr - client hardware address - per questioni di scalabilità. Utile quando il client vuole comunicare con un server su una rete differente (vedi relay agent), dato che il MAC a livello H2N andrebbe perso

Come fa il server a distinguere i vari host? usa lo xid. Vedi avanti

Il protocollo DHCP permette anche di configurare regole come route, che non sono indirizzi IP. Queste regole aggiuntive sono definite nel campo *options*.
### Operations
Il campo *op* definisce il tipo di pacchetto dal punto di vista della direzione. Ha un valore in base alla **direzione** del pacchetto:
- 1 da client a server
- 2 da server a client

## Messaggi
### Client to server
- DHCPDISCOVER: un client non configurato invia questo messaggio in broadcast; contatta il server per la prima volta
- DHCPREQUEST: un client richiede conferma per uno degli indirizzi che gli sono stati offerti
- DHCPDECLINE: un client rifiuta un indirizzo
- DHCPINFORM: un client già in possesso di un indirizzo IP chiede opzioni aggiuntive
- DHCPRELEASE: rilascio esplicito dell'indirizzp
### Server to client
- DHCPOFFER: proposta di un indirizzo
- DHCPACK: accettazione di un indirizzo richiesto dal client
- DHCPNACK: rifiuto di un indirizzo
## Workflow
### Caso comune
```
client                    server
  |     -DHCPDISCOVER->     |
  |     <-DHCPOFFER--       |
  |     --DHCPREQUEST->     |
  |    <--DHCPACK/NAK--     |
  |    [--DHCPDECLINE->]    |
  |                         |
  |                         |
```

1. DHCPDISCOVER (c->s - op=1) con transaction id (**xid**) random generato dal client:
	1. broadcast h2n
	2. dst IP = 255.255.255.255 (broadcast globale)
	3. src IP = 0.0.0.0.0
	4. src port = UDP 68
	5. dst port = UDP 67
2. DHCPOFFER (s->c - op=2): il server fa un'offerta al client
	1. src IP = IP del server
	2. dst IP = yiaddr, ovvero un indirizzo non ancora assegnato. Perché unicast? ha una funzionalità di test, serve a verificare se l'IP è già stato assegnato. Il server può testare in maniera differente in base alla configurazione:
		1. per reti differenti: ping
		2. per stessa rete: ARP request
3. DHCPREQUEST (c->s): il client conosce l'IP del server
	1. dst IP = broadcast, con siaddr a livello applicativo
	2. src IP = yiaddr
4. DHCPACK/DHCPNACK
	1. DHCPACK: OK - il server conferma il binding *client id + addr*
	2. DHCPNAK: KO - l'indirizzo è già stato occupato
5. \[DHCPDECLINE\]: KO - il client nel mentre esegue test sull'occupazione dell'indirizzo IP. L'indirizzo potrebbe essere già occupato, ma il server potrebbe non averlo scoperto a causa di regole di firewalling.
6. Tutto OK? test delle impostazioni, memorizzazione in cache con tempo di lease

#Nota esistono diverse varianti del protocollo, l'offerta potrebbe essere inviata in broadcast

Perché è necessario che il client invii la DHCP REQUEST al server? Su una rete possono essere attivi più server DHCP sovrapposti per questioni di affidabilità e ridondanza. Tra le tante offerte ricevute il client ne seleziona una, di cui fa richiesta al server.

>**Tempo di lease**: validità di assegnazione dell'indirizzo IP. A metà del tempo di lease viene reinviata una request. Se il server non riceve nessuna richiesta di rinnovo riutilizza l'indirizzo.

#Attenzione il **transaction id** dei pacchetti rimane lo stesso durante tutto il flusso. Serve al client per filtrare il flusso. Il server potrebbe usare il chaddress. È anche una mitigazione per problematiche di sicurezza: in presenza di un server malevolo il client riceve più offer - situazione di race condition - e accetta la prima ricevuta. Il campo xid evita spamming da parte di server malevoli. Esiste una best-practice nelle reti: il server DHCP deve avere bassa latenza.
## Altre tipologie dei messaggi
- DHCPRELEASE: il client libera l'indirizzo prima della disconnessione alla rete
- DHCPINFORM: il client richiede parametri di configurazione local che non sono stati immessi manualmente dall'amministratore. Viene usato quando, ad esempio, l'indirizzo IP è già noto all'host, da una precedente configurazione manuale.
# Configurazione
Configurazione di dhclient sul client: `/etc/dhcp/dhclient.conf`

Il file `/etc/dhcp/dhcpd.conf` è il file di configurazione del server. Le prime opzioni sono quelle con scope globale; si nota perché sono a livello 0:
- `option domain-name "domain.org"`
- `option domain-name-servers 192.168.1.1` Solitamente il router di casa distribuisce se stesso perché agisce da local nameserver

Blocco-direttiva subnet, che permette di specificare un pool/range di indirizzi:
```
subnet 192.168.0.0 netmask 255.255.255.0 {
	range 192.168.0.100 192.168.0.200;
	option subnet-mask 255.255.255.0;
	option routers 192.168.0.253;
}
```
Nell'esempio sopra un blocco è configurato dinamicamente, mentre i restanti indirizzi staticamente. La seconda direttiva imposta il default gateway degli host.

Altre direttive util:
- `option domain-name`
- `option domain-name-servers`
- `default-lease-time`
- `max-lease-time`

In `/var/log/syslog` accentra le informazioni di logging dei servizi

Avvio del servizio sul server (Internet Service Consortium):
```
service isc-dhcp-server start
```
#Nota questo avvio fallisce se la configurazione è errata

#Prova `ss -unlp` (UDP Not resolve Listen Process)

Client:
```bash
dhclient -i eth0  # Richiesta
dhclient -r eth0  # Release
```

#Prova a bloccare i ping con firewall per creare un'assegazione duplicata degli indirizzi1

## Configurazione statica degli host
```
host hostname {
 hardware ethernet 02:04:06:10:79:8c;
 fixed-address 192.168.0.1;
}
```

#Nota l'indirizzo deve appartenere a una subnet precedentemente definita, ma non importa che cada in un range dinamico precedentemente definito.

## Regole di routing statico
L'opzione per configurare mediante DHCP le regole di routing per subnet CIDR è stata standardizzata con RFC 3442, prima si usava il classful routing.

```
option classless-routes code 121 = array of unsigned integer 8;
```
#Nota il codice all'interno dello standard è 121

Mentre per i client Microsoft (diverso codice):
```
option classless-routes-ms code 248 = array of unsigned integer 8;
```

Le due direttive sopra agiscono come un define, così da poter usare nomi più comodi sotto.

È necessario inserire la route classless nel blocco `subnet`:
```bash
subnet ... {
	...
	# route add -net 192.168.1.0/23 gw 192.168.0.254 
	option classless-routes 23,192,168,1,192,168,0,254; 
}
```

#Attenzione non devi inserire TUTTI i byte della rete. Inserisci il numero di byte `ceil(CIDR/8)`

Il questa opzione vengono concatenate tutte le regole di routing. Questa regola sovrascrive l'opzione `option routers 192.168.0.253`

Da RFC3442 se si configura routing classless, le altre regole verranno ignorate, come quelle imposte dalla direttiva `routers`. Con la subnet mask `0` si invia la regole di default:
```
option classless-routes 0,<gw>
```

## Relay agent
#Attenzione se il server si trova su una rete differente il client non può raggiungerlo. Perché i pacchetti broadcast a livello IP non vengono inoltrati.

Versioni antecedenti a DHCP:
- RARP - non permette di avere client e server su reti differenti
- BOOTP (Bootstrap) - si trova ancora nell'ispezione del traffico
- DHCP - diretto successore di BOOTP

Per inoltrare traffico DHCP serve un **Relay Agent** sul router. Questo servizio riceve le discovery broadcast, le inoltra al server DHCP, conoscendo il suo IP.

### Configurazione
`/etc/default/isc-dhcp-relay`

Info minima: in SERVERS inserisco i server (inserisco IP) verso i quali inoltrare richieste DHCP
Posso anche definire le interfacce di ascolto. File di configurazione:
```
service isc-dhcp-relay start
```