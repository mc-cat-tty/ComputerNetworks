Lezione interlacciata con [[7. ICMP]]
# Cattura
1. seleziona una *capture interface*, equivalente all'argomento `-i` passato a tcpdump
2. filtro di cattura (*capture filter*)
3. filtro di visualizzazione (*display filter*), ad esempio `icmp.type == 0 || icmp.type == 8`. È come eseguire una query sulla lista di pacchetti arrivati a Wireshark

#Nota che i filtri di cattura non possono essere modificati a tempo di esecuzione, mentre quelli di visualizzazione possono essere modificati al volo, ma richiedono necessariamente più risorse per essere applicati. I filtri di cattura vengono richiesti al kernel, che evita così di copiare pacchetti inutilmente da kernelspace a userspace.

#Vedi https://en.wikipedia.org/wiki/Berkeley_Packet_Filter

È possibile catturare traffico con `tcpdump` salvandolo su file pcap, passando il nome del file all'argomento `-W`

# Visualizzazione
È molto ISO-OSI oriented. Permette una visualizzazione a livelli.

#Nota Marionnet permette di montare directory condivise. Questa directory è solitamente montata sui sistemi guest in `/mnt/hostfs`; puoi scoprirlo con `mount`. Sul sistema host puoi trovare la directory con `cat /proc/cmdline`, alla voce `hostfs`.

#Vedi scripting su file pcap in Python con `libpcap`

# Berkeley packet filter
È un linguaggio comune a tcpdump e Wireshark per esprimere filtri di cattura, NON di visualizzazione.
