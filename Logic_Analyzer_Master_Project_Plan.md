# Logic Analyzer USB — Master Project Plan

**Documento:** LA-MASTER-001  
**Versione:** 0.2  
**Data:** 25 agosto 2026  
**Stato:** Bozza architetturale / baseline iniziale  
**Target iniziale:** STM32H723 + USB 2.0 Full-Speed  
**Evoluzione prevista:** V1 STM32 → V2 FPGA → V3 Custom Board  

---

## 1. Scopo del documento

Questo documento definisce la struttura complessiva del progetto per la realizzazione di un logic analyzer USB proprietario, composto da hardware embedded, firmware di acquisizione e software host per PC.

L'obiettivo non è costruire una semplice demo capace di leggere un GPIO, ma sviluppare progressivamente uno strumento realmente utilizzabile per il debug di sistemi embedded, con particolare attenzione ai protocolli:

- UART;
- SPI;
- I²C.

Il progetto sarà sviluppato in tre generazioni:

1. **Versione 1 — STM32H723**
   - prototipo funzionante basato su microcontrollore;
   - 8 ingressi digitali;
   - acquisizione parallela;
   - DMA;
   - trigger;
   - pre-trigger e post-trigger;
   - USB 2.0 Full-Speed;
   - software host;
   - decoder UART/SPI/I²C.

2. **Versione 2 — FPGA**
   - aumento sostanziale della frequenza di campionamento;
   - maggiore parallelismo;
   - trigger hardware avanzati;
   - buffer più profondi;
   - possibile compressione hardware;
   - predisposizione per 16 o più canali;
   - possibile migrazione a USB 3.x.

3. **Versione 3 — Custom Board**
   - progettazione di una PCB dedicata;
   - integrazione front-end digitale;
   - protezioni;
   - FPGA/MCU;
   - memoria;
   - interfaccia USB;
   - alimentazione;
   - connettori per sonde;
   - eventuale enclosure.

Questo documento costituisce il **master plan**. Non deve contenere ancora tutti i dettagli implementativi di ogni singola fase. Ogni milestone principale sarà successivamente descritta in un documento tecnico dedicato.

---

# 2. Visione del progetto

Il sistema completo dovrà seguire una separazione netta tra:

1. **front-end elettrico**;
2. **acquisition engine**;
3. **trigger engine**;
4. **capture memory**;
5. **transport layer USB**;
6. **host acquisition software**;
7. **protocol decoders**;
8. **waveform viewer / GUI**.

L'architettura concettuale è:

```text
                        LOGIC ANALYZER

       DUT
        │
        │ digital signals
        ▼
┌─────────────────────┐
│ Input front-end     │
│                     │
│ protection          │
│ buffering           │
│ level compatibility │
└─────────┬───────────┘
          │
          │ CH[7:0]
          ▼
┌─────────────────────┐
│ Acquisition Engine  │
│                     │
│ PSSI / FPGA sampler │
│ sample clock        │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│ Trigger Engine      │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│ Capture Buffer      │
│                     │
│ circular buffer     │
│ pre/post trigger    │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│ USB Transport       │
└─────────┬───────────┘
          │
══════════╪═══════════════════════════════
          │ PC
          ▼
┌─────────────────────┐
│ Host Acquisition    │
└─────────┬───────────┘
          │
          ├───────────────┐
          │               │
          ▼               ▼
┌────────────────┐  ┌─────────────────┐
│ Waveform View  │  │ Decoder Engine  │
└────────────────┘  └───────┬─────────┘
                             │
                  ┌──────────┼──────────┐
                  ▼          ▼          ▼
                UART        SPI        I²C
```

L'elemento fondamentale è che **i decoder non devono essere accoppiati all'hardware di acquisizione**.

L'acquisition engine produce una sequenza di campioni digitali. Il software PC riceve questi campioni e successivamente li interpreta.

Questa scelta permette di:

- riutilizzare il software host quando si passerà da STM32 a FPGA;
- aggiungere nuovi decoder senza modificare il firmware;
- salvare acquisizioni raw e rianalizzarle offline;
- testare i decoder senza hardware;
- implementare in futuro compatibilità con formati esterni o sigrok/PulseView.

---

# 3. Principi architetturali

## 3.1 Separazione tra acquisizione e decodifica

Il dispositivo embedded non dovrà inizialmente interpretare UART, SPI o I²C.

Il dispositivo dovrà trasmettere campioni digitali grezzi:

```text
sample[0]
sample[1]
sample[2]
...
sample[N-1]
```

Per 8 canali:

```text
bit 7  bit 6  bit 5  bit 4  bit 3  bit 2  bit 1  bit 0
 CH7    CH6    CH5    CH4    CH3    CH2    CH1    CH0
```

Quindi:

```text
1 sample = 1 byte
```

Il tempo associato al campione `n` è:

```text
t[n] = n / Fs
```

dove `Fs` è la sample rate configurata.

Non è quindi necessario trasmettere un timestamp per ogni singolo campione.

---

## 3.2 Data path indipendente dalla CPU

Il Cortex-M7 non deve leggere continuamente i GPIO mediante polling.

Il percorso principale deve essere:

```text
GPIO
  │
  ▼
PSSI
  │
  ▼
DMA
  │
  ▼
RAM
```

La CPU deve occuparsi principalmente di:

- configurazione;
- stato del dispositivo;
- gestione trigger;
- orchestrazione DMA;
- protocollo USB;
- error handling;
- telemetria;
- gestione comandi provenienti dal PC.

L'acquisizione ad alta frequenza deve avvenire il più possibile in hardware.

---

## 3.3 Raw data first

La prima versione del sistema deve privilegiare la correttezza del dato raw.

Ordine delle priorità:

```text
1. campionamento corretto
2. nessuna perdita di campioni
3. corretta ricostruzione temporale
4. trigger affidabile
5. trasporto USB affidabile
6. decoder
7. GUI
8. feature avanzate
```

Una GUI sofisticata sopra un acquisition engine non affidabile non ha alcun valore.

---

# 4. Concetti fondamentali

## 4.1 Sample rate

La sample rate indica quante volte al secondo il logic analyzer osserva gli ingressi digitali.

Esempio:

```text
Fs = 10 MS/s
```

significa:

```text
10 000 000 campioni / secondo
```

Il periodo di campionamento è:

```text
Ts = 1 / Fs
```

A 10 MS/s:

```text
Ts = 100 ns
```

A 50 MS/s:

```text
Ts = 20 ns
```

---

## 4.2 Oversampling di un protocollo

Per ricostruire correttamente un protocollo digitale è consigliabile campionarlo a una frequenza significativamente superiore rispetto alla frequenza del segnale.

Esempio SPI:

```text
SPI clock = 5 MHz
logic analyzer = 50 MS/s
```

Si ottengono:

```text
50 / 5 = 10 campioni per periodo SPI
```

Questo consente di identificare con buona precisione:

- rising edge;
- falling edge;
- setup;
- hold;
- valore MOSI;
- valore MISO.

Una regola pratica iniziale per il progetto sarà:

> Per una decodifica robusta, puntare quando possibile ad almeno 4–10 campioni per periodo del segnale più veloce di interesse.

---

## 4.3 Trigger

Il trigger identifica un evento di interesse.

Esempio:

```text
trigger = rising edge su CH3
```

Quando l'evento viene rilevato, esso viene assunto come riferimento temporale:

```text
t = 0
```

Il trigger non coincide necessariamente con l'inizio dell'acquisizione.

---

## 4.4 Pre-trigger

Il pre-trigger contiene i campioni registrati **prima** dell'evento di trigger.

Esempio:

```text
capture totale = 10 ms
trigger position = 40%
```

Risultato:

```text
4 ms prima del trigger
6 ms dopo il trigger
```

Diagramma:

```text
                        TRIGGER
                           │
                           ▼
███████████████████████████│████████████████████████████
        PRE-TRIGGER        │          POST-TRIGGER
                           │
                          t=0
```

Il pre-trigger è fondamentale quando si cerca la causa di un errore.

Esempio:

```text
SPI transaction
       │
       ▼
I2C transaction
       │
       ▼
software state change
       │
       ▼
FAULT GPIO ───────────────► trigger
```

Senza pre-trigger si vedrebbe soltanto ciò che accade dopo il fault.

---

## 4.5 Post-trigger

Il post-trigger contiene i campioni acquisiti dopo l'evento.

Esempio:

```text
trigger position = 20%
```

significa:

```text
20% buffer = pre-trigger
80% buffer = post-trigger
```

Il parametro da esporre all'utente sarà preferibilmente:

```text
Trigger Position [%]
```

piuttosto che due parametri indipendenti.

---

## 4.6 Circular buffer

Per implementare il pre-trigger, il buffer di acquisizione lavora continuamente in modalità circolare.

```text
[0][1][2][3][4][5][6][7][8][9]
             ▲
             │
         write pointer
```

Quando viene raggiunta la fine:

```text
[0][1][2][3][4][5][6][7][8][9]
 ▲
 │
write pointer
```

i nuovi campioni sovrascrivono quelli più vecchi.

Al momento del trigger viene memorizzata la posizione logica dell'evento; l'acquisizione continua fino al completamento della porzione post-trigger richiesta.

---

# 5. USB: Full-Speed e High-Speed

## 5.1 Terminologia

Per questo progetto è fondamentale distinguere tra:

### USB Full-Speed

```text
12 Mbit/s
```

### USB High-Speed

```text
480 Mbit/s
```

USB 2.0 include sia Full-Speed sia High-Speed. La dicitura "USB 2.0" da sola non implica quindi una velocità di 480 Mbit/s.

La **Versione 1** userà deliberatamente **USB 2.0 Full-Speed**, sfruttando il PHY integrato nello STM32H723. L'obiettivo della V1 non è lo streaming continuo ad alta banda, ma la validazione completa dell'acquisition engine, del trigger, del buffering e del software host.

---

## 5.2 STM32H723 e USB nella V1

Lo STM32H723 integra:

- controller USB OTG;
- PHY Full-Speed interno;
- supporto USB Full-Speed a 12 Mbit/s;
- interfaccia ULPI disponibile per un eventuale PHY High-Speed esterno.

La baseline V1 sarà quindi:

```text
STM32H723
    │
    │ USB D+ / D-
    ▼
USB 2.0 Full-Speed
    │
    ▼
    PC
```

Non verrà utilizzato un PHY ULPI esterno nella baseline V1.

La USB sarà utilizzata principalmente per:

- configurare il dispositivo;
- armare e fermare le acquisizioni;
- scaricare acquisizioni buffered;
- trasferire telemetria e stato;
- sperimentare uno streaming limitato alle velocità sostenibili dalla Full-Speed.

---

## 5.3 Impatto della scelta Full-Speed

USB Full-Speed ha una signaling rate nominale di:

```text
12 Mbit/s
```

equivalente teoricamente a:

```text
1.5 MB/s
```

Il payload applicativo reale sarà inferiore a causa di:

- framing;
- token;
- handshake;
- scheduling;
- endpoint management;
- stack USB;
- driver host;
- copie e gestione buffer.

Per questo motivo il throughput sostenuto dovrà essere misurato sperimentalmente.

La conseguenza fondamentale è:

> USB Full-Speed limita il trasferimento verso il PC e lo streaming continuo, ma non impedisce di acquisire internamente a sample rate molto superiori in modalità buffered.

Esempio:

```text
8 canali
50 MS/s
1 byte/sample
```

producono:

```text
50 MB/s
```

durante l'acquisizione, ma il dato può essere prima salvato in RAM e successivamente trasferito al PC a una velocità molto più bassa.

Questa separazione tra acquisition bandwidth e transport bandwidth è una scelta architetturale esplicita della V1.

---

# 6. Data-rate budget della V1

Con 8 canali e un byte per campione:

| Sample rate | Data rate raw |
|---:|---:|
| 1 MS/s | 1 MB/s |
| 2 MS/s | 2 MB/s |
| 5 MS/s | 5 MB/s |
| 10 MS/s | 10 MB/s |
| 20 MS/s | 20 MB/s |
| 25 MS/s | 25 MB/s |
| 50 MS/s | 50 MB/s |

Il target massimo iniziale rimane:

```text
50 MS/s
```

in modalità buffered. La sample rate di acquisizione è indipendente dalla banda USB fintanto che i campioni vengono memorizzati localmente prima del trasferimento.

Devono essere distinte due modalità.

---

## 6.1 Buffered Capture Mode

```text
inputs
  │
  ▼
PSSI
  │
  ▼
DMA
  │
  ▼
capture RAM
  │
  │ acquisition complete
  ▼
USB
  │
  ▼
PC
```

In questa modalità:

- USB non deve sostenere il data rate istantaneo del sampler;
- il dispositivo acquisisce prima;
- trasferisce successivamente.

È la modalità più semplice e robusta per raggiungere sample rate elevate nella V1.

---

## 6.2 Streaming Mode

```text
PSSI
 │
 ▼
DMA
 │
 ▼
ping buffer ────┐
                ├──► USB FS ───► PC
pong buffer ────┘
```

In questa modalità deve valere:

```text
sustained_USB_payload_rate > acquisition_data_rate
```

Con USB Full-Speed il limite sarà molto più basso rispetto alla sample rate massima buffered.

### Target preliminare

Lo streaming continuo non è una feature prestazionale primaria della V1. Verrà comunque caratterizzato sperimentalmente.

Un ordine di grandezza ragionevole da verificare è:

```text
~0.5–1 MS/s
```

per 8 canali rappresentati con 1 byte per campione, a seconda dell'efficienza effettiva del trasferimento USB.

### Requisito V1

La modalità principale è:

```text
triggered / buffered capture
```

La modalità streaming è secondaria e non deve condizionare il raggiungimento del target di campionamento buffered.

---

# 7. Versione 1 — STM32H723

# 7.1 Obiettivo

La V1 deve dimostrare l'intera architettura del logic analyzer end-to-end:

```text
digital input
    ↓
sampling
    ↓
DMA
    ↓
buffer
    ↓
trigger
    ↓
USB
    ↓
PC
    ↓
waveform
    ↓
protocol decode
```

La V1 rappresenta la piattaforma con cui sviluppare soprattutto:

- architettura firmware;
- protocollo dispositivo-PC;
- gestione delle acquisizioni;
- formato dati;
- decoder;
- GUI;
- test automatici.

---

# 7.2 Hardware target

## MCU

```text
STM32H723ZG
```

Caratteristiche rilevanti:

- Arm Cortex-M7;
- frequenza CPU fino a 550 MHz;
- cache L1;
- RAM interna fino a 564 KB;
- PSSI;
- DMA;
- MDMA;
- FMC;
- USB OTG HS controller;
- USB DMA;
- ULPI per PHY USB HS esterno.

---

## Development board

Baseline iniziale:

```text
NUCLEO-H723ZG
```

Motivazioni:

- MCU già disponibile su scheda;
- ST-LINK integrato;
- accesso ai pin tramite Morpho/Zio;
- documentazione ufficiale;
- esempio PSSI DMA disponibile nello STM32CubeH7;
- ottima piattaforma per il bring-up.

### USB nella baseline V1

La NUCLEO-H723ZG dispone di collegamento USB OTG Full-Speed, che sarà utilizzato direttamente nella V1.

Non è richiesto alcun PHY ULPI esterno per la baseline.

Questo riduce:

- numero di segnali ad alta velocità;
- occupazione pin;
- conflitti di alternate function;
- complessità hardware;
- rischi di signal integrity;
- tempo necessario al bring-up.

L'eventuale USB High-Speed su STM32 potrà essere studiata come estensione separata, ma non costituisce requisito della V1.

---

# 7.3 Front-end V1

La prima versione non deve tentare immediatamente di supportare ogni standard logico.

Baseline:

```text
3.3 V digital logic
```

Il DUT e il logic analyzer devono condividere:

```text
GND
```

### Front-end minimo da valutare

```text
DUT
 │
 ├── series resistor
 │
 ├── optional ESD protection
 │
 ▼
STM32 input
```

È importante non considerare automaticamente gli ingressi del microcontrollore come equivalenti a quelli di uno strumento professionale.

La V1 può utilizzare un front-end semplificato, ma dovranno essere specificati chiaramente:

- tensione massima accettabile;
- soglia logica;
- impedenza;
- capacità parassita;
- assenza/presenza di protezione;
- eventuali pin 5-V tolerant;
- limiti elettrici.

Un front-end universale sarà parte integrante della V3.

---

# 8. Acquisition Engine V1

## 8.1 PSSI

PSSI significa:

```text
Parallel Synchronous Slave Interface
```

È una periferica dell'H723 capace di ricevere o trasmettere dati paralleli a 8 o 16 bit.

Per il logic analyzer:

```text
PSSI mode = receive
PSSI width = 8 bit
```

Mapping:

```text
PSSI_D0 = CH0
PSSI_D1 = CH1
...
PSSI_D7 = CH7
```

Ogni fronte del sample clock provoca l'acquisizione simultanea degli 8 bit.

---

## 8.2 PSSI è una periferica slave

Questo è un vincolo architetturale importante.

Il PSSI necessita del segnale:

```text
PSSI_PDCK
```

come clock di acquisizione.

Non deve quindi essere immaginato come una periferica che genera autonomamente il proprio sample clock.

Per il prototipo si prevede:

```text
STM32 timer / MCO
       │
       │ generated sample clock
       ▼
output GPIO
       │
       │ external jumper / PCB trace
       ▼
PSSI_PDCK
```

Alternative:

- timer output;
- MCO;
- clock generator esterno.

La soluzione definitiva dovrà essere scelta nella fase dedicata al sampling engine.

---

## 8.3 Sample clock target

Sample rate nominali iniziali:

```text
1 MS/s
2 MS/s
5 MS/s
10 MS/s
20 MS/s
25 MS/s
50 MS/s
```

Il PSSI dell'H723 supporta in ricezione un clock significativamente superiore a 50 MHz, quindi il target di 50 MS/s è compatibile con il limite nominale della periferica.

Questo non dimostra tuttavia che l'intero data path:

```text
PSSI → DMA → RAM
```

sia automaticamente lossless a 50 MS/s.

La prestazione deve essere verificata.

---

## 8.4 DMA

Il PSSI deve trasferire i campioni verso RAM tramite DMA.

```text
PSSI FIFO
    │
    ▼
DMA
    │
    ▼
AXI/AHB SRAM
```

Non utilizzare polling CPU per il percorso dei campioni.

### Requisito

Il DMA deve poter acquisire un'intera finestra senza perdita di campioni.

### Controlli richiesti

- overrun;
- DMA error;
- FIFO error;
- buffer overflow;
- sample count;
- sequence integrity.

---

# 9. Memory Architecture V1

La memoria dell'H723 non è un blocco unico.

Sono presenti differenti regioni SRAM con differenti accessibilità e caratteristiche.

La progettazione deve tenere conto di:

- AXI SRAM;
- AHB SRAM;
- DTCM;
- ITCM;
- DMA accessibility;
- bus contention;
- cache coherency.

---

## 9.1 DTCM

La DTCM è estremamente veloce per il Cortex-M7 ma non deve essere considerata automaticamente una destinazione valida per i normali DMA.

Per i buffer condivisi con DMA si useranno regioni appropriate, ad esempio AXI/AHB SRAM, dopo verifica della matrice bus dell'H723.

---

## 9.2 Cache coherency

Il Cortex-M7 dispone di D-cache.

Questo crea un problema classico:

```text
DMA writes RAM
     │
     ▼
RAM contains new data

CPU reads same address
     │
     ▼
CPU may see stale cached data
```

Analogamente:

```text
CPU writes buffer
     │
     ▼
data remains in cache

USB DMA reads RAM
     │
     ▼
USB DMA may see old data
```

Il progetto dovrà adottare una strategia esplicita.

Possibili strategie:

1. buffer DMA in regioni MPU non-cacheable;
2. cache clean/invalidate espliciti;
3. regioni shared/non-cacheable dedicate;
4. separazione fra buffer DMA e dati applicativi.

### Baseline consigliata

Per il bring-up:

```text
DMA buffers in explicitly non-cacheable MPU region
```

Successivamente si potranno benchmarkare strategie più aggressive.

---

# 10. Profondità di acquisizione

Con un campione da 1 byte:

```text
capture_time = buffer_size / sample_rate
```

Esempio con 256 KiB:

| Sample rate | Finestra approssimativa |
|---:|---:|
| 1 MS/s | 262 ms |
| 5 MS/s | 52 ms |
| 10 MS/s | 26 ms |
| 25 MS/s | 10.5 ms |
| 50 MS/s | 5.2 ms |

La RAM effettivamente allocabile al capture buffer sarà determinata durante l'implementazione.

La V1 può essere sviluppata in due sottostadi:

### V1-A

```text
internal RAM only
```

### V1-B opzionale

```text
external SDRAM via FMC
```

L'aggiunta di SDRAM esterna permetterebbe acquisizioni molto più profonde senza dover attendere la V2.

Tuttavia non deve bloccare il primo MVP.

---

# 11. Trigger Engine V1

## 11.1 Edge trigger

Trigger minimo obbligatorio:

```text
rising edge
falling edge
```

su un canale selezionabile.

Esempio:

```text
trigger_channel = CH3
trigger_edge    = RISING
```

---

## 11.2 Implementazione possibile su STM32

Un possibile meccanismo è:

```text
CHx
 │
 ├──────────────► PSSI sample stream
 │
 └──────────────► EXTI
                      │
                      ▼
                 trigger ISR
```

Al trigger:

1. registrare stato DMA;
2. registrare il buffer position stimato;
3. impostare lo stato `TRIGGERED`;
4. attendere il numero richiesto di campioni post-trigger;
5. arrestare acquisizione.

L'interrupt introduce latenza.

Quindi il trigger index definitivo potrà essere corretto software cercando l'edge effettivo all'interno dei campioni vicini alla posizione stimata.

Questo evita di confondere:

```text
interrupt arrival time
```

con:

```text
actual sampled edge
```

---

## 11.3 Pattern trigger

Esempio:

```text
mask  = 11110000
value = 10100000
```

condizione:

```text
(sample & mask) == value
```

Una comparazione CPU a 50 milioni di campioni al secondo non è un'architettura accettabile.

Quindi nella V1 si distinguono:

### Pattern analysis offline

supportato.

### Pattern trigger hardware real-time a sample rate massima

non requisito baseline della V1.

Potrà essere:

- sperimentato a frequenze inferiori;
- implementato con periferiche aggiuntive se possibile;
- trasferito alla V2 FPGA.

---

## 11.4 Protocol trigger

Esempi:

```text
UART byte == 0xA5
I2C address == 0x68
SPI word == 0x9F
```

Questi trigger non fanno parte del requisito minimo V1.

Sono candidati naturali per V2 FPGA.

---

# 12. Acquisition State Machine

Il firmware dovrà implementare una state machine esplicita.

```text
                ┌──────────────┐
                │     IDLE     │
                └──────┬───────┘
                       │ CONFIGURE
                       ▼
                ┌──────────────┐
                │  CONFIGURED  │
                └──────┬───────┘
                       │ ARM
                       ▼
                ┌──────────────┐
                │    ARMED     │
                └──────┬───────┘
                       │ start sampling
                       ▼
                ┌──────────────┐
                │ PRETRIGGER   │
                └──────┬───────┘
                       │ trigger
                       ▼
                ┌──────────────┐
                │ POSTTRIGGER  │
                └──────┬───────┘
                       │ complete
                       ▼
                ┌──────────────┐
                │   COMPLETE   │
                └──────┬───────┘
                       │ upload/reset
                       ▼
                     IDLE
```

Stati di errore:

```text
ERROR_DMA
ERROR_PSSI
ERROR_USB
ERROR_CONFIG
ERROR_OVERFLOW
```

---

# 13. Firmware Architecture V1

Struttura proposta:

```text
firmware/
│
├── app/
│   ├── main.c
│   ├── analyzer_app.c
│   └── analyzer_state_machine.c
│
├── acquisition/
│   ├── acquisition.c
│   ├── acquisition.h
│   ├── pssi_sampler.c
│   ├── sample_clock.c
│   ├── capture_buffer.c
│   └── trigger.c
│
├── transport/
│   ├── usb_device.c
│   ├── usb_bulk.c
│   ├── transport_protocol.c
│   └── packetizer.c
│
├── platform/
│   ├── clocks.c
│   ├── mpu.c
│   ├── cache.c
│   ├── gpio.c
│   └── error.c
│
├── diagnostics/
│   ├── counters.c
│   ├── profiler.c
│   └── telemetry.c
│
└── tests/
```

---

# 14. Firmware layering

## Layer 1 — BSP / platform

Responsabilità:

- system clock;
- pin muxing;
- GPIO;
- MPU;
- cache;
- IRQ;
- timestamp/debug timer;
- low-level hardware initialization.

---

## Layer 2 — Acquisition driver

Responsabilità:

- PSSI;
- sample clock;
- DMA;
- buffer;
- capture start;
- capture stop;
- overflow detection.

API concettuale:

```c
la_status_t la_acq_init(const la_acq_config_t *cfg);
la_status_t la_acq_arm(void);
la_status_t la_acq_stop(void);
la_status_t la_acq_get_status(la_acq_status_t *status);
```

---

## Layer 3 — Trigger

Responsabilità:

- trigger configuration;
- edge detection;
- trigger event;
- trigger buffer index;
- post-trigger remaining samples.

---

## Layer 4 — Transport

Responsabilità:

- USB endpoints;
- command reception;
- binary framing;
- packet sequence;
- capture upload;
- retry/error handling dove applicabile.

---

## Layer 5 — Application

Responsabilità:

- state machine;
- validazione configurazione;
- gestione comandi host;
- coordinamento acquisizione/USB.

---

# 15. USB Transport Protocol

Non utilizzare una seriale testuale come data path principale.

Il protocollo dovrà essere binario.

---

## 15.1 Command classes

Esempio:

```text
GET_DEVICE_INFO
GET_CAPABILITIES
SET_CONFIG
GET_CONFIG
ARM
STOP
GET_STATUS
READ_CAPTURE
START_STREAM
STOP_STREAM
RESET
```

---

## 15.2 Device information

Esempio concettuale:

```c
typedef struct
{
    uint16_t protocol_version;
    uint16_t fw_major;
    uint16_t fw_minor;

    uint8_t channel_count;
    uint8_t sample_width_bits;

    uint32_t max_sample_rate_hz;
    uint32_t max_capture_samples;

    uint32_t feature_flags;
} la_device_info_t;
```

---

## 15.3 Acquisition configuration

```c
typedef struct
{
    uint32_t sample_rate_hz;
    uint32_t sample_count;

    uint8_t channel_mask;

    uint8_t trigger_mode;
    uint8_t trigger_channel;
    uint8_t trigger_edge;

    uint8_t trigger_position_percent;
} la_capture_config_t;
```

---

## 15.4 Capture packet

Possibile header:

```c
typedef struct
{
    uint32_t magic;
    uint16_t protocol_version;
    uint16_t packet_type;

    uint32_t capture_id;
    uint32_t sequence_number;

    uint32_t sample_offset;
    uint32_t sample_count;

    uint32_t payload_size;
} la_packet_header_t;
```

Possibile aggiunta futura:

```text
CRC32
```

---

# 16. USB endpoint strategy

La V1 dovrà preferire endpoint **Bulk** per il trasferimento dei dati.

Motivazioni:

- affidabilità;
- throughput elevato;
- retry gestiti dal protocollo USB;
- adatti a trasferimenti massivi non isocroni.

Possibile separazione:

```text
EP OUT  : host → commands
EP IN   : device → responses/data
```

oppure endpoint separati per:

```text
control
capture data
telemetry
```

La scelta definitiva verrà fatta nella fase USB.

---

# 17. Host Software

Il software PC sarà sviluppato inizialmente in Python per ridurre il tempo di iterazione.

Architettura:

```text
host/
│
├── device/
│   ├── discovery.py
│   ├── transport.py
│   ├── protocol.py
│   └── device.py
│
├── capture/
│   ├── capture.py
│   ├── file_format.py
│   └── timeline.py
│
├── decoders/
│   ├── base.py
│   ├── uart.py
│   ├── spi.py
│   └── i2c.py
│
├── cli/
│   └── main.py
│
├── gui/
│   ├── main_window.py
│   ├── waveform.py
│   └── decode_overlay.py
│
└── tests/
```

---

# 18. Host development order

Ordine obbligatorio:

```text
1. transport library
2. CLI
3. capture file
4. decoder tests
5. waveform viewer
6. integrated GUI
```

La GUI non deve essere il primo elemento host sviluppato.

---

# 19. CLI

Esempio d'uso:

```bash
logic-analyzer capture \
    --rate 20M \
    --samples 200000 \
    --channels 0-7 \
    --trigger CH2:rising \
    --trigger-position 40 \
    --output capture.la
```

Possibile output:

```text
Logic Analyzer V1

Device: STM32H723
Firmware: 0.1.0

Sample rate:      20.000 MS/s
Channels:         CH0..CH7
Samples:          200000
Trigger:          CH2 rising
Trigger position: 40 %

Arming...
Triggered.
Capture complete.

Pre-trigger:      80000 samples
Post-trigger:     120000 samples

Transferring...
200000 / 200000 samples

Saved: capture.la
```

---

# 20. Capture file format

Il software dovrà poter salvare acquisizioni offline.

Informazioni minime:

```text
magic
format version
device identifier
firmware version
sample rate
channel count
channel mask
sample count
trigger index
trigger type
raw samples
metadata
```

Una soluzione iniziale semplice può usare:

```text
small binary header + raw sample payload
```

Successivamente si potrà utilizzare:

- chunked format;
- compression;
- JSON metadata + binary data;
- interoperabilità con sigrok.

---

# 21. Decoder Framework

Ogni decoder deve implementare un'interfaccia comune.

Concettualmente:

```python
class Decoder:
    def decode(self, capture, config):
        ...
```

Output comune:

```text
event
timestamp_start
timestamp_end
channel(s)
type
value
metadata
error flags
```

Questo permette alla GUI di visualizzare ogni decoder nello stesso modo.

---

# 22. UART Decoder

## Input

Configurazione:

```text
channel
baud rate
data bits
parity
stop bits
polarity
```

Baseline:

```text
8N1
```

---

## Algoritmo

1. trovare idle state;
2. cercare start edge;
3. verificare start bit;
4. calcolare bit period in samples;
5. campionare al centro di ogni data bit;
6. ricostruire byte LSB first;
7. verificare parity se presente;
8. verificare stop bit;
9. generare evento decoded.

Output:

```text
125.340 ms  0x48  'H'
125.427 ms  0x65  'e'
125.514 ms  0x6C  'l'
```

Errori:

```text
framing error
parity error
invalid stop bit
insufficient sample rate
```

---

# 23. SPI Decoder

Canali:

```text
CLK
MOSI
MISO
CS
```

Configurazione:

```text
CPOL
CPHA
bit order
bits per word
CS polarity
```

Modalità:

```text
Mode 0 = CPOL 0 / CPHA 0
Mode 1 = CPOL 0 / CPHA 1
Mode 2 = CPOL 1 / CPHA 0
Mode 3 = CPOL 1 / CPHA 1
```

Algoritmo:

1. identificare CS active;
2. trovare edge CLK corretti;
3. campionare MOSI/MISO;
4. accumulare bit;
5. ricostruire word;
6. terminare frame al CS inactive.

Output:

```text
Frame 17

MOSI: 9F 00 00 00
MISO: 00 EF 40 18
```

---

# 24. I²C Decoder

Canali:

```text
SCL
SDA
```

Eventi fondamentali:

```text
START
STOP
REPEATED START
ACK
NACK
```

START:

```text
SDA: 1 → 0
while
SCL = 1
```

STOP:

```text
SDA: 0 → 1
while
SCL = 1
```

I dati vengono interpretati durante gli edge previsti di SCL.

Output:

```text
START
ADDRESS 0x68 WRITE
ACK
DATA 0x10
ACK
RESTART
ADDRESS 0x68 READ
ACK
DATA 0x42
NACK
STOP
```

---

# 25. Waveform Viewer

La GUI dovrà fornire almeno:

- timeline;
- zoom;
- pan;
- cursori;
- misura Δt;
- frequenza derivata;
- enable/disable canali;
- rename canali;
- trigger marker;
- protocol overlay;
- decoded event list;
- configurazione decoder;
- salvataggio acquisizione;
- caricamento acquisizione.

Visualizzazione concettuale:

```text
CH0  ____████________████████________
CH1  ________██__██__________██______
CH2  ████████________████████________
CH3  ______████████__________________

              ▲
            TRIGGER

UART      [0x48] [0x65] [0x6C] [0x6C] [0x6F]
```

---

# 26. Testing Strategy

Il progetto deve essere altamente testabile.

Si prevede un secondo generatore di segnali.

```text
┌─────────────────┐
│ Signal Generator│
│ MCU / FPGA      │
└────────┬────────┘
         │
         │ UART/SPI/I2C/pattern
         ▼
┌─────────────────┐
│ Logic Analyzer  │
└────────┬────────┘
         │
         ▼
        PC
```

---

## 26.1 Test UART

Generare:

```text
baud = 115200
payload = 55 AA 12 34
```

Test:

```text
capture
decode
assert decoded == [0x55, 0xAA, 0x12, 0x34]
```

Variare:

- baud;
- parity;
- stop bits;
- payload;
- idle time;
- framing errors.

---

## 26.2 Test SPI

Generare:

- Mode 0;
- Mode 1;
- Mode 2;
- Mode 3;
- frequenze differenti;
- payload noti;
- CS gaps.

Verificare MOSI e MISO.

---

## 26.3 Test I²C

Generare:

- START;
- WRITE;
- READ;
- ACK;
- NACK;
- RESTART;
- STOP;
- differenti clock rate.

---

## 26.4 Pattern test

Pattern digitale:

```text
00
FF
55
AA
01
02
04
08
10
20
40
80
```

Questo permette di verificare:

- mapping canali;
- bit order;
- perdita campioni;
- phase alignment;
- trigger.

---

# 27. Performance Testing

Ogni sample rate dovrà essere qualificata.

Per:

```text
1
2
5
10
20
25
50 MS/s
```

misurare:

- perdita campioni;
- DMA errors;
- PSSI overrun;
- durata acquisizione;
- CPU load;
- bus load;
- buffer integrity;
- trigger precision;
- USB upload throughput.

---

## 27.1 Sequence pattern

Un modo utile per verificare la continuità è utilizzare un generatore digitale con pattern noto e periodico.

La cattura viene confrontata con il pattern atteso.

Qualunque salto indica:

```text
lost sample
duplicated sample
bit mapping error
timing error
```

---

# 28. USB Benchmark

Prima dell'integrazione finale del logic analyzer, il sottosistema USB dovrà essere benchmarkato indipendentemente.

Test:

```text
RAM buffer
   │
   ▼
USB DMA
   │
   ▼
PC
```

senza PSSI.

Misurare sustained payload rate per:

```text
1 s
10 s
60 s
```

e differenti packet/buffer sizes.

Metriche:

```text
average MB/s
minimum MB/s
maximum MB/s
host CPU usage
device CPU usage
errors
stalls
```

Solo dopo questo benchmark verrà definito il limite ufficiale dello streaming V1.

---

# 29. V1 Milestones

---

## M0 — Requirements Baseline

### Obiettivo

Congelare il comportamento atteso della V1.

### Output

```text
LA-V1-SRS.md
```

### Deve definire

- channel count;
- sample rates;
- input voltage;
- trigger modes;
- capture depth;
- USB requirements;
- host OS;
- supported protocols;
- error handling;
- performance targets.

### Exit criteria

Tutti i requisiti V1 hanno:

```text
ID
description
rationale
verification method
priority
```

---

## M1 — Hardware Bring-up STM32H723

### Obiettivo

Ottenere una piattaforma STM32 stabile.

### Attività

- NUCLEO-H723ZG;
- STM32CubeIDE/project setup;
- CPU 550 MHz;
- HCLK;
- clock tree;
- UART debug;
- GPIO test;
- DWT/timer profiling;
- cache;
- MPU;
- linker memory map.

### Exit criteria

- firmware stabile;
- clock verificati;
- GPIO funzionanti;
- profiling disponibile;
- memory regions comprese e documentate.

---

## M2 — PSSI Sampling Prototype

### Obiettivo

Acquisire 8 GPIO simultaneamente.

### Primo test

```text
1 MS/s
```

### Evoluzione

```text
1 → 2 → 5 → 10 → 20 → 25 → 50 MS/s
```

### Attività

- pin mapping;
- PSSI RX 8 bit;
- sample clock generation;
- PDCK loopback;
- DMA;
- capture buffer;
- pattern generator.

### Exit criteria

Cattura lossless verificata almeno a:

```text
10 MS/s
```

prima di procedere all'ottimizzazione.

---

## M3 — High-Speed Acquisition Qualification

### Obiettivo

Determinare il limite reale:

```text
PSSI → DMA → internal RAM
```

### Test

- 20 MS/s;
- 25 MS/s;
- 50 MS/s;
- burst differenti;
- buffer sizes differenti;
- cache configuration.

### Exit criteria

Definizione di:

```text
LA_V1_MAX_BUFFERED_SAMPLE_RATE
```

basata su dati sperimentali.

Target:

```text
50 MS/s
```

---

## M4 — Circular Buffer + Trigger

### Obiettivo

Implementare:

- continuous circular acquisition;
- rising edge trigger;
- falling edge trigger;
- configurable trigger position.

### Trigger position

```text
0–100%
```

### Exit criteria

Acquisizione corretta per:

```text
0%
25%
50%
75%
90%
```

con trigger marker correttamente ricostruito.

---

## M5 — USB 2.0 Full-Speed

### Obiettivo

Utilizzare il PHY integrato dello STM32H723 per ottenere una connessione USB device stabile e semplice da portare in bring-up.

### Attività

- USB Full-Speed device mode;
- configurazione D+ / D-;
- descriptors;
- Bulk endpoints;
- buffer management;
- host test utility;
- trasferimento delle acquisizioni buffered.

### Exit criteria

- enumerazione stabile come dispositivo Full-Speed;
- trasferimento Bulk affidabile;
- assenza di corruzione dei dati;
- gestione corretta degli errori.

---

## M6 — USB Throughput Benchmark

### Obiettivo

Caratterizzare il throughput reale Full-Speed.

### Test

```text
RAM buffer
   │
   ▼
USB
   │
   ▼
PC
```

senza PSSI.

Misurare:

- sustained payload rate;
- packet/buffer size ottimali;
- CPU load;
- errori;
- stall;
- affidabilità per trasferimenti lunghi.

### Exit criteria

Definire:

```text
MAX_USB_FS_PAYLOAD_MBPS
MAX_FS_STREAMING_SAMPLE_RATE
```

Lo streaming continuo rimane una feature secondaria. Il requisito principale V1 è il download affidabile di acquisizioni buffered.

---

## M7 — Device Protocol + CLI

### Obiettivo

Controllare completamente il logic analyzer senza GUI.

### Funzioni

- device discovery;
- get info;
- configure;
- arm;
- status;
- capture;
- download;
- save.

### Exit criteria

Un'intera acquisizione può essere eseguita da terminale.

---

## M8 — UART Decoder

### Obiettivo

Decodifica offline robusta.

### Exit criteria

Suite di test automatica con payload noti.

---

## M9 — SPI Decoder

### Obiettivo

Supportare almeno:

```text
Mode 0–3
MOSI
MISO
CS
LSB/MSB first
```

---

## M10 — I²C Decoder

### Obiettivo

Supportare:

```text
START
STOP
RESTART
address
R/W
ACK
NACK
data
```

---

## M11 — Waveform Viewer

### Obiettivo

Visualizzazione interattiva delle acquisizioni.

### Exit criteria

È possibile:

- zoomare;
- scorrere;
- vedere trigger;
- misurare Δt;
- configurare decoder;
- vedere decoded overlay.

---

## M12 — V1 Integration & Qualification

### Obiettivo

Congelare Logic Analyzer V1.

### Qualification matrix

| Feature | Required |
|---|---|
| 8 channels | Yes |
| Buffered acquisition | Yes |
| 50 MS/s target | Yes, if validated |
| USB Full-Speed | Yes |
| Rising/falling trigger | Yes |
| Pre/post-trigger | Yes |
| UART decode | Yes |
| SPI decode | Yes |
| I²C decode | Yes |
| CLI | Yes |
| GUI | Yes |
| Continuous streaming | Secondary / best validated FS rate |
| Protocol trigger | No |
| 16 channels | No |

---

# 30. Versione 2 — FPGA

La V2 non deve essere un nuovo progetto completamente scollegato dalla V1.

Deve mantenere compatibile quanto possibile:

- host protocol;
- capture format;
- decoder API;
- GUI;
- user workflow.

Il PC idealmente dovrà vedere:

```text
Logic Analyzer Device
```

indipendentemente dal fatto che l'acquisition engine sia STM32 o FPGA.

---

# 31. Obiettivi V2

Target preliminare:

```text
8–16 channels
100–200+ MS/s
hardware trigger
deep capture
optional RLE
```

Possibile architettura:

```text
Digital inputs
      │
      ▼
┌───────────────┐
│ Input FF bank │
└───────┬───────┘
        │
        ▼
┌───────────────┐
│ Trigger Engine│
└───────┬───────┘
        │
        ▼
┌───────────────┐
│ FIFO / BRAM   │
└───────┬───────┘
        │
        ▼
┌───────────────┐
│ DDR / SDRAM   │
└───────┬───────┘
        │
        ▼
┌───────────────┐
│ USB Interface │
└───────────────┘
```

---

# 32. FPGA modules

Struttura RTL proposta:

```text
rtl/
│
├── top/
│   └── logic_analyzer_top.sv
│
├── sampler/
│   ├── input_sampler.sv
│   └── sample_clock.sv
│
├── trigger/
│   ├── edge_trigger.sv
│   ├── pattern_trigger.sv
│   └── protocol_trigger.sv
│
├── buffer/
│   ├── capture_fifo.sv
│   ├── circular_buffer.sv
│   └── memory_controller.sv
│
├── compression/
│   └── rle_encoder.sv
│
├── control/
│   └── register_bank.sv
│
└── interface/
    └── usb_stream.sv
```

---

# 33. Trigger V2

La FPGA rende possibile valutare ogni campione in hardware.

Esempio:

```verilog
trigger <= ((sample & mask) == value);
```

a ogni sample clock.

Trigger previsti:

- edge;
- level;
- masked pattern;
- sequence;
- pulse width;
- glitch;
- protocol-aware trigger.

---

# 34. RLE Compression

I segnali digitali spesso rimangono invariati per molti campioni.

Esempio raw:

```text
00 00 00 00 00 00 00 00 FF FF FF FF
```

Run-Length Encoding:

```text
00 × 8
FF × 4
```

La FPGA può comprimere i dati prima di trasferirli.

Vantaggi:

- buffer virtualmente più profondo;
- minore banda USB;
- maggiore durata acquisizione.

Svantaggio:

- worst-case incomprimibile;
- maggiore complessità;
- decoder host deve supportare timestamps/run length.

RLE sarà quindi una feature V2, non una dipendenza della V1.

---

# 35. USB nella V2

Con 8 canali:

```text
100 MS/s = 100 MB/s raw
```

Con 16 canali:

```text
100 MS/s = 200 MB/s raw
```

USB 2.0 HS non è più sufficiente come streaming raw generale.

La V2 dovrà quindi valutare:

- USB 3.x;
- FX3/FX3S;
- controller USB dedicato;
- FPGA con transceiver appropriati;
- PCIe in una fase molto più futura.

---

# 36. Versione 3 — Custom Board

La V3 rappresenta la trasformazione del prototipo in uno strumento integrato.

---

# 37. Macro-architettura V3

```text
          PROBES
            │
            ▼
┌─────────────────────────┐
│ Input Protection        │
│                         │
│ ESD                     │
│ series resistance       │
│ overvoltage protection  │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│ Input Buffer / Level    │
│                         │
│ high impedance          │
│ selectable threshold    │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│ FPGA                    │
│                         │
│ sampler                 │
│ trigger                 │
│ FIFO                    │
│ compression             │
└──────┬──────────┬───────┘
       │          │
       ▼          ▼
   SDRAM/DDR     MCU
                  │
                  ▼
            USB 3.x PHY
                  │
                  ▼
                 PC
```

---

# 38. Front-end V3

Obiettivi da valutare:

```text
1.8 V logic
2.5 V logic
3.3 V logic
5 V logic
```

Possibili feature:

- input buffer;
- programmable threshold;
- comparators;
- protection network;
- probe ground;
- matched input delay;
- termination options;
- ESD.

Il front-end avrà un documento separato perché coinvolge:

- elettricità;
- signal integrity;
- protezione;
- loading del DUT;
- propagation delay;
- channel-to-channel skew.

---

# 39. PCB V3

Aspetti critici:

- USB differential routing;
- controlled impedance;
- ULPI / USB3 high-speed bus;
- FPGA power rails;
- decoupling;
- clock integrity;
- DDR routing;
- input channel matching;
- ground return;
- ESD placement;
- connector mechanics.

Questa fase dovrà essere affrontata soltanto dopo la validazione completa delle architetture precedenti.

---

# 40. Software compatibility across versions

Uno degli obiettivi principali è mantenere il software PC comune.

Il protocollo deve avere capability discovery.

Esempio:

```text
GET_CAPABILITIES
```

V1 risponde:

```text
channels       = 8
max_rate       = 50 MS/s
edge_trigger   = yes
pattern_trigger= no
compression    = no
```

V2:

```text
channels       = 16
max_rate       = 200 MS/s
edge_trigger   = yes
pattern_trigger= yes
compression    = RLE
```

La GUI abilita/disabilita automaticamente le funzioni.

---

# 41. Versioning

Servono versioni indipendenti per:

```text
hardware revision
firmware version
protocol version
capture file version
host software version
```

Esempio:

```text
HW_REV          = V1.0
FW_VERSION      = 0.4.2
PROTOCOL        = 1.1
CAPTURE_FORMAT  = 1.0
HOST_VERSION    = 0.6.0
```

Il protocollo deve prevedere backward compatibility dove ragionevole.

---

# 42. Error Model

Errori dispositivo:

```text
ERR_INVALID_CONFIG
ERR_UNSUPPORTED_RATE
ERR_INVALID_TRIGGER
ERR_DMA
ERR_PSSI_OVERRUN
ERR_BUFFER_OVERFLOW
ERR_USB
ERR_BUSY
ERR_NOT_ARMED
ERR_CAPTURE_NOT_READY
```

Ogni errore deve essere:

- identificabile;
- loggabile;
- riproducibile;
- riportabile al PC.

---

# 43. Diagnostics

Contatori runtime:

```text
captures_total
captures_failed
pssi_overruns
dma_errors
usb_tx_bytes
usb_errors
trigger_count
max_usb_queue_depth
```

Servono sia per debug sia per qualification.

---

# 44. Logging

Il firmware non deve stampare continuamente via printf durante l'acquisizione ad alta velocità.

Prevedere livelli:

```text
ERROR
WARN
INFO
DEBUG
TRACE
```

e possibilità di disabilitare completamente il logging nel performance build.

---

# 45. Build configurations

Almeno:

```text
Debug
Release
Performance
Test
```

### Debug

- assertions;
- log;
- symbol;
- diagnostica.

### Performance

- logging ridotto;
- ottimizzazioni;
- timing benchmark.

---

# 46. Repository

Struttura complessiva:

```text
logic-analyzer/
│
├── docs/
│   ├── master/
│   ├── requirements/
│   ├── architecture/
│   ├── firmware/
│   ├── hardware/
│   ├── usb/
│   ├── decoders/
│   ├── gui/
│   └── verification/
│
├── firmware/
│
├── host/
│
├── fpga/
│
├── hardware/
│   ├── prototype/
│   └── custom_board/
│
├── test/
│   ├── generators/
│   ├── captures/
│   ├── vectors/
│   └── automation/
│
└── tools/
```

---

# 47. Documentation strategy

Il presente documento è:

```text
LA-MASTER-001
```

I successivi documenti suggeriti sono:

```text
LA-V1-SRS-001
Logic Analyzer V1 — System Requirements Specification

LA-V1-ARCH-001
STM32H723 Logic Analyzer — Architecture

LA-V1-HW-001
V1 Prototype Hardware and Pin Mapping

LA-V1-CLOCK-001
Sampling Clock and PSSI Design

LA-V1-DMA-001
PSSI DMA and Memory Architecture

LA-V1-TRG-001
Trigger and Pre/Post-Trigger Design

LA-V1-USB-001
USB 2.0 Full-Speed Architecture

LA-HOST-PROTO-001
Device-to-PC Binary Protocol

LA-HOST-CLI-001
Host CLI Architecture

LA-DEC-UART-001
UART Decoder Design

LA-DEC-SPI-001
SPI Decoder Design

LA-DEC-I2C-001
I2C Decoder Design

LA-GUI-001
Waveform Viewer Architecture

LA-VER-001
Verification and Qualification Plan

LA-V2-FPGA-001
FPGA Logic Analyzer Architecture

LA-V3-HW-001
Custom Board Architecture
```

---

# 48. Suggested detailed phase order

La sequenza completa è:

```text
PHASE 0
Requirements
    │
    ▼
PHASE 1
STM32 bring-up
    │
    ▼
PHASE 2
Sampling clock
    │
    ▼
PHASE 3
PSSI + DMA
    │
    ▼
PHASE 4
Acquisition performance
    │
    ▼
PHASE 5
Circular buffer
    │
    ▼
PHASE 6
Trigger
    │
    ▼
PHASE 7
USB Full-Speed
    │
    ▼
PHASE 8
USB benchmark
    │
    ▼
PHASE 9
Binary protocol
    │
    ▼
PHASE 10
PC CLI
    │
    ▼
PHASE 11
Capture file
    │
    ▼
PHASE 12
UART decoder
    │
    ▼
PHASE 13
SPI decoder
    │
    ▼
PHASE 14
I2C decoder
    │
    ▼
PHASE 15
Waveform GUI
    │
    ▼
PHASE 16
Integration
    │
    ▼
PHASE 17
Qualification
    │
    ▼
V1 COMPLETE
    │
    ▼
FPGA V2
    │
    ▼
CUSTOM BOARD V3
```

---

# 49. Definition of Done V1

La V1 può essere dichiarata completa quando:

## Hardware

- STM32H723 operativo;
- 8 ingressi digitali utilizzabili;
- sample clock stabile;
- PSSI acquisition stabile;
- USB Full-Speed stabile tramite PHY integrato.

## Acquisition

- sample rate configurabile;
- acquisition lossless fino al massimo valore qualificato;
- circular buffer;
- trigger rising/falling;
- pre/post-trigger configurabile.

## USB

- device enumeration;
- Bulk transfer;
- protocol binary;
- error handling;
- throughput Full-Speed caratterizzato.

## Host

- device discovery;
- configure;
- arm;
- capture;
- save;
- load.

## Decoding

- UART;
- SPI;
- I²C.

## GUI

- waveform;
- zoom;
- pan;
- cursor;
- trigger marker;
- decode overlays.

## Verification

- test vectors automatici;
- performance report;
- known limitations documentate.

---

# 50. Requirements preliminari V1

Questi requisiti saranno formalizzati in un documento SRS dedicato.

| ID | Requirement |
|---|---|
| LA-REQ-001 | Il dispositivo deve acquisire almeno 8 canali digitali simultaneamente. |
| LA-REQ-002 | Ogni campione a 8 canali deve poter essere rappresentato con un byte. |
| LA-REQ-003 | Il sistema deve supportare sample rate configurabili. |
| LA-REQ-004 | Il target massimo iniziale deve essere 50 MS/s in buffered mode. |
| LA-REQ-005 | Il limite massimo qualificato deve essere determinato mediante test lossless. |
| LA-REQ-006 | Il data path di acquisizione non deve dipendere da polling CPU per campione. |
| LA-REQ-007 | Il dispositivo deve utilizzare DMA per il trasferimento PSSI→RAM. |
| LA-REQ-008 | Il dispositivo deve supportare trigger rising edge. |
| LA-REQ-009 | Il dispositivo deve supportare trigger falling edge. |
| LA-REQ-010 | Il dispositivo deve supportare pre-trigger. |
| LA-REQ-011 | Il dispositivo deve supportare post-trigger. |
| LA-REQ-012 | La posizione del trigger deve essere configurabile. |
| LA-REQ-013 | Il dispositivo V1 deve supportare USB 2.0 Full-Speed utilizzando il PHY integrato dello STM32H723. |
| LA-REQ-014 | USB High-Speed tramite PHY ULPI esterno non è requisito della baseline V1 e potrà essere valutato come estensione opzionale. |
| LA-REQ-015 | Il trasporto dati deve utilizzare un protocollo binario. |
| LA-REQ-016 | Il PC deve poter configurare e avviare una cattura senza GUI. |
| LA-REQ-017 | Il software deve poter salvare e ricaricare una cattura. |
| LA-REQ-018 | Il software host deve includere decoder UART. |
| LA-REQ-019 | Il software host deve includere decoder SPI. |
| LA-REQ-020 | Il software host deve includere decoder I²C. |
| LA-REQ-021 | I decoder devono operare sui campioni raw e non dipendere dall'hardware STM32. |
| LA-REQ-022 | Gli errori DMA/PSSI/USB devono essere rilevabili e riportabili. |
| LA-REQ-023 | La GUI deve visualizzare un trigger marker. |
| LA-REQ-024 | La GUI deve supportare zoom e pan. |
| LA-REQ-025 | Il software deve poter eseguire test automatici dei decoder. |
| LA-REQ-026 | Il massimo streaming rate USB Full-Speed deve essere determinato sperimentalmente e non deve limitare il target buffered della V1. |
| LA-REQ-027 | La V1 deve essere compatibile con una futura sostituzione dell'acquisition engine con FPGA. |
| LA-REQ-028 | Il protocollo host deve supportare capability discovery. |
| LA-REQ-029 | Versione firmware e protocollo devono essere interrogabili dal PC. |
| LA-REQ-030 | Il firmware deve esporre diagnostica sufficiente a identificare sample loss o overflow. |

---

# 51. Open Questions

Le seguenti decisioni NON sono ancora congelate.

## Hardware

- mapping definitivo PSSI;
- mapping del sample clock / PSSI_PDCK;
- compatibilità dei pin PSSI con USB FS, debug e periferiche ausiliarie;
- front-end input V1;
- eventuale SDRAM esterna;
- eventuale PHY ULPI soltanto se si deciderà di sperimentare USB High-Speed come estensione non baseline.

## Sampling

- timer o MCO per PDCK;
- sample clock jitter;
- 50 MS/s qualification;
- massimo numero di canali con PSSI V1;
- configurazione FIFO/DMA ottimale.

## Trigger

- meccanismo esatto per correlare EXTI e DMA pointer;
- precisione trigger;
- eventuale pattern trigger V1.

## USB

- stack ST USB Device classico o USBX;
- endpoint layout;
- buffer size;
- sustained Full-Speed throughput;
- eventuale studio futuro di USB High-Speed/ULPI come estensione non baseline.

## Host

- libusb / pyusb / backend alternativo;
- Windows driver strategy;
- formato capture;
- framework GUI;
- possibilità di supportare Linux/macOS.

---

# 52. Technical Risks

## RISK-001 — PSSI sustained DMA bandwidth

**Rischio:** il limite pratico PSSI→RAM potrebbe essere inferiore al target.

**Mitigazione:**

- benchmark progressivo;
- memory placement;
- DMA tuning;
- bus analysis.

---

## RISK-002 — USB Full-Speed throughput

**Rischio:** throughput reale insufficiente per streaming continuo a sample rate elevate.

**Impatto:** basso sulla modalità buffered, alto sullo streaming continuo.

**Mitigazione:**

- buffered capture come modalità primaria;
- benchmark separato;
- Bulk transfers;
- ping-pong buffers;
- trasferimento successivo alla cattura;
- passaggio a USB 3.x nella V2 quando la banda diventa requisito primario.

---

## RISK-003 — Pin multiplexing

**Rischio:** PSSI, sample clock, debug e periferiche ausiliarie possono competere per alternate function sul package disponibile.

**Mitigazione:**

- pin mapping study prima del cablaggio definitivo;
- verificare package LQFP144;
- usare alternate pins quando disponibili;
- mantenere USB V1 su D+/D- evitando l'occupazione dei numerosi pin ULPI.

---

## RISK-004 — Cache coherency

**Rischio:** campioni corrotti o vecchi a causa di D-cache.

**Mitigazione:**

- MPU non-cacheable DMA buffer;
- cache maintenance;
- tests con pattern deterministici.

---

## RISK-005 — Trigger latency

**Rischio:** l'ISR EXTI non identifica esattamente il sample di trigger.

**Mitigazione:**

- registrare posizione DMA;
- cercare l'edge reale nei campioni;
- misurare jitter/latency;
- FPGA trigger in V2.

---

## RISK-006 — Front-end loading

**Rischio:** collegamento diretto degli ingressi altera o danneggia il DUT/analyzer.

**Mitigazione:**

- definire limiti V1;
- series resistors;
- protezione;
- front-end dedicato in V3.

---

# 53. Performance Targets — Baseline

## V1

```text
Channels            8
Target max sample    50 MS/s buffered
Streaming            best qualified rate
USB                  2.0 Full-Speed
Transfer model        buffered-first
Trigger              edge
Pre-trigger          yes
Post-trigger         yes
Decoder              UART/SPI/I2C
```

## V2

```text
Channels            8–16+
Target sample       100–200+ MS/s
Streaming           USB3-class target
Trigger             hardware edge/pattern/protocol
Compression         optional RLE
Memory              BRAM + external RAM
```

## V3

```text
Integrated PCB
protected front-end
multi-voltage input support
FPGA + memory + MCU/USB
USB-C
production-style enclosure
```

---

# 54. Why the three-version strategy is useful

La scelta STM32 → FPGA → custom board riduce il rischio.

Se si iniziasse direttamente da una PCB FPGA complessa, sarebbe necessario risolvere contemporaneamente:

- PCB high speed;
- FPGA;
- clock;
- DDR;
- USB;
- firmware;
- driver;
- GUI;
- decoders.

La V1 permette invece di validare prima:

```text
user workflow
data format
trigger semantics
protocol
decoder architecture
GUI
test infrastructure
```

La V2 sostituisce soltanto il data acquisition engine.

La V3 integra ciò che è già stato verificato.

---

# 55. Final Project Evolution

```text
┌─────────────────────────────────────────────────────┐
│ V1 — STM32H723                                     │
│                                                     │
│ Learn + validate complete system architecture       │
│                                                     │
│ 8 ch / up to ~50 MS/s target / USB2 FS             │
└───────────────────────┬─────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────┐
│ V2 — FPGA                                           │
│                                                     │
│ Increase deterministic parallel acquisition         │
│                                                     │
│ 8–16 ch / 100–200+ MS/s / advanced triggers        │
└───────────────────────┬─────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────┐
│ V3 — CUSTOM BOARD                                   │
│                                                     │
│ Integrated instrument                              │
│                                                     │
│ front-end + FPGA + RAM + USB + protection          │
└─────────────────────────────────────────────────────┘
```

---

# 56. Immediate next documents

L'ordine consigliato per i prossimi documenti è:

1. **LA-V1-SRS-001 — System Requirements Specification**
2. **LA-V1-HW-001 — STM32H723 Prototype Hardware**
3. **LA-V1-CLOCK-001 — Sampling Clock and PSSI**
4. **LA-V1-DMA-001 — DMA and Memory Architecture**
5. **LA-V1-TRG-001 — Trigger / Pre-trigger / Post-trigger**
6. **LA-V1-USB-001 — USB 2.0 Full-Speed Architecture**
7. **LA-HOST-PROTO-001 — Device-PC Protocol**
8. **LA-HOST-CLI-001 — Host CLI**
9. **LA-DEC-UART-001**
10. **LA-DEC-SPI-001**
11. **LA-DEC-I2C-001**
12. **LA-GUI-001**
13. **LA-VER-001 — Verification Plan**

Il primo documento da sviluppare dovrebbe essere l'SRS, perché ogni scelta hardware e software successiva dovrebbe essere riconducibile a un requisito verificabile.

---

# 57. References

Documentazione ufficiale primaria da utilizzare durante il progetto:

1. **STMicroelectronics — STM32H723xE/G Datasheet (DS13313)**  
   https://www.st.com/resource/en/datasheet/stm32h723zg.pdf

2. **STMicroelectronics — STM32H723/733, STM32H725/735, STM32H730 Reference Manual (RM0468)**  
   https://www.st.com/resource/en/reference_manual/dm00603761.pdf

3. **STMicroelectronics — NUCLEO-H723ZG**  
   https://www.st.com/en/evaluation-tools/nucleo-h723zg.html

4. **STMicroelectronics — STM32H7 Nucleo-144 User Manual (UM2407)**  
   https://www.st.com/resource/en/user_manual/um2407-stm32h7-nucleo144-board-stmicroelectronics.pdf

5. **STMicroelectronics — Level 1 cache on STM32F7/H7 (AN4839)**  
   https://www.st.com/resource/en/application_note/an4839-level-1-cache-on-stm32f7-series-and-stm32h7-series-stmicroelectronics.pdf

6. **STMicroelectronics — PSSI configuration example / STM32H723**  
   Consultare inoltre gli esempi `PSSI_Transmit_Receive_DMA` inclusi in STM32CubeH7.

7. **USB 2.0 Specification / USB-IF**  
   Da utilizzare come riferimento per endpoint, transfer type, frame/microframe e High-Speed operation.

---

# 58. Baseline statement

Alla versione 0.1 del presente documento vengono congelate le seguenti scelte di alto livello:

```text
PROJECT
USB Logic Analyzer

VERSION 1
STM32H723
8 channels
PSSI + DMA
target 50 MS/s buffered
USB 2.0 Full-Speed via internal PHY
buffered capture as primary transfer model
edge trigger
pre/post trigger
PC-side UART/SPI/I2C decoding

VERSION 2
FPGA acquisition engine
higher channel count
100–200+ MS/s target
advanced hardware trigger
possible USB 3.x
possible RLE

VERSION 3
custom integrated PCB
protected front-end
FPGA/MCU/memory/USB
production-style instrument
```

Le prestazioni indicate come **target** non sono considerate garantite fino alla conclusione delle relative attività di benchmark e qualification.

---


# 59. Change Log

## Version 0.2 — 25 August 2026

Modifiche principali rispetto alla versione 0.1:

- USB 2.0 High-Speed tramite PHY ULPI esterno rimosso dai requisiti baseline della V1;
- V1 aggiornata a USB 2.0 Full-Speed usando il PHY integrato dello STM32H723;
- buffered capture definita esplicitamente come modalità primaria;
- target di acquisizione buffered mantenuto a 50 MS/s, soggetto a qualification;
- streaming continuo riclassificato come feature secondaria e limitata dalla banda USB Full-Speed;
- milestone USB aggiornata per il bring-up Full-Speed;
- benchmark USB aggiornato al throughput Full-Speed;
- requisiti LA-REQ-013, LA-REQ-014 e LA-REQ-026 aggiornati;
- rischi USB e pin multiplexing aggiornati;
- USB High-Speed/ULPI mantenuto solo come possibile estensione sperimentale;
- V2 confermata come punto naturale per il salto di banda verso FPGA + USB 3.x.

---

**Fine documento — LA-MASTER-001 v0.2**
