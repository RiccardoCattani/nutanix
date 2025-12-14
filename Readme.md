📌 1. IL DATA CENTER TRADIZIONALE: DAS (Direct Attached Storage)

# Com’era il datacenter all’inizio
- Ogni server (“nodo”) aveva i suoi dischi locali
- L’applicazione girava direttamente sul sistema operativo installato sul server.
- Nessuna virtualizzazione, nessuna condivisione delle risorse.

# Vantaggi
- Architettura semplice.
- Non serve hardware esterno.

# Svantaggi
- Scarsa scalabilità → se un server è pieno, devi comprarne un altro.
- Bassa disponibilità → se il server si guasta, l'applicazione muore.
- Nessuna condivisione dello storage → non puoi fare HA, live migration, ecc.

📌 2. ARRIVA LO STORAGE CONDIVISO: SAN / NAS

Per abilitare l’alta disponibilità e la virtualizzazione, si introduce lo storage condiviso.

# Cosa cambia
- I server non usano più i propri dischi locali.
- Tutti si collegano a un sistema di storage centralizzato tramite rete SAN/NAS.
- Lo storage ha i suoi controller, che gestiscono tutti gli I/O.

# Vantaggi
- Possiamo usare gli hypervisor (ESXi, Hyper-V, AHV).
- Possiamo creare molte VM usando gli stessi server fisici.
- Possiamo fare HA, vMotion/Live Migration, DRS, ecc.

# Svantaggi
- Ma nascono nuovi problemi
- Il controller dello storage diventa un collo di bottiglia → Più VM = più IOPS richiesti = peggiori prestazioni.
- Aumentare le prestazioni è molto costoso → Lo storage è un sistema monolitico, difficile da scalare.
- Scalabilità disallineata → CPU e IOPS non crescono insieme, creando inefficienza.
- Se lo storage ha problemi, si ferma tutto il datacenter.

> [!NOTE]
> **Approfondimento: Perché il Controller è il collo di bottiglia? (Dischi vs Controller)**
> Spesso si pensa che le prestazioni dipendano solo dalla velocità dei dischi (es. SSD vs HDD), ma in una SAN il vero limite è il **Controller**.
>
> **Il Ruolo della CPU del Controller (SAN):**
> In una SAN, la CPU del controller deve fare tutto il lavoro pesante per l'intero datacenter:
> 1.  **Calcolo RAID (XOR):** Calcoli matematici complessi per ogni scrittura (parità).
> 2.  **Mapping & Caching:** Decidere dove mettere ogni blocco e cosa tenere in RAM.
> 3.  **Data Services:** Deduplica e compressione in tempo reale.
>
> **L'Analogia del Casello:**
> Immagina un casello autostradale (il Controller) con solo 2 porte aperte:
> - L'autostrada dopo il casello (i Dischi) può essere velocissima e vuota.
> - Ma se arrivano 1.000 auto contemporaneamente (le VM), si crea una coda chilometrica *prima* del casello.
>
> **Differenza Chiave:**
> - **SAN:** La CPU del server (Host) fa girare le VM, ma delega tutto il lavoro storage al Controller SAN (che è fisso e limitato).
> - **Nutanix:** La CPU del server fa girare le VM **E** (tramite la CVM) fa anche il lavoro del controller storage. Aggiungi un server -> aggiungi potenza di calcolo per lo storage.

**In sintesi: Chi lavora per le prestazioni?**

| Attività | Chi lo fa nella SAN? | Chi lo fa in Nutanix? |
| :--- | :--- | :--- |
| **Eseguire VM** | CPU del Server | CPU del Server |
| **Calcolare il RAID/Resilienza** | **CPU del Controller SAN** (Collo di bottiglia) | CPU del Server (tramite CVM) |
| **Comprimere i dati** | **CPU del Controller SAN** | CPU del Server (tramite CVM) |
| **Gestire la coda I/O** | **CPU del Controller SAN** | CPU del Server (tramite CVM) |


> [!TIP]
> **Approfondimento Tecnico: CPU del Controller SAN vs CPU del Controller del Disco**
>
> Sì, sono due cose completamente diverse e lavorano a livelli gerarchici differenti. Ecco la distinzione tecnica:
>
> **1. CPU del Controller SAN (Il "Generale")**
> È il processore principale dello Storage Array (es. un EMC Unity, un NetApp, un HPE 3PAR).
> - **Dov'è:** Nel "cassetto" principale dello storage (la testa pensante).
> - **Potenza:** È una CPU potente (spesso Intel Xeon), simile a quella di un server.
> - **Cosa fa:** Gestisce la logica complessa: RAID, deduplica, compressione, snapshot, gestione della cache, presentazione delle LUN ai server.
> - **Visibilità:** Vede tutti i dischi e decide dove mandare i dati.
>
> **2. CPU del Controller del Disco (Il "Soldato")**
> È un piccolo chip integrato direttamente sulla scheda elettronica (PCB) di ogni singolo Hard Disk o SSD.
> - **Dov'è:** Avvitato sotto ogni singolo disco rigido.
> - **Potenza:** È un processore embedded molto semplice e specializzato (es. ARM).
> - **Cosa fa:** Esegue gli ordini fisici: "sposta la testina al settore 500", "leggi il voltaggio della cella di memoria", "gestisci i settori danneggiati", "controlla la velocità di rotazione".
> - **Visibilità:** Vede solo il suo disco e non sa nulla del RAID o degli altri dischi.
>
> **L'Analogia:**
> - **Controller SAN:** È il **Capo Cantiere**. Ha il progetto in mano, sa che bisogna costruire un muro (il volume RAID) e urla gli ordini: "Voi 5, mettete i mattoni qui!".
> - **Controller del Disco:** È il **Singolo Muratore**. Non sa cosa sia l'edificio finale, sa solo che deve prendere quel mattone e metterlo esattamente in quel punto, assicurandosi che la malta tenga.

📌 3. ARRIVA LA VIRTUALIZZAZIONE (Hypervisor)

Con la virtualizzazione puoi creare molte VM su un singolo server (es. 1 server = 10, 20, 30 VM).

# Ma cosa succede allo storage?

Le VM aumentano molto più velocemente rispetto agli IOPS che lo storage può fornire.

Risultato:

⚠️ le prestazioni calano anche se i server hanno CPU e RAM a sufficienza.

La causa è lo storage condiviso che non scala.

📌 4. COS'È L'INFRASTRUTTURA IPERCONVERGENTE (HCI)

L'Infrastruttura Iperconvergente (HCI) è un'infrastruttura IT software-defined che aggrega risorse di calcolo e storage in un'unica piattaforma distribuita, superando i limiti dello storage esterno.

Definizione e Componenti di Base

I suoi componenti chiave convergono nello stesso server fisico (nodo):
Virtual Computing (Hypervisor)
Software-Defined Storage (SDS)
Rete IT Virtualizzata
I Requisiti Chiave di una Piattaforma HCI
Convergenza (Compute + Storage): Elaborazione e storage collassati in un unico stack hardware.
Storage Pool Unico: Lo storage locale di tutti i nodi viene condiviso in un unico grande pool di storage logico e scalabile.
Funzionalità Enterprise: Supporto nativo per HA, live migration, clonazione e snapshot.
Data Locality: I dati sono mantenuti il più vicino possibile all'esecuzione del calcolo (alla VM) per una latenza dello storage molto ridotta.
Scalabilità Lineare: L'aggiunta di un nodo aumenta simultaneamente CPU, RAM e IOPS in modo proporzionale.

📌 5. LA SOLUZIONE MODERNA: SDS + HCI (NUTANIX)

Ed eccoci a Nutanix, che risolve tutti i problemi visti prima.

🟢 Come funziona Nutanix
Ogni nodo Nutanix ha CPU, RAM e dischi (SSD/HDD).
Lo storage locale di tutti i nodi viene unito in un cluster distribuito.
Non esiste più un controller centrale → ogni nodo è anche un controller.
Risultato:
✔ Lo storage cresce semplicemente aggiungendo un nodo.
✔ Gli IOPS aumentano in modo lineare.
✔ La resilienza aumenta (repliche dei dati su più nodi).
✔ Nessun singolo punto di guasto.

🟢 Perché Nutanix è superiore alla SAN
Caratteristica	SAN/NAS-> Nutanix HCI
Scalabilità	verticale (costosa) -> scalabilità orizzontale, infinita
Controller	1 o 2 controller -> controller distribuiti su ogni nodo
IOPS	limitati, colli di bottiglia -> aumentano con ogni nodo
Complessità	alta (zone, fabric, LUN, RAID) -> molto semplice
Costo	alto -> più prevedibile e modulare

📌 6. ARCHITETTURA DI BASE DI UN NODO NUTANIX

Per operare con le funzionalità complete, un cluster Nutanix richiede un minimo di tre host fisici (nodi). 
L'architettura si basa sulla convergenza di hardware e software su ogni nodo:

1. Livello Hardware (Server Fisico)
Storage Ibrido: Lo storage locale è ibrido (SSD + HDD) o interamente Flash.
SSD (Solid State Drive): Utilizzati per il caching e le operazioni ad alta velocità.
HDD (Hard Disk Drive): Utilizzati per l'archiviazione (capacità a lungo termine).

2. Livello Hypervisor e Software Acropolis
L'hypervisor può essere AHV (nativo), VMware ESXi, Microsoft Hyper-V, o KVM.
Il software di Nutanix (Acropolis) gestisce lo storage locale.

3. Livello del Controller: La CVM (Controller Virtual Machine)
Il cuore dell'architettura. La CVM è una VM dedicata su ogni host che sostituisce il controller hardware esterno.
Distributed Storage Fabric: La CVM converte lo storage locale in un unico pool di storage logico distribuito a livello di cluster.
Se non c'è CVM, non c'è HCI, ossia una infrastruttura iperconvergente (Hyper-Converged Infrastructure)

---
📌 6B. CONFIGURAZIONE DELLO STORAGE IN NUTANIX: POOL, CONTAINER, VOLUME GROUP

**Dopo l’installazione e il primo accesso al cluster Nutanix, la prima operazione fondamentale è la configurazione dello storage.**

### Step iniziali
1. Accedi all’interfaccia di gestione Nutanix Prism.
2. Vai alla sezione “Storage”.
3. Visualizza la panoramica: qui trovi la situazione dei dischi, la capacità totale, lo spazio libero, i pool e i container esistenti.
4. Il sistema crea automaticamente un **Storage Pool** predefinito che aggrega tutti i dischi dei nodi del cluster.
5. Puoi aggiungere nuovi nodi: i loro dischi verranno aggiunti dinamicamente al pool esistente.

---
### Concetti chiave dello storage Nutanix

#### 📦 1) Storage Pool (SP)
- **Cos’è:** Il livello fisico dello storage Nutanix. Raggruppa tutti i dischi fisici(SSD + HDD) degli host del cluster in un unico pool distribuito.
- **Caratteristiche:**
  - Astrazione hardware: gestisce resilienza, tiering, compressione, deduplica.
  - Fornisce la capacità bruta e resiliente ai livelli superiori.
  - Ogni cluster Nutanix ha almeno uno storage pool.
- **A cosa serve:** Creare lo spazio condiviso da cui derivano gli Storage Container.

#### 📁 2) Storage Container (SC)
- **Cos’è:** Una porzione logica dello Storage Pool. All’interno del container risiedono le VM: dischi virtuali (VMDK/VHDX), snapshot, template.
- **Caratteristiche:**
  - Può avere politiche diverse dallo Storage Pool (compressione, deduplica, QoS, riservazioni spazio).
  - Utilizzato per: dischi VM, immagini, snapshot/replication targets.
- **A cosa serve:** Dare un’area logica e gestionale in cui risiedono le VM, pur usando lo storage fisico dello Storage Pool.

#### 🧩 3) Volume Group (VG)
- **Cos’è:** Un insieme di volumi a blocchi (iSCSI) forniti da Nutanix ad applicazioni o server fisici/VM che richiedono accesso block-level.
- **A cosa serve:** Offrire storage a blocchi (iSCSI) invece che file-based come gli Storage Container. Usato per database, cluster guest-based, applicazioni legacy.
- **Come funziona:** I volumi del VG si appoggiano allo Storage Pool, ma non sono contenuti in uno Storage Container. Vengono presentati via iSCSI come LUN a VM o server esterni.

---
### Sintesi dei concetti

Lo Storage Pool è il livello fisico dello storage Nutanix: raggruppa tutti i dischi del cluster, fornendo capacità grezza e resilienza, ed è la base su cui vengono creati i livelli superiori. 
Lo Storage Container è una suddivisione logica all’interno dello storage pool, dove risiedono le macchine virtuali, i dischi virtuali e gli snapshot; serve a gestire e organizzare lo spazio per le VM. 
Il Volume Group, invece, è un insieme di volumi a blocchi (iSCSI) pensato per offrire storage a blocchi a database, cluster o applicazioni legacy che richiedono accesso diretto ai volumi, distinto dallo storage file-based dei container.    |

# Vediamo un confronto con VMware per chiarire i concetti:

- Storage Pool: è il livello fisico che aggrega i dischi dei nodi Nutanix. In VMware il concetto più vicino è il gruppo di dischi/vSAN datastore, cioè il pool di capacità fisica che alimenta poi i datastore logici. È la parte in cui decidi quali dischi locali di un host entrano in un determinato disk group, stabilisci quali SSD fanno da cache e quali HDD forniscono capacità, e vSAN li somma in un'unica “capacità condivisa” prima ancora che tu crei i datastore VMFS/NFS che verranno montati dagli host. Se non usi vSAN ma hai una SAN/NAS tradizionale, pensa allo Storage Pool Nutanix come al RAID group/pool fisico che configuri sullo storage array (es. creare un pool RAID 5/6 su 10 dischi) prima di presentare una o più LUN agli host ESXi: è lo stesso livello “grezzo” dove definisci resilienza e capacità fisica, da cui poi nascono i datastore logici che l’hypervisor monta.
- Storage Container: è la porzione logica di quello storage che l’hypervisor monta per ospitare le VM. In VMware corrisponde al datastore (VMFS/NFS) che vedi dal vSphere Client e su cui risiedono i file delle macchine virtuali.
- Volume Group: espone volumi a blocchi via iSCSI verso i guest per esigenze “tradizionali” (database, cluster applicativi). In VMware l’analogia è con un LUN/RDM o con un target iSCSI che presenti direttamente al guest quando non vuoi passare dallo storage file-based del datastore. È il caso in cui, invece di lasciare che la VM usi un disco VMDK appoggiato a un datastore, vuoi farle “vedere” un disco raw, come faresti quando mappi un LUN SAN dentro il guest per un cluster SQL/Oracle o quando presenti una LUN RDM per un failover cluster: Nutanix crea questo LUN virtuale (Volume Group), lo espone via iSCSI, e il sistema operativo della VM lo gestisce come se fosse uno storage esterno dedicato, fuori dai file VMDK.

### Esempio pratico

Supponiamo di avere un cluster con 3 nodi. Dopo l’installazione:
- Tutti i dischi vengono aggregati in uno Storage Pool.
- Viene creato un Container predefinito dove risiederanno le VM.
- Se aggiungi un nodo, i suoi dischi si aggiungono automaticamente al pool.
- Puoi creare uno o più Volume Group per esigenze specifiche (es. database esterni via iSCSI).

---
**Nota:** Le funzionalità di efficienza (compressione, deduplica, erasure coding, fattore di replica) sono configurabili a livello di container.

---
📌 6C. FATTORE DI REPLICA (RF2/RF3) E RESILIENZA IN NUTANIX

- **Cos’è:** Il fattore di replica (RF) definisce quante copie complete di ogni blocco vengono mantenute nel cluster. Si configura per container da Prism.
- **Prerequisiti minimi:** RF2 richiede almeno 3 nodi (tolleranza a 1 nodo/disk fault); RF3 richiede almeno 5 nodi (tolleranza a 2 fault). Senza il numero minimo di nodi, il container non può essere impostato al relativo RF.
- **Come viene scritto il dato:** La CVM usa un commit log (simile a un journal) come area di staging per scritture casuali; i dati vengono poi flushati in modo sequenziale nell’Extent Store. Ogni write produce un checksum salvato nei metadati.
- **Ricostruzione e failover:** Se un disco o un nodo fallisce, le VM continuano a servire I/O leggendo dalle copie valide; in background parte l’autoriparazione per ripristinare la ridondanza persa. Con RF3 la piattaforma può perdere due failure domain senza fermarsi.
- **Verifica e protezione silente:** A ogni read viene ricalcolato il checksum; se non combacia, la CVM legge la replica e riscrive la copia corrotta. Lo scrubber gira quando i dischi sono poco utilizzati per intercettare errori latenti (bit rot, settori difettosi).
- **Sintesi:** La ridondanza (RF2/RF3) è il mezzo; la resilienza è il risultato: continuità del servizio anche con guasti di nodi o dischi, con self-healing automatico per mantenere il livello di protezione richiesto.

### Nota: cosa contiene veramente un VMDK?
- Un VMDK è solo un file che rappresenta un disco a blocchi. Finché la VM non lo inizializza è “vuoto” (raw). Quando il sistema operativo dentro la VM lo vede per la prima volta, lo partiziona e ci crea un file system (NTFS, ext4, XFS, ecc.) esattamente come farebbe su un disco fisico.
- Un disco raw/RDM non è diverso dal punto di vista del guest: al posto di un file VMDK riceve direttamente una LUN iSCSI/FC esterna, ma sempre blocchi grezzi sono. Anche qui il file system vive nel sistema operativo della VM.
- Morale: né i VMDK né i dischi raw hanno un file system “dell’hypervisor”. L’hypervisor consegna blocchi; è sempre il guest OS a decidere come formattarli e a gestire il proprio file system.
- Quindi sì, **entrambi sono grezzi** prima della formattazione: la sola differenza pratica è dove vivono quei blocchi (file `.vmdk` dentro un datastore vs LUN esterna passata in RDM), non il modo in cui il guest li usa.

> **Metafora disco vs file**  
> - “Vedere un disco” = ti danno un quaderno vuoto: sta a te decidere come dividerlo (partizioni) e cosa scriverci (file system). Finché non scrivi nulla, è solo carta bianca.  
> - “Vedere un file” = apri un capitolo già scritto dentro un quaderno che qualcuno ha già organizzato. Stai leggendo qualcosa che esiste perché il quaderno è stato preparato prima (formattato) e ci sono testi già salvati.  
> Così funziona anche per le VM: l’hypervisor ti consegna il quaderno vuoto (disco); il sistema operativo della VM lo organizza e ci crea i file. 

> **Datastore vs LUN esterna**  
> - **Datastore VMFS/NFS:** è uno spazio condiviso che l’hypervisor monta e gestisce a livello file: dentro ci stanno i VMDK, gli ISO, gli snapshot. Più host possono accedere allo stesso datastore per eseguire le VM.  
> - **LUN esterna/RDM:** è un volume a blocchi presentato direttamente al guest. L’hypervisor non lo usa per salvare file; fa solo pass-through verso la VM, che lo vede come disco dedicato.  
> - Quando usi un datastore, condividi lo storage via file VMDK. Quando usi una LUN esterna, dai a una VM (o a un cluster guest) un disco raw riservato, tipicamente per database o applicazioni che vogliono gestire da sole il proprio storage.
> - **Differenza vera:** è la modalità d’uso e gestione, non tanto le prestazioni. Un VMDK sta dentro un file su datastore ed è più facile da gestire (snapshot, vMotion). Un RDM/LUN raw è utile quando serve compatibilità con cluster guest o applicazioni che richiedono accesso diretto al volume. A livello di I/O nativo le prestazioni sono equivalenti perché entrambi parlano con lo stesso storage; cambiano solo le feature disponibili.
> - **Vantaggi VMDK:** gestione centralizzata (snapshot, cloning, Storage vMotion), portabilità, backup più semplice, supporto a tutte le feature VMware senza vincoli.  
> - **Vantaggi RDM/LUN raw:** compatibilità con software che pretende un disco “fisico” (cluster guest di Windows/Oracle, applicazioni che usano comandi SCSI speciali), possibilità di condividere lo stesso LUN tra più VM guest in cluster, mantenendo la logica di resilienza del vendor esterno.

È il concetto fondamentale su cui si basa Nutanix, in parole semplici:
a) Tradizionale (Converged): Hai 3 silos separati che devi comprare, collegare e gestire separatamente:
  - Server (Compute)
  - Storage Array (SAN)
  - Rete Storage (Fibre Channel)
b) Iperconvergente (HCI): Tutto è collassato dentro un unico box (il server).
Non c'è più la SAN esterna.
Lo storage è fatto dai dischi dentro i server.
Il software (come la CVM di Nutanix) unisce tutto e lo fa sembrare un unico grande sistema.

📌 7. IL CUORE DI NUTANIX: LA CVM (CONTROLLER VIRTUAL MACHINE)

La Controller Virtual Machine (CVM) è il vero "cervello" dell'infrastruttura iperconvergente.
Controller Distribuito: Elimina il controller storage centrale, distribuendo la gestione dello storage su tutti i nodi.
Data Locality: La CVM assicura che la VM legga i propri dati dai dischi locali del proprio host, garantendo una latenza bassissima.
Scalabilità Lineare: L'aggiunta di un nodo aggiunge simultaneamente CPU, RAM, dischi e un nuovo controller (CVM), garantendo un aumento prevedibile delle prestazioni (IOPS).

---
**Come funziona il percorso dei dati tra VM e storage in Nutanix**

Quando una macchina virtuale (VM) invia una richiesta di lettura o scrittura, questa passa prima attraverso lo switch virtuale dell'hypervisor. Lo switch virtuale collega la VM alla rete interna e agli altri servizi.

La richiesta viene poi inoltrata alla Controller Virtual Machine (CVM) presente sullo stesso nodo. La CVM gestisce l'I/O e decide dove salvare o recuperare i dati:
- Se i dati richiesti sono locali, la CVM li legge direttamente dal disco del nodo, garantendo prestazioni e latenza ottimali.
- Se i dati sono su un altro nodo, la CVM comunica con le CVM degli altri nodi tramite la rete interna del cluster per recuperare i dati.

In sintesi, la VM comunica con lo storage locale tramite la CVM e l'hardware del nodo. Questo processo sfrutta la data locality: i dati della VM vengono salvati preferibilmente sul disco locale del nodo dove la VM risiede. Se necessario, la CVM può accedere anche ai dati distribuiti su altri nodi.

Questo meccanismo garantisce:
- Latenza minima tra VM e storage.
- Prestazioni elevate grazie all'accesso diretto all'hardware locale.
- Ridondanza e resilienza, perché le repliche dei dati sono gestite tra nodi diversi.

📌 8. PERCHÉ NUTANIX RISOLVE I PROBLEMI DI PRESTAZIONE

Data locality → Latenza bassissima.
Distribuzione degli I/O → Più nodi = più potenza, nessun collo di bottiglia centrale.
Ridondanza nativa “a blocchi” → I dati sono distribuiti automaticamente tra nodi diversi.
Adatto agli hypervisor → Funziona con i principali hypervisor sul mercato.

📌 9. COME SI COLLEGA TUTTO QUESTO ALL’EVOLUZIONE DEL DATACENTER

In sintesi: 1️⃣ DAS → 2️⃣ SAN/NAS → 3️⃣ SDS/HCI (Nutanix)

Nutanix è utile perché:
elimina la complessità e i colli di bottiglia dello storage esterno,
fa scalare compute + storage insieme,
semplifica la gestione (simile a un servizio cloud),
aumenta resilienza e performance.

📌 10. RESILIENZA E REPLICHE DEI DATI (RF2 / RF3)

### Concetti base
- **Resilienza**: capacità del sistema di continuare a funzionare anche dopo guasti hardware o perdite dell’infrastruttura.
- **Ridondanza**: duplicazione di componenti o dati per evitare interruzioni del servizio in caso di guasto.
- **Fattore di Ridondanza (RF)**: misura quanti componenti hardware possono guastarsi senza interrompere il cluster; Nutanix supporta RF2 (default) e RF3. Passaggio consentito da RF2 a RF3, non viceversa: va deciso in fase di design.
- **Cosa protegge il RF**: guasti di nodi, dischi, blocchi o rack; il servizio resta disponibile grazie alle copie presenti su altri nodi.
- **Meccanismi di failover**: rilevamento dei guasti automatico, con spostamento immediato del carico su componenti ridondanti; la tolleranza può essere ragionata per nodo, blocco o rack.
- **Fattore di Replica dei dati**: opera a livello di container/dati e stabilisce quante copie logiche vengono mantenute (due o tre). È orientato alla protezione del dato, mentre il fattore di ridondanza è centrato sulla tolleranza ai guasti hardware.

### Fattore di Ridondanza (RF2 / RF3)
- **RF2 (Redundancy Factor 2)**: supporta il guasto di un singolo componente hardware (nodo, disco, alimentatore). Predefinito Nutanix, richiede almeno 3 nodi; consigliato per cluster piccoli/medi (es. 5–10 nodi) dove la probabilità di guasti multipli simultanei è bassa.
- **RF3 (Redundancy Factor 3)**: supporta il guasto simultaneo di due failure domain (nodi/dischi/blocchi/rack, in base a come distribuisci le repliche); indicato per ambienti grandi (centinaia/migliaia di nodi) o per dati estremamente critici anche in cluster piccoli. Offre più protezione ma consuma più risorse. Per tollerare più di due failure domain servono architetture diverse (es. replica sincrona/Metro tra cluster o soluzioni multi-site).
- **Nota**: è possibile salire da RF2 a RF3, non tornare indietro.
- **Esempio**: con RF2 la perdita contemporanea di due nodi può compromettere i dati presenti solo su quei nodi; con RF3 esiste un’ulteriore copia su un terzo nodo, evitando la perdita.
- **Requisiti minimi per RF2**: almeno 3 nodi e, per KVM, 24 GB di RAM come base; in scenari con I/O elevato o uso intensivo di SSD/NVMe/Flash, allocare più risorse. Nutanix integra ridondanze hardware (es. doppia alimentazione: se un PSU si guasta l’altro mantiene il servizio).
- **Scelta RF2 vs RF3**: RF2 è in genere sufficiente in ambienti piccoli/medi; RF3 è preferibile in ambienti molto grandi o quando i dati sono estremamente critici, indipendentemente dalla dimensione del cluster.
- **Commit sincrono**: ogni write è confermata solo dopo che tutte le copie previste dal RF scelto sono persistenti; RF2 non è “best effort”, è sempre commit su due nodi, RF3 su tre.

In sintesi, la disponibilità deriva dalla combinazione di ridondanza hardware, replica dei dati e failover automatico; la scelta di RF2 o RF3 va tarata su dimensione del cluster, probabilità di guasti simultanei, criticità dei dati e SLA richiesti.

### Esempio di failover hardware e data locality
- Se un nodo esce di servizio (CPU, RAM, rete o alimentazione), le VM vengono migrate/riallineate automaticamente su altri nodi che hanno capacità CPU/RAM/storage disponibile.
- La VM sul nuovo nodo legge/scrive prima sui dati locali; le repliche mancanti vengono recuperate in remoto e la piattaforma ricostruisce le copie (self-healing) in background, riportando la località dei dati sul nodo che ospita la VM.
- Le repliche vengono riallocate sui nodi meno caricati per ristabilire il livello di protezione RF scelto, senza interrompere i workload.

📌 10B. NUTANIX – RESILIENZA, RF E VM APPLICATIVE (SCENARIO CLOUDERA)

- **Resilienza & Ridondanza**: la piattaforma continua a funzionare nonostante guasti hardware; la ridondanza (RF2/RF3) duplica componenti/dati e il failover automatico sposta carichi e I/O su componenti sani.
- **RF2 vs RF3**: RF2 (min 3 nodi) tollera un failure domain; RF3 (min 5 nodi) ne tollera due. Si può salire da RF2 a RF3, non scendere. In cluster piccoli/medi RF2 è tipico; RF3 per cluster grandi o dati mission-critical.
- **Fattore di Replica**: a livello container/dati stabilisce quante copie (2 o 3); si occupa del dato, mentre il fattore di ridondanza è orientato all’hardware.
- **VM applicative vs CVM**: le VM utente (es. Rocky/Ubuntu/Windows, database/web/app) possono essere migrate/riavviate su altri nodi; la CVM è legata al nodo e non migra, ma le altre CVM mantengono i servizi storage.
- **Guasto nodo**: il nodo diventa indisponibile; le VM applicative vengono riavviate su altri nodi in base a CPU/RAM/storage; la data locality privilegia l’accesso locale, con recupero remoto e replica locale in background.
- **Scenario Nutanix + Cloudera**:
  - Nutanix: hypervisor (AHV/KVM), storage distribuito AOS, HA VM, replica RF2/RF3.
  - Cloudera: VM di management (Cloudera Manager, NameNode/Standby, ResourceManager, ZooKeeper), worker (DataNode, NodeManager, Spark Executor, HBase RegionServer), servizio (Edge/Gateway, Hive, Impala, Kafka, NiFi).
  - In caso di guasto fisico: le VM Cloudera si riavviano su altri nodi; le CVM restanti servono lo storage; Cloudera gestisce HA applicativa (NameNode HA, replica HDFS, riavvio job). Doppia resilienza: Nutanix per infrastruttura, Cloudera per dati/servizi Big Data.

Nutanix gestisce la tolleranza ai guasti e l'alta disponibilità anche attraverso il concetto di Fattore di Replica (Replication Factor - RF) a livello dati. Questo garantisce che i blocchi siano sempre disponibili, anche in caso di guasti a dischi o nodi interi.

### Approfondimenti rapidi
- **Resilienza & Ridondanza**: la continuità è ottenuta sommandone tre: (1) copie dei dati distribuite (RF), (2) servizi di controllo ridondanti (CVM su ogni nodo), (3) orchestrazione di failover automatica. In pratica, se una replica o un servizio cade, un altro prende il posto senza interventi manuali.
- **RF2 vs RF3, failure domain**: il failure domain può essere nodo, disco, blocco o rack (se i rack sono differenziati). RF2 garantisce che un failure domain possa cadere senza perdita di dati; RF3 garantisce lo stesso per due failure domain. Distribuire le repliche su rack/blocchi diversi aumenta la resilienza reale. Overhead stimato: RF2 ≈ 50% (1 copia di sicurezza), RF3 ≈ 66% (2 copie di sicurezza) di capacità grezza; va considerato nel sizing e nella crescita.
- **Upgrade RF2→RF3**: richiede capacità libera sufficiente per creare le copie aggiuntive; durante l’upgrade viene eseguito un rebalance per portare ogni estensione dati a tre copie. Non è reversibile senza ricreare i container/cluster.
- **Fattore di Replica**: è la policy per container; puoi avere container RF2 e RF3 nello stesso cluster per segmentare la protezione in base alla criticità dei dati, indipendentemente dal RF complessivo scelto per l’hardware.
- **Blocco vs chassis**: “blocco” e “chassis” coincidono; è l’enclosure fisica che ospita uno o più nodi (con PSU/backplane comuni). Un rack è l’armadio che contiene più blocchi; un nodo è il singolo server dentro il blocco. Distribuire repliche su blocchi/rack diversi riduce il rischio di perdere più copie per un guasto di alimentazione/backplane.
- **Filer: fisico vs software**: un filer fisico è un appliance NAS dedicato (controller + dischi) che espone share NFS/SMB o LUN; non è una SAN né una vSAN, ma un box separato dai server compute. Un filer software è un servizio/file server in VM o container che usa lo storage sottostante (es. container/VG Nutanix) per offrire condivisioni: è software-defined e gira sull’infrastruttura HCI, senza hardware dedicato.
- **VM applicative vs CVM**: le VM applicative seguono le regole dell’hypervisor (HA, DRS, affinità/anti-affinità) e possono migrare o essere riavviate altrove. La CVM è “pinned” al nodo perché usa l’HBA locale per servire I/O; non migra. Se il nodo è offline, la sua CVM sparisce, ma le altre CVM mantengono il servizio storage e i metadati; quando il nodo torna, la sua CVM riparte e partecipa al rebalance.
- **Flusso in caso di guasto nodo**: l’hypervisor rileva il failure, spegne forzatamente le VM, Prism/HA le riaccende su host sani con risorse disponibili. La VM riparte, legge prima i dati locali; se alcuni blocchi sono remoti, li legge via rete. Curator/rebalance riportano le repliche mancanti e la località dei dati sul nuovo host, ripristinando il livello RF target in background, senza downtime aggiuntivo.
- **Cosa rappresenta “B” nello schema**: le lettere (a, b, c, d, e, f, g) indicano blocchi di dati/extent di una VM, non VM o CVM. “B” è un blocco scritto originariamente su un nodo e replicato su altri nodi secondo RF2/RF3. Se il nodo originale è down, la VM riavviata legge “B” in remoto (es. da Node 3) e, se il dato diventa hot, Nutanix lo replica localmente in background sul nuovo nodo. Le write vengono riconosciute solo dopo che tutte le repliche previste da RF sono sicure, anche con un nodo giù.
- **Scenario Cloudera su Nutanix**: l’HA è stratificata. Nutanix assicura che le VM Cloudera ripartano e che lo storage sia servito; Cloudera assicura che i ruoli applicativi sopravvivano (NameNode/Standby, HDFS a 3 repliche, YARN restart). Esempio: se un nodo Nutanix cade, i DataNode e gli Executor su quel nodo vengono riavviati altrove; HDFS ribilancia/ricrea repliche; gli scheduler YARN ripartono i job. Il risultato è una resilienza combinata infrastruttura + applicazione.

### Schema ASCII: RF e failover di una VM
```
Cluster RF2 (3 nodi)

NODO 1 (down)      NODO 2 (VM attiva)      NODO 3
------------       ------------------      ------------
VM-X (offline)     VM-X (riavviata)        (repliche)
Blocchi dati:      Blocchi dati:           Blocchi dati:
 A (primaria)      A (replica)             B (primaria)
 B (replica)       B (remoto)              C (replica)
 C (remoto)        C (remoto)              ...

1) Nodo 1 cade → VM-X viene riavviata su Nodo 2.
2) VM-X legge localmente ciò che trova (es. A replica). Se un blocco (es. B) è remoto, lo legge da Nodo 3.
3) Le write sono ack solo quando tutte le copie RF sono persistenti (su Nodo 2 + Nodo 3).
4) In background curator/rebalance porta le repliche mancanti su nodi sani e ricrea la località (B verrà copiato su Nodo 2 se “hot”).
5) Nessun nodo contiene “tutte” le copie: ogni blocco ha quante repliche prevede RF e sono distribuite su failure domain diversi.
```

---
### Cos’è il Fattore di Replica (RF)?
Il Fattore di Replica (Replication Factor, RF) è il meccanismo con cui Nutanix garantisce che i dati siano sempre disponibili, anche in caso di guasti hardware (dischi o nodi). In pratica, ogni blocco di dati viene copiato più volte su nodi diversi del cluster.

#### RF2 (Replication Factor 2)
- **Minimo 3 nodi**: Serve almeno un cluster di 3 server fisici.
- **Come funziona**: Ogni blocco di dati viene scritto sull’host principale e replicato su altri 2 nodi (totale 3 copie: 1 originale + 2 repliche). Per host principale si intende il nodo dove risiede la VM che sta scrivendo i dati.
- **Tolleranza ai guasti**: Puoi perdere un intero nodo (server) oppure due dischi, e il sistema continua a funzionare senza perdere dati o interrompere i servizi.

#### RF3 (Replication Factor 3)
- **Minimo 5 nodi**: Serve almeno un cluster di 5 server fisici.
- **Come funziona**: Ogni blocco di dati viene scritto sull’host principale e replicato su altri 3 nodi (totale 4 copie: 1 originale + 3 repliche).
- **Tolleranza ai guasti**: Puoi perdere contemporaneamente due nodi (server) e il sistema continua a funzionare senza interruzioni.

---
#### Schema semplificato (commit sincrono su tutte le copie RF)

| RF | Minimo nodi | Copie totali | Nodi che puoi perdere |
|----|-------------|--------------|-----------------------|
| RF2| 3           | 3            | 1 nodo (o 2 dischi)   |
| RF3| 5           | 4            | 2 nodi                |

---
#### Esempio pratico
Supponiamo di avere un cluster Nutanix con 5 nodi e RF3:
- Scrivi un file: viene salvato sul nodo A e replicato su B, C e D.
- Se il nodo A e il nodo B si guastano, il file è ancora disponibile su C e D.

---
### Come avviene la replica?
- La replica è gestita dal software Nutanix (tramite la CVM, Controller Virtual Machine).
- Le copie dei dati sono distribuite in modo intelligente su nodi diversi, evitando che un singolo guasto possa causare la perdita di dati.
- Non c’è un controller centrale  come nelle SAN tradizionali: la resilienza è “distribuita” e non esiste un singolo punto di fallimento.

---
### In sintesi
- **RF2**: Protegge da guasti singoli (nodo o disco).
- **RF3**: Protegge da guasti multipli (fino a due nodi).
- **Vantaggio**: I dati sono sempre disponibili e il cluster continua a funzionare anche in caso di guasti hardware, grazie alla replica distribuita.

---
### Scelta del Fattore di Ridondanza (RF2 vs RF3)

- **RF2 (min 3 nodi)**: impostazione di default; tollera la perdita di 1 nodo o disco. Consigliato per cluster piccoli/medi o carichi meno critici.
- **RF3 (min 5 nodi)**: tollera la perdita contemporanea di 2 nodi. Preferibile per dati mission‑critical o cluster grandi; utilizzabile anche in ambienti piccoli se i dati sono critici.
- **Livello blocco/rack**: RF2 richiede almeno 3 blocchi/rack; RF3 almeno 5 blocchi/rack.
- **Container**: per ottenere i benefici di RF3, i container devono essere configurati con RF=3.
- **Nota pratica**: la ridondanza di alimentatori e dischi, insieme al self‑healing della piattaforma, contribuisce ulteriormente alla continuità operativa.

---
### Resilienza e Ridondanza contestualizzata a Nutanix

La resilienza in Nutanix non è solo il Fattore di Replica (RF). È un ecosistema completo di protezione e continuità:

**1. Ridondanza a livello di Cluster**
- I dati sono sempre distribuiti su nodi diversi. Se un nodo cade, il cluster continua a operare senza interruzioni.
- Non esiste un singolo punto di fallimento: ogni nodo è autonomo e gestisce il proprio storage.
- Se aggiungi un nodo al cluster, il sistema ribilancia automaticamente i dati per ottimizzare prestazioni e protezione.

**2. Self-Healing Automatico**
- Quando un nodo viene riportato online dopo un guasto, Nutanix rileva automaticamente i dati mancanti.
- Il sistema sincronizza i dati da altri nodi senza intervento manuale.
- Questo processo di "guarigione" avviene in background, senza impattare le VM in esecuzione.

**3. Ridondanza dei Servizi**
- Oltre ai dati, anche i servizi critici sono ridondanti: la CVM (Controller Virtual Machine) è presente su ogni nodo.
- Se la CVM di un nodo fallisce, un'altra CVM del cluster assume i compiti di controllo dello storage.
- Le VM continuano a funzionare su altri nodi senza perdita di dati o interruzioni significative.

**4. Protezione Geografica (Metro Availability)**
- Nutanix supporta replica synchrona di dati tra cluster geograficamente distribuiti.
- In caso di disastro in un datacenter, il cluster remoto è pronto a prendere il controllo delle VM senza perdita di dati.
- RPO (Recovery Point Objective) = 0, RTO (Recovery Time Objective) = minuti.

**5. Snapshot e Backup**
- Nutanix consente snapshot istantanei delle VM per proteggere i dati da corruzione o errori applicativi.
- Gli snapshot occupano spazio minimo grazie a deduplica e compressione.
- Backup replicati su cluster remoti per protezione ulteriore.

**Contesto Pratico:**
In una SAN tradizionale, il fallimento del controller centralizzato ferma tutto il datacenter. In Nutanix, il fallimento di un nodo (o perfino due) non causa nessuna interruzione. I dati rimangono accessibili, le VM continuano a funzionare, e il sistema si auto-ripara.

Questo è il motivo per cui Nutanix raggiunge SLA del 99,99% o superiore senza richiedere hardware ridondante dedicato o failover complessi.

📌 11. CONFRONTO DIRETTO: NUTANIX HCI vs. VMWARE vSAN

**1. Architettura del Controller**
- Nutanix: Ogni nodo ha una Controller Virtual Machine (CVM) dedicata che gestisce lo storage. La gestione è distribuita, senza colli di bottiglia.
- vSAN: La logica di storage è integrata nel kernel dell’hypervisor ESXi. Il controllo è centralizzato nel software dell’hypervisor.

**2. Hypervisor Supportato**
- Nutanix: Supporta diversi hypervisor (AHV nativo, VMware ESXi, Microsoft Hyper-V, KVM). Flessibile e agnostico.
- vSAN: Funziona solo con VMware vSphere/ESXi. Soluzione monolitica e vincolata all’ecosistema VMware.

**3. Piattaforma**
- Nutanix: Offre una piattaforma di cloud ibrido con servizi integrati (file, database, networking, ecc.), andando oltre il semplice storage.
- vSAN: Focalizzata principalmente sullo storage per ambienti vSphere.

**4. Flessibilità Hardware**
- Nutanix: Può essere installata su hardware certificato di vari produttori (Dell, HPE, Lenovo, ecc.) o su appliance Nutanix NX.
- vSAN: Richiede server che rispettino la vSAN Hardware Compatibility List (HCL), quindi la scelta hardware è più limitata.

**Sintesi**
- Nutanix è una soluzione software-defined, flessibile e distribuita, con una CVM dedicata su ogni nodo.
- vSAN è integrata nel kernel di VMware, meno flessibile e legata all’ecosistema VMware.


📌 12. PERCORSO DI MIGRAZIONE: DA SAN TRADIZIONALE A HCI

Il passaggio da un'infrastruttura tradizionale basata su SAN/NAS a Nutanix HCI è tipicamente un processo graduale, che minimizza i rischi e i tempi di inattività.

# Fasi della Migrazione:
- Installazione e Configurazione: Installazione di un nuovo cluster Nutanix HCI in parallelo all'infrastruttura SAN esistente. Entrambi gli ambienti coesistono.
- Validazione: Esecuzione di test (prestazioni, funzionalità) sul nuovo cluster HCI utilizzando carichi di lavoro non critici.
- Migrazione dei Dati/VM: Le macchine virtuali vengono spostate gradualmente dalla vecchia SAN al nuovo cluster Nutanix.
- Metodo a Zero Downtime: Se l'hypervisor è VMware, si può utilizzare la tecnologia vMotion per spostare le VM in esecuzione dalla SAN a Nutanix senza interruzioni di servizio.
- Metodo Nutanix: Strumenti nativi di Nutanix (come l'utility Move) possono facilitare la migrazione di massa.
- Decommissioning: Una volta che tutti i carichi di lavoro critici sono migrati e il cluster HCI è completamente validato, l'hardware legacy (controller SAN, storage array, switch Fibre Channel) viene spento e rimosso.
Questo approccio a "big bang" (tutto o niente) è evitato in favore di una migrazione graduale e non interruttiva, sfruttando la capacità di Nutanix di operare in parallelo all'infrastruttura esistente.

📌 13. APPROFONDIMENTO: BLOCCHI, NODI E GESTIONE DELLO STORAGE

Per verificare la presenza e l'operatività di un cluster Nutanix, è necessario disporre di almeno **tre host fisici (nodi)**.

**Concetto di Blocco e Nodo:**
- Un **Blocco** (Chassis) è l'unità fisica che ospita i server. Un blocco può contenere, ad esempio, quattro nodi, due nodi o un singolo nodo.
- Un **Nodo** corrisponde a un server fisico.

Se non si dispone di almeno tre nodi (server fisici), non è possibile beneficiare di tutti i vantaggi della soluzione Nutanix (come la resilienza RF2).

**Architettura del Nodo:**
Ogni nodo è un server bare metal che contiene:
1.  **Hardware di calcolo**: CPU e Memoria.
2.  **Controller Storage**: HBA (Host Bus Adapter) che controlla direttamente i dischi.
3.  **Storage Ibrido o All-Flash**:
    *   **Ibrido**: Combina SSD e HDD. Nutanix utilizza gli SSD per il **caching** (prestazioni) e gli HDD per l'**archiviazione** (capacity).
    *   **All-Flash**: Utilizza solo SSD per garantire massime prestazioni sia per il caching che per lo storage.

**Livelli Software su ogni Nodo:**

1.  **Hypervisor**: Installato direttamente sull'hardware. Può essere l'hypervisor nativo **AHV**, oppure **VMware ESXi**, **Hyper-V**, ecc.
2.  **CVM (Controller Virtual Machine)**: Sopra l'hypervisor, su ogni nodo, gira una macchina virtuale speciale chiamata CVM.
    *   La CVM gestisce lo storage locale (tramite pass-through del controller HBA).
    *   Sostituisce i controller RAID hardware tradizionali.
    *   Trasforma lo storage locale in un **Distributed Storage Fabric**.

In sintesi, il software Nutanix (Acropolis) converte i server fisici in un'infrastruttura iperconvergente, gestendo lo storage via software tramite la CVM invece che via hardware dedicato.

📌 14. APPROFONDIMENTO: NETWORKING E FUNZIONALITÀ DSF

Oltre all'hardware, il corretto funzionamento del cluster Nutanix dipende da una solida configurazione di rete e dalle funzionalità avanzate del Distributed Storage Fabric.

**Comunicazione tra CVM:**
- Le CVM (Controller VM) di ogni nodo devono comunicare costantemente tra loro per gestire il cluster e lo storage.
- È fondamentale che le CVM risiedano nella **stessa rete (VLAN/Subnet)** e abbiano indirizzi IP in grado di parlarsi direttamente.
- Questa rete "Backplane" gestisce il traffico di replica dei dati e i comandi di gestione.

**Requisiti di Rete Fisica:**
- **Velocità**: Sebbene 1 GbE sia il minimo teorico, per un ambiente di produzione è fortemente raccomandato l'uso di schede di rete e switch a **10 GbE** (o superiore).
- Questo garantisce che la latenza di rete non diventi un collo di bottiglia per le prestazioni dello storage distribuito.

**Funzionalità Avanzate del Distributed Storage Fabric (DSF):**
Una volta creato il cluster (minimo 3 nodi), il DSF abilita funzionalità enterprise avanzate sui dati:
- **Data Locality**: Mantiene i dati "caldi" sul nodo dove gira la VM per massime prestazioni.
- **Deduplication & Compression**: Ottimizzazione dello spazio disco.
- **Erasure Coding (EC-X)**: Aumenta lo spazio utilizzabile mantenendo la resilienza (simile al RAID 5/6 ma distribuito).
- **Snapshot & Cloni**: Gestione efficiente delle copie dei dati.
- **Tiering**: Spostamento automatico dei dati tra SSD (Hot) e HDD (Cold).

📌 15. APPROFONDIMENTO: DATA LOCALITY (IL VANTAGGIO CHIAVE)

Uno dei vantaggi fondamentali dell'architettura Nutanix è la **Data Locality** (Località dei Dati).

**Come funziona:**
Quando si crea una macchina virtuale (es. sul **Nodo A**), il sistema intelligente (CVM) fa in modo che i dati di quella VM vengano scritti e salvati preferibilmente nel **disco locale del Nodo A**.
- Nonostante lo storage sia un unico pool distribuito, la CVM "tira" i dati verso l'host dove risiede la VM.
- **Risultato**: La VM legge e scrive direttamente sull'hardware locale (tramite HBA), ottenendo prestazioni massime e latenza minima, senza dover attraversare la rete.

**Gestione della Resilienza (RF):**
Anche se i dati sono locali, Nutanix garantisce la sicurezza:
- Una copia del dato viene scritta localmente (per le prestazioni).
- Una o più copie (repliche) vengono scritte simultaneamente su altri nodi (es. Nodo B o C) attraverso la rete, per garantire la disponibilità in caso di guasto del Nodo A.

**Confronto con Storage Tradizionale (SAN):**
- **In una SAN tradizionale**: La VM deve sempre uscire dall'host, attraversare la rete (Fibre Channel/Ethernet) e raggiungere lo storage array esterno per ogni operazione di lettura/scrittura.
- **In Nutanix**: Il dato è "a km 0", risiede dove serve. La rete viene usata principalmente per la replica di sicurezza, non per il traffico I/O primario.

📌 ## 16. COMPONENTI PRINCIPALI DEL CLUSTER NUTANIX ##

Ora vogliamo capire quali sono i diversi componenti del cluster Nutanix. Ogni indagine ha una componente specifica che utilizziamo per gestire il nostro ambiente. L'elenco seguente vi aiuterà a capire come funzionano i componenti del cluster e a individuare rapidamente quale servizio è coinvolto in caso di problemi.
Iniziamo con la semplice architettura di flusso: il browser (o un tool di orchestrazione) si connette all'ambiente Nutanix per inviare richieste o attività. 
Queste richieste passano attraverso Prism, Zookeeper, Curator, Cassandra e Stargate fino all'hypervisor del server. Vediamo i componenti uno per uno:

- **Stargate**: gestore I/O dei dati (distribuito). Espone lo storage agli altri sistemi e rappresenta il punto di contatto principale per l'hypervisor tramite NFS/iSCSI/SMB. Dipende da Medusa per i metadati e da Zeus per la configurazione del cluster.
- **Cassandra**: archivio distribuito dei metadati, basato su Apache Cassandra modificato con coerenza forte (Paxos). Eseguito su ogni nodo; accesso tramite l'interfaccia Medusa; dipende da Zeus per le informazioni di cluster.
- **Medusa (interfaccia Cassandra)**: livello di astrazione e accesso a Cassandra, utilizzato da componenti che devono leggere/scrivere metadati (es. per tracciare posizione dei dati e repliche).
- **Curator**: orchestratore MapReduce per gestione e pulizia del cluster (bilanciamento dischi, scrubbing, ecc.). Gira su ogni nodo e opera tramite un leader eletto; dipende da Zeus per lo stato dei nodi e da Medusa per i dati, inviando poi comandi a Stargate.
- **Zookeeper (Zeus)**: configuration manager del cluster basato su Apache Zookeeper. Conserva configurazione e stato di host, API, regole, ecc. Gira su 3 o 5 nodi in base al fattore di replica; uno viene eletto leader. Non ha dipendenze e può partire senza altri servizi.
- **Prism**: interfaccia di gestione Nutanix (UI, CLI, REST API). Gira su ogni nodo con un leader eletto; sfrutta le tabelle IP di Linux per inoltrare le richieste al leader anche se ci si collega a un IP “slave” o virtuale. Comunica con Zeus per configurazione, con Cassandra per statistiche e con l'hypervisor per lo stato delle VM.

Conoscere questi componenti permette di risalire velocemente al servizio giusto quando si verifica un problema (es. verificare se un servizio è attivo o meno) e capire cosa avviene in background quando ci si connette all'ambiente cluster via browser.

📌 17. BLOCCO, NODO E RACK (HARDWARE) E SCELTA DELL’APPLIANCE

Ciao e benvenuto di nuovo. Prima di progettare capacità e resilienza è fondamentale capire come si compongono blocco, nodo e rack.

- **Hardware non vincolante**: Nutanix usa i propri blocchi come esempio, ma puoi adottare qualsiasi hardware certificato per HCI (Dell, HPE, Lenovo, IBM, ecc.). Se non usi appliance Nutanix, dovrai acquistare la licenza software Nutanix per ottenere le stesse funzionalità (hypervisor AHV e gestione incluse solo nell’hardware Nutanix).
- **Blocco (chassis)**: Enclosure che contiene più nodi (es. 1, 2, 4 nodi). La resilienza può essere ragionata a livello di nodo, di blocco o di rack.
- **Nodo**: Singolo server fisico dentro il blocco; unità minima per il cluster e per le scelte di resilienza (RF2/RF3).
- **Rack**: Insieme di blocchi e switch; la distribuzione dei nodi su rack diversi migliora la disponibilità in scenari di fault dominanti.

In sintesi: per beneficiare della piattaforma iperconvergente conta che l’hardware supporti HCI; l’esempio con blocchi Nutanix è solo di riferimento. La resilienza viene calcolata distribuendo i nodi (e i blocchi) tra rack diversi quando possibile.

📌 18. CHE COS’È UN RACK

Un rack (o server rack) è un armadio/struttura che ospita più blocchi o server montati su guide (bay) fissate con viti. Profilo basso e chiuso, a differenza di un tower.

- **Spazio e cablaggio**: più server uno sopra l’altro, con switch di rete in testa o alla base per organizzare connettività e cavi.
- **Raffreddamento**: serve un sistema di cooling adeguato perché molti componenti in poco spazio generano calore.
- **Unità rack (U)**: l’altezza si misura in “U” (1U = 1,75 pollici). Un blocco 1U usa una sola unità; un blocco 2U ne usa due, ecc.
- **Capacità**: un rack da 16U può ospitare, ad esempio, 16 blocchi 1U oppure 8 blocchi 2U; la capacità varia per modello e marca.

📌 19. COS’È UN BLOCCO

Il blocco (chassis) è l’unità hardware che ospita uno o più nodi fisici. Espone sul frontale gli slot per SSD/HDD hot-swap. A seconda del modello può contenere 1, 2, 3 o 4 nodi e occupa 1U o 2U nel rack. Il concetto è vendor-agnostico: vale per i blocchi Nutanix NX e per chassis equivalenti di altri produttori.

📌 20. COS’È UN NODO

Il nodo è il singolo server fisico dentro il blocco, con CPU, RAM, dischi frontali dedicati e proprie interfacce di rete. I nodi non condividono i dischi tra loro: ciascuno utilizza solo i drive assegnati nel blocco. In base al modello del blocco puoi avere layout a nodo unico (un nodo grande che occupa tutto lo chassis) oppure multi-nodo (fino a 4 nodi nello stesso blocco).

📌 21. COME SI LEGGE IL MODELLO HARDWARE

Le stringhe di modello riportate sul frontale/laterale del blocco indicano marca, famiglia e configurazione:

- **Prefisso di marca**: identifica il vendor (es. NX per Nutanix, altri prefissi per hardware di terze parti).
- **Famiglia (prima cifra)**: 1 = edge/filiali o piccoli uffici; 3 = uso generale con compute più robusto; 5 = file/object workload dedicati; 6 = storage-heavy; 8 = alte prestazioni (CPU/GPU, I/O elevato); 9 = alta prestazione/sperimentale.
- **Conteggio nodi (seconda cifra)**: quante unità server sono presenti nel blocco (1, 2, 3 o 4).
- **Form factor chassis (terza cifra)**: indica l’altezza in rack e il layout (es. 7 spesso 1U mono-nodo, 6 tipicamente 2U mono-nodo; varia per famiglia/modello).
- **Form factor dischi (quarta cifra)**: 0 per drive da 2.5", 5 per drive da 3.5".
- **Variante (suffisso opzionale)**: lettere/numeri per capacità, presenza GPU o CPU single/dual.
- **Generazione CPU (Gx)**: G4 Haswell, G5 Broadwell, G6 Skylake, G7 Cascade Lake (e successivi).

📌 22. INSTALLAZIONE CON NUTANIX FOUNDATION

Per installare Nutanix in laboratorio o in produzione si utilizza Nutanix Foundation.

- **Opzioni Foundation**: VM preconfigurata da eseguire su VMware Workstation/Fusion/VirtualBox, oppure Foundation for Windows/Mac installabile direttamente sul PC (senza hypervisor locale).
- **Download**: scarica la Foundation VM dal portale Nutanix o l’installer Foundation per Windows/Mac.
- **Switch di rete**: per un home lab con un blocco fino a 4 nodi serve uno switch Gigabit da almeno 8 porte; una porta per ciascun nodo più una per il notebook di installazione. In ambienti più grandi, dimensiona le porte in base al numero di nodi.
- **Cabling**: un cavo Ethernet per ogni nodo + uno per il notebook, tutti collegati allo stesso switch.
- **Hypervisor ISO**: tieni disponibili le immagini ISO dell’hypervisor che intendi installare (VMware ESXi, Microsoft Hyper-V, Citrix, ecc.).
- **Notebook/PC di controllo**: ospita la Foundation VM oppure l’installer Foundation for Windows/Mac e avvia il workflow di installazione verso i nodi del blocco.

📌 23. ESEMPIO DI BLOCCO NUTANIX A 4 NODI (VISTA FRONT/BACK)

Ecco un esempio pratico di blocco Nutanix con quattro nodi (A, B, C, D), utile per orientarsi su slot dischi e porte.

- **Isolamento nodi**: i nodi non condividono risorse tra loro, salvo le alimentazioni e lo chassis.
- **Alimentazione ridondata**: due PSU condivise nello chassis, da collegare a feed elettrici separati.
- **Porte di rete per nodo**: tipicamente 2x SFP+ 10GbE (quattro porte totali se si considerano coppie) + 1x RJ45 (management/servizio), più eventuale spazio per aggiungere una NIC ulteriore; porte USB 3.0 e VGA per console locale.
- **Slot dischi per nodo (frontale)**: sei bay per SSD/HDD (es. 6× ~2 TB per nodo); ogni nodo usa solo i propri dischi, non accede a quelli degli altri.
- **Identificazione nodi**: serigrafie “Node A/B/C/D” sul frontale/posteriore per individuare dischi e porte corrispondenti.

📌 24. PORTE DI GESTIONE E CABLAGGIO PER FOUNDATION

Per avviare Foundation va identificata la porta di gestione IP (BMC/IPMI) di ciascun nodo e va collegata allo switch usato per l’installazione.

- **Porta di gestione**: spesso RJ45 dedicata; su alcuni modelli si può usare una porta dati 10GbE condivisa per la gestione. Usare la porta indicata come management/IPMI per la messa in servizio.
- **Locator**: LED “locator”/identify sul nodo aiutano a riconoscere l’host corretto da software.
- **Cabling**: un cavo Ethernet dalla porta di gestione di ogni nodo allo switch (home/office) usato per Foundation; collegare anche il notebook allo stesso switch.
- **Esempio 3-4 nodi**: per 3 nodi servono 3 porte di switch + 1 per il notebook; per 4 nodi, 4 porte + 1 notebook. Dimensiona le porte in base al numero di nodi.
- **Nota**: le porte dati 10GbE resteranno per il traffico di produzione; per Foundation si usa la porta di gestione.
