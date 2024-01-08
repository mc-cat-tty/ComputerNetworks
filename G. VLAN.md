Lezione interlacciata con [[10. VLAN]].

# Indirizzi IP mutlipli
## Approccio moderno
Su sistemi moderni si usano comandi e file di configurazione che supportano nativamente più indirizzi IP.

Per assegnare indirizzi ip multipli alla stessa if:
```bash
auto eth0
iface eth0 inet static
 address XXXX/XX
 post-up ip addr add XXXX/XX dev XXX
```

Anche la seguente soluzione è valida:
```bash
auto eth0
iface eth0 inet static
 address XXXX/XX

iface eth0 inet static
 address YYYY/YY
```

## Approccio storico
```bash
auto eth0

iface eth0 inet static
 address XXXX/XX

iface eth0:0 inet static
 address XXXX/XX
```

Con `eth0:0` viene creato un alias dell'interfaccia di rete 0, con tutte le conseguenze.

Con `ip` vedo gli indirizzi multipli; mentre `ifconfig` ha bisogno che venga specificato l'alias.

#Attenzione alla differenza tra alias e interfacce di rete virtuali (come bridge o VLAN).
# Scoperta dell'assenza delle VLAN sulla rete
Vedendo una rete senza rotte verso una rete, eseguendo `arping`, ci aspetteremmo un timeout del comando. Nel caso della configurazione senza VLAN non è così.

Le varie reti possono comunque comunicare a LV2.

Supponendo di essere *192.168.1.1* e di avere la regola *192.168.2.0/24 via 192.168.1.254*
Un altro metodo per accorgermene è: `ping 192.168.2.1`. Inizialmente i pacchetti passano per il router. Poi avviene un *Host Redirect*: viene consigliata la destinazione stessa come migliore rotta per raggiungerla. Lo posso vedere nelle cache del SO con `ip route`. Con `traceroute` noto che il dispositivo terminale viene raggiunto direttamente.

# Progettazione di VLAN
Consiglio progettuale: non associare informazioni concettuali e tecniche/implementative. Ad esempio:
- LAN 1 - VLAN 10
- LAN 2 - VLAN 20
- ...

```
LAN1 - 192.168.1.0/24 -> VLAN10
LAN2 - 192.168.2.0/24 -> VLAN20

VLAN10
 H1.eth0       - Access Link 10 - Port 2 -   Switch S1
 ROUTER.eth0   - Trunk Link 10  - Port 1 -   Switch S1
 // potrei scegliere Hybrid (con VLAN di default)

VLAN20
 H2.eth0       - Access Link 20 - Port 3 -   Switch S1
 ROUTER.eth0   - Trunk Link 20  - Port 1 -   Switch S1
```

Altre scelte progettuali ammissibili:
- (seppur meno raffinata) trunk anche per H1 e H2
- (più arzigogolata) hybrid link con tag di default per Port 1

Scelte non ammissibili:
- due access link su Port 1

# Configurazione VLAN
## Switch: VDE console
```vde
vlan/create 10
vlan/create 20
port/setvlan 2 10
port/setvlan 3 20
vlan/addport 10 1
vlan/addport 20 1
```

Per una configurazione permanente serve scrivere la sequenza di comandi in *Startup configuration*.
## Router: Linux
Molto simile se veniamo dall'alias (configurazione permanente):
```
# Trunk
auto eth0.10 eth0.20

iface eth0.10 inet static
 address XXXX/XX

iface eth0.20 inet static
 address XXXX/XX
```

Oppure per link ibrido:
```
# Hybrid
auto eth0 eth0.20

# Access link su if eth0
iface eth0 inet static
 address XXXX/XX

iface eth0.20 inet static
 address XXXX/XX
```

Configurazione temporanea manuale:
```bash
ip link add link <phyif> <virtif> type vlan id <id>  # Creazione dell'if virt
ip link del <virtif>  # Rimozione
```

#Attenzione il nome di virtif e identificativo non sono rigidamente legati tra loro. Per recuperare il VID a partire dal nome dell'interfaccia: ``grep VID /proc/net/vlan/<virtif>
# Verifica funzionamento
#Nota non ha senso dire che *la rete funziona* -> le VLAN riguardano un requisito non-funzionale

Le VLAN operano a LV2 quindi è necessario verificare il loro funzionamento mediante `arping`

## Considerazioni
Cosa succede se mi metto in ascolto?
```bash
tcpdump -eni eth0.10
```

Non noto che i frame sono taggati. L'interfaccia virtuale associata alla VLAN nasconde la complessità.

Posso notarlo mettendomi in ascolto su eth0: `tcpdump -eni eth0`

Cosa succede se faccio un arping sull'interfaccia "aggregata"?
```bash
arping -0i eth0
```
Dove `-0` serve per evitare di specificare un indirizzo sorgente