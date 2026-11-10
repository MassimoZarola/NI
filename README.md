# Progetto Network Infrastructures

Laboratorio Kathara realizzato per l'Homework di Network Infrastructures A.A. 2025/2026. La topologia riproduce i router, i server, i PC e gli indirizzi indicati nel PDF `homework_25_26_v2.pdf`.

## Topologia e configurazione

Il laboratorio comprende 10 router (`r1`-`r10`), 3 server (`s1`-`s3`) e 3 PC (`pc1`-`pc3`). I collegamenti e le collision domain sono definiti in `lab.conf`:

- `pc1` è collegato alla LAN privata `192.168.1.0/24` tramite `r1`.
- `pc2`, `s2` e l'interfaccia LAN di `r9` condividono la rete privata `192.168.1.0/24`.
- `pc3`, `s3` e l'interfaccia LAN di `r10` condividono `192.168.88.0/24`.
- `r1`-`r4` formano la rete di routing statica a sinistra; `r5`-`r8` costituiscono il nucleo OSPF; `r9` e `r10` collegano le rispettive LAN al resto della topologia.
- `s1` è collegato a `r6` sulla rete `23.75.0.0/16`. La rete tra `r4` e `r5` è `193.201.28.0/23`; il collegamento tra `r7` e `r10` è `151.0.45.0/31`.

Gli indirizzi statici sono configurati nei file `*/etc/network/interfaces`; le rotte e i servizi di avvio sono definiti nei file `*.startup`.

## Funzionalità

- **DHCP:** `r1` e `r9` assegnano indirizzi ai client delle rispettive LAN `192.168.1.0/24`; il gateway distribuito è `192.168.1.1`.
- **Routing:** `r5`, `r6`, `r7` e `r8` eseguono OSPF nell'area `0.0.0.0`. Le rotte statiche sui router restanti consentono di raggiungere le reti e gli indirizzi non privati previsti dalla topologia. `r5` ha inoltre una rotta statica per `50.0.0.0/24`.
- **NAT e firewall:** `r1`, `r9` e `r10` applicano masquerading e regole di inoltro per le rispettive reti private. `r10` inoltra anche il TCP sulla porta `7070` verso il server OpenVPN su `s3`. `r4` consente il traffico inoltrato SSH e blocca gli altri inoltri: dopo l'attivazione di questa regola, la connessione VPN diretta non è più il percorso previsto.
- **SSH:** `s1` avvia il server SSH e crea l'utente `ubuntu`. È configurata l'autenticazione con chiave pubblica da `pc1`.
- **Certificati:** sono presenti la CA condivisa (`shared/ca.crt`), il certificato e la chiave del server su `s3` e quelli dei client su `pc1` e `pc2`. Il server presenta i client autorizzati tramite i file in `s3/root/ccd/`.
- **OpenVPN:** la configurazione server è in `s3/root/server.conf`; le configurazioni client sono in `pc1/root/client.conf` e `pc2/root/client.conf`. VPN TCP sulla porta `7070`, rete `10.10.0.0/24`, comunicazione client-to-client, route verso `192.168.88.0/24` distribuita ai client e indirizzi VPN statici per i client (`10.10.0.100` e `10.10.0.200`). OpenVPN non viene avviato automaticamente.
- **Tunnel SSH:** `commands.txt` contiene il comando per inoltrare la porta locale `7070` di `pc1` attraverso `s1` verso il servizio OpenVPN. La configurazione OpenVPN di `pc1` usa `127.0.0.1:7070`, così il client può raggiungere la VPN passando dal tunnel SSH.
- **Catture:** `shared/capture_1.pcap`, `shared/capture_2.pcap` e `shared/capture_3.pcap` contengono le catture previste dai punti 10, 11 e 14 del PDF. Le catture e il traffico di prova non vengono generati all'avvio.

## Requisiti

- Docker installato e avviato.
- Kathara installato sull'host.
- Il PDF e i file del laboratorio devono restare nella stessa cartella; avviare i comandi Kathara dalla directory `NI`.

## Avvio e arresto

Da un terminale, entrare nella cartella `NI` e avviare il laboratorio:

```bash
cd NI
kathara lstart
```

Per verificare i nodi avviati:

```bash
kathara linfo
```

Per aprire una shell in un nodo, per esempio `pc1`:

```bash
kathara connect pc1
```

Sostituire `pc1` con il nome del nodo desiderato. Per fermare il laboratorio:

```bash
kathara lstop
```

Per arrestarlo e rimuovere i container:

```bash
kathara lclean
```

## Avviare manualmente la VPN

Dopo `kathara lstart`, avviare separatamente i processi OpenVPN: non sono lanciati dagli script di startup.

1. Aprire una shell su `s3` e avviare il server:

   ```bash
   openvpn --config /root/server.conf
   ```

2. Aprire una shell su `pc1` e avviare il port forwarding SSH usando il comando già riportato in `commands.txt`. Il comando resta in background grazie alle opzioni `-N -f`.
3. Avviare il client VPN in `pc1`:

   ```bash
   openvpn --config /root/client.conf
   ```

4. Avviare il client su `pc2`:

   ```bash
   openvpn --config /root/client.conf
   ```

   In `pc2`, `remote` punta direttamente all'indirizzo pubblico di `r10`; l'inoltro TCP su `r10` porta la connessione a `s3`.

Per chiudere il tunnel SSH, individuare e terminare il relativo processo `ssh` su `pc1`; terminare analogamente i processi OpenVPN sui nodi client e server prima di arrestare il laboratorio.

## Prove e catture manuali

Le prove seguenti sono manuali, non vengono eseguite dagli startup. Avviare i processi VPN necessari prima di generare traffico.

- **`capture_1.pcap`:** aprire un listener TCP sulla porta `8080` su `pc1`, collegarsi da `pc2` all'indirizzo VPN `10.10.0.100:8080` e acquisire il traffico VPN su un'interfaccia di `r1` con `tcpdump -w /shared/capture_1.pcap`.
- **`capture_2.pcap`:** fare ping da `pc1` a `pc3`. `pc3` ha la rotta statica `10.10.0.0/24` via `192.168.88.100`; acquisire il traffico sull'interfaccia appropriata di `s3` e salvarlo in `/shared/capture_2.pcap`.
- **`capture_3.pcap`:** dopo aver attivato il firewall di `r4`, stabilire la VPN di `pc1` tramite il port forwarding SSH su `s1`, ripetere la prova TCP tra `pc1` e `pc2` e salvare la cattura in `/shared/capture_3.pcap`.

Per esempio, una cattura avviata manualmente da un nodo può usare la forma:

```bash
tcpdump -i <interfaccia> -w /shared/capture_1.pcap
```

Sostituire `<interfaccia>` con quella effettivamente interessata al traffico; interrompere `tcpdump` con `Ctrl+C` al termine.

## File principali

- `lab.conf`: nodi, interfacce e collision domain Kathara.
- `*.startup`: comandi eseguiti all'avvio dei nodi.
- `*/etc/network/interfaces`: indirizzi e configurazione delle interfacce.
- `r1/etc/dhcp/dhcpd.conf`, `r9/etc/dhcp/dhcpd.conf`: pool DHCP.
- `r5`-`r8` `etc/quagga/`: configurazione OSPF.
- `s3/root/server.conf`, `pc1/root/client.conf`, `pc2/root/client.conf`: configurazioni OpenVPN.
- `commands.txt`: comando per il port forwarding SSH.
- `shared/`: certificato CA e file pcap.

