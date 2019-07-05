# pic-notes
Appunti del corso di microcontrollori, Politecnico di Milano, 2018-19

<!-- TOC depthFrom:2 depthTo:6 withLinks:1 updateOnSave:1 orderedList:1 -->

1. [GPIO SETUP](#gpio-setup)
2. [APPUNTI TIMER0 LEZIONE 2](#appunti-timer0-lezione-2)
3. [APPUNTI LCD LEZIONE 2](#appunti-lcd-lezione-2)
4. [APPUNTI LEZIONE 3 ( NOTA BENE PER ORALE )](#appunti-lezione-3-nota-bene-per-orale-)
5. [APPUNTI SONAR (CCP) LEZIONE 4](#appunti-sonar-ccp-lezione-4)
6. [APPUNTI ADC LEZIONE 6](#appunti-adc-lezione-6)
7. [APPUNTI SUL PWM LEZIONE 7](#appunti-sul-pwm-lezione-7)
8. [QUALI ISTRUZIONI AFFLIGGONO LO STATUS?](#quali-istruzioni-affliggono-lo-status)
9. [COS’È LO STATUS?](#cos-lo-status)
10. [SPIEGAZIONE ALCUNE ISTRUZIONI ASM (RIVEDERE GOTO E BRA CON PIC16 PIC18)](#spiegazione-alcune-istruzioni-asm-rivedere-goto-e-bra-con-pic16-pic18)
11. [DIRETTIVE ASM](#direttive-asm)

<!-- /TOC -->

## GPIO SETUP

NB: pin settati in input per default. Quando si setta un digital out scrivere sempre LATx=0 per reset.

Digital output tris=0

Digital input 		ansel=0 	tris=1. 	(ANSEL=0 -> Buffer digitali in ingresso accesi)

Analog input 		ansel=1 	tris=1.

**NB**: se dei pin non vengono definitivamente utilizzati nel progetto, vengono settati come output e poi il **LAT viene impostato a 0**. Questo per prevenire cambi della porta che comportano consumo dovuto alla potenza dinamica di switching dell’ingresso.
Ogni port ha diversi registri:

- **TRIS** (input=1/output=0, input di default al reset del uC),
- **PORT**
 (legge il livello logico sul pin fisico, non definito al reset, LAT (output latch HI=1, LOW=0, non definito al reset),
- **ANSEL**(input analogico, 1 per spegnere il digital buffer, 0 per tenerlo acceso, tutto a 1 di default)

**EASYPIC BOARD SETUP ( usi tipici delle PORTx )**
- **PORTA**: usata per aggiungere pulsanti e per fare polling.
- **PORTB**: quasi interamente dedicata al LCD, attenzione che potrebbero venire usati RB6 e RB7 come pulsanti per interrupt on change (IOCB). **NB**: RB6 e RB7 vanno fuori uso con il debugger ON.
- **PORTC**: poco spesso usata per altri scopi se non per il modulo sonar.
- **PORTD**: spesso usata per operazioni con i led (tutta la porta).
- **PORTE**: usata per un PWM, PWM software o per dei led di segnalazione

## APPUNTI TIMER0 LEZIONE 2
Il TIMER0 è per definizione un contatore che subisce un overflow dopo un tempo definito dall'utente attraverso setup dei registri dedicati.
Si parte da un oscillatore esterno o interno. Mediante un multiplexer possiamo la frequenza di incremento del contatore. **la nostra frequenza tipica è Fosc/4 in cui, grazie al PLL, Fosc è 32MHz (8MHz di default)**.

**T0CKI** pin non lo useremo mai, è il pin per comunicare con l'esterno che se abilitato come sorgente fornisce un impulso edge sensitive in base alla impostazione selezionata con T0SE (variazione sul rising/falling edge).

Successivamente alla selezione della sorgente c'è la scelta del sì o no prescaler, **il prescaler è un divisore di frequenza**. Il modo più facile per creare un prescaler è fare un contatore. All'interno del prescaler c'è quindi un contatore che quando supera una certa soglia dà in uscita un altro impulso. Questo prescaler può essere settato solo a potenze di 2 da 4 a 256 (massimo prescaler-> massimo tempo-> minima frequenza).

Ultima selezione, tralascando il sync clock, che dice semplicemente che devono passare due colpi di clock per aggiornare il timer, abbiamo il registro **TMR0L (registro low del timer 0)**. È un registro a 8 bit, i meno significativi del nostro registro di timer. **Questo registro è un altro contatore che setta la flag dell'interrupt TMR0IF quando va in overflow**. Se disabilito l’interrupt posso usarlo anche solo come contatore. **Il registro principale per configurare è T0CON**.

## APPUNTI LCD LEZIONE 2

Lo schermo LCD è composto da due righe e 16 colonne. La cosa importante da capire a riguardo è che per comunicare con questo display c’è un protocollo apposito, altrimenti dovremmo comunicare con la memoria interna dell’LCD. Per semplificarci la vita noi useremo la sua libreria che è ad un “livello di astrazione più alto” rispetto al manipolare l’LCD noi. Fabio dice “se volete imparare sta roba fatelo ma è inutile lol”.

Per comandare la memoria del dispositivo LCD bisogna infatti rispettare tempi precisi, quindi nella libreria ci dovrà essere qualcosa che mi genera questi ritardi, non è importante sapere come generarli ma sapere che ci sono. Quando andiamo a scrivere sull’LCD una stringa questa funzione della libreria impiega tempo (1000 cicli macchina ipotizza Fabio???), più che il quanto però l’importante è sapere che non è una operazione immediata come fare una somma scrivere qualcosa sull’LCD.

Come fanno a generarsi dei ritardi fissi? Con una funzione come il delay_ms(). State attenti che usare delay_ms() vuol dire sapere con sicurezza a che frequenza si sta andando. Per questo dovremmo impostare in un altro modo l’LCD se volessimo gestirlo ad una frequenza diversa dalla 32MHz (alla quale lo gestiamo noi). **Se imposti a 32 MHz la frequenza e non abiliti il PLL infatti, l’IDE fa i calcoli precisi con la frequenza che gli abbiamo impostato noi**.  

Attenti allo switch pin 6 (da schematico e che dovrebbe essere già settato hardware giusto). Il display LCD è collegato alla porta B, sui primi 6 pin della porta B per la precisione, dalla 0 alla 5.  

**Rimangono liberi B6 e B7**. Se abbiamo il display collegato da B0 a B5 non possono essere usati per altro se non per l’LCD. L’IOCB era dal B7 al B4. Ma se abbiamo intenzione di usare come In Circuit Debugger, **quindi se vogliamo usare il debugger per capire cosa non va nel nostro programma, anche B6 e B7 risulteranno fuori uso**. Per questo motivo l’IOCB non lo useremo più dai prossimi laboratori, ma ci concentreremo sulle altre porte, precisamente sul quando ci sarà una transazione di stato su qualunque pin.  

C’è una parte di codice fissa da impostare se vuoi accendere l’LCD, un pezzo va subito dopo le variabili globali, l’altro pezzo dopo le variabili locali e dopo i registri nel main. LCD si attiva se, ci sono le impostazioni della sua libreria fuori dal main, **se si fa LCD init, i due LCD command** e infine se si scrive su LCD out che cosa mostrare. In particolare l’ultima istruzione è **Lcd_Out(riga, colonna, “cosa vuoi scrivere”). Ricordatevi che le stringhe in C sono un semplice array di char, quindi quando le dichiariamo, una volta dichiarata la dimensione, quella è, non modificabile in runtime**. Quindi mi raccomando rifletti bene sulla dimensione di questo array, che risulta essere un puntatore alla prima cella della stringa. In C non esiste nessun metodo a runtime per conoscere la lunghezza di un vettore. Quando quindi dichiariamo queste stringhe in C dobbiamo usare per forza quello che si chiama carattere di terminazione \0 e indica la terminazione di una stringa. **Quindi quando passo una stringa come variabile, il compilatore legge fino a \0 e poi si ferma**. È importante questa cosa perché se gli passiamo una stringa senza carattere di terminazione lui continua a leggere anche oltre la terminazione di memoria che non appartiene più alla nostra variabile. **Quindi lascia sempre lunghezzastringa+ 1 di spazio o rischi di sbagliare a stampare sull’LCD**. Riguardo all’LCD all’esame ti viene dato un file con la intestazione e la manipolazione base della stringhe.
Col timer 0 quindi usiamo l’interrupt suo interno per temporizzare e salviamo la variabile ogni quanto abbiamo un interrupt. Per la precisione la frequenza di interrupt è frequenza d’ingresso (Fosc/4), poi diviso il prescaler che va da non attivo a 256. E successivamente tutto dipende dal valore iniziale di TMR0L perché altrimenti non servirebbero 256 impulsi per far avvenire l’overflow se non ci fosse il valore del TMRL0 a zero. Ma ovviamente meno di 255 per la formula scritta successivamente. La configurazione con cui usiamo la nostra scheda è 32MHz, ergo la frequenza di interrupt minima (che corrisponde al massimo tempo tra un interrupt e l’altro che possiamo ottenere in assoluto con il prescaler alla massima divisione), è:  

**f_min=f_osc/(4*256*(256-TMR0L))**

Ovviamente la frequenza è minima se anche TMR0L è uguale a zero. Il tempo di interrupt si ottiene semplicemente ribaltando la formula:  

**t_max=(4*256*(256-TMR0L))/f_osc**

**4/32MHz = 1/8MHz che corrisponde a 125 ns. Questo è fisso, a meno di disabilitare il PLL**. Il risultato finale del tempo massimo dà 8,192 ms. 8 come approssimazione all’esame è più che sufficiente ma se volessi fare meglio hai 2 opzioni: la prima è salvare bene il valore del tempo di interrupt e/o sistemare il prescaler oppure modulare TMR0L in modo che dia un numero intero sempre sempre con l’aiuto del prescaler (Ho fatto il calcolo, **se TMR0L==6 allora il tempo è precisamente 8 millisecondi)**. Come funziona la seconda soluzione? Il valore iniziale del TMR0L, ogni volta che c’è un overflow, si attiva un interrupt ed è lì che dovrei impostare il valore che voglio nel TMR0L. **Per Timer0 impostare registri T0CON e INTCON.**

## APPUNTI LEZIONE 3 (**nota bene per orale**)
Vediamo come mai posso rischiare di entrare due volte nella ISR dato un interrupt on change. Premo un bottone. Il valore logico di tensione che è fisicamente su PORTB hardware è 1 perché io sto premendo il bottone. Sulla XOR in alto l’uscita è 1. Quando l’uscita di quella XOR è 1, in automatico RBIF è automaticamente portato ad 1, il main si ferma ed entriamo in ISR. Ora che siamo nella ISR INTCON.RBIF==1? Sì perché la flag è partita con la pressione del bottone.

Invece se resetti subito il flag ma non fai nessuna lettura della PORTB, lo XOR è ancora a 1, il PIC rialza il flag: questo perché la Q del flip flop non è cambiata, è ancora 0, ma noi su PORTB stiamo ancora premendo il pulsante quindi c’è un mismatch e l’uscita dello XOR è ancora 1. Ma quindi, dato che la flag è ancora alta, appena esco dal primo ISR rientro subito ed eseguo ISR una seconda volta, tutto questo mentre io sto ancora premendo il pulsante. Ancora una volta INTCON.RBIF==1, resetto la flag alla prima riga della ISR ma la seconda volta non è più vero che c’è il mismatch perché l’uscita del flip flop è stata aggiornata dalla ISR precedente (abbiamo fatto una read sulla PORTB tramite la condizione if (PORTB.RB6) appena dopo aver buttato giù la flag). L’uscita della XOR ora è a 0 e non rientrerò più nel ISR.
Riassumendo, mi ritrovo che se butto giù subito la flag ma non aggiorno la PORTB mi frego e rientro due volte nella stessa ISR ad ogni pressione del pulsante dell’interrupt on change.

## APPUNTI SONAR (CCP) LEZIONE 4
Utilizzo del sonar. Quando si intende utilizzare il sonar, le cose importanti da ricordare sono principalmente:  
1.	Il sonar è collegato a **PORTC**. Se il sonar viene utilizzato in modalità **Pulse Width Output**, devo impostare **RC2 digital Input**  (ANSELC=0 TRISC=1); sennò uso **RC3** come **Analog Input** -> è necessario usare l’ADC.

2.	Il sonar si appoggia al Timer **TXCON** change sovrascrive i dati nel registro del capture **CCPXCON** da 16 bit, suddiviso nei registri **CCPXH** e **CCPXL** (8 bit), high e low rispettivamente. I timer dispari vengono usati **Capture and Compare** mentre per i timer pari vengono usati per il **PWM**. Si imposta il timer con il registro **CCPTMRS0**.
3.	È una periferica del uC, quindi per attivarne l’interrupt **INTCON.PEIE=1** (non dimenticare il general **INTCON.GIE=1**)
4.	L’attivazione degli interrupt non è solo legata al registro **INTCON** ma anche ai registri **PIEX** e **PIRX**. Bisogna cercare il numero corretto del registro a cui sostituire la X. **Esistono cinque registri per gli interrupt**.
5.	Se vuoi lo stream continuo dei dati imposta **LATC.RC6=1**, inoltre deve essere digital Output, quindi TRISC del bit 6 è sempre; Gli altri bit si possono mettere benissimo in modalità input, in particolare **RC2** o **RC3** (digial/analog)
6.	Non dimenticare la routine di interrupt che è sempre identica:
```sh
if(PIR1.CCP1IF){
if(CCP1CON.CCP1M0){	          // If we are sensing the Rising-Edge
ta = ( CCPR1H << 8 ) + CCPR1L; //merge 8 and 8 bit in 16 bit
CCP1CON.CCP1M0 = 0;            // Set Sense to Falling
		}
		else{	                    // If we are sensing the Falling-Edge
tb = ( CCPR1H << 8 ) + CCPR1L;
   		width = tb - ta;
CCP1CON.CCP1M0 = 1;            // Set Sense to Rising
		}
PIR1.CCP1IF = 0;
}
```

Il sonar viene utilizzato per misurare le distanze lanciando un segnale ad ultrasuoni e contando il tempo che impiega a tornare.

Il sonar funziona con la modalità **Pulse Width Output** : genera un impulso proporzionale alla distanza misurata. Infatti 1mm equivale a 1us di "larghezza".  

Analizziamo il sonar:  
**Pin 2 --> RC2** dove c’è l’uscita dell’impulso collegato in ampiezza   
**Pin 4 --> RC6** serve a dire che il sensore effettua la misura ogni volta che trova il rising edge (se alto allora la misura è continua)

Passano 100mS tra ogni misura. Dal datasheet leggiamo 1mm di risoluzione, minima distanza misurata 300 mm.  

Il nostro PIC ci offre a disposizione un modulo che, senza usare cicli macchina ci permette di far partire le misurazioni in automatico mentre noi svolgiamo le altre operazioni: **il modulo CCP** .  
**CCP = Capture Compare PWM**.  
**Esistono 5 moduli CCP in totale collegati a tre pin**, non tutti gli I/O possono essere usati per questa funzione.  
**Il modulo capture si appoggia ad un timer (TMR1/3/5)** e lo utilizza in modalità 16 bit. Quando dal pin esterno riceve un rising/falling edge, lui va a campionare il valore del timer su un registro che noi possiamo andare a leggere. All’arrivo dell’evento campiona e salva il valore del timer in un registro.  

Nel nostro caso abbiamo **TMR1** utilizzato in free running a **fosc/4**. Il contatore del timer non fa altro, ovviamente, che sommare al tempo più uno.  

Bisogna utilizzare l’interrupt del CAPTURE. Ogni volta che cambio rising e falling (modalità per acquisire la distanza percorsa dal segnale), rischio che scatti un interrupt -> **NB: disabilita l’interrupt del TMR1**  

TMR1 è un contatore da 16 bit, **prima o poi andrà in overflow** .  
E se la seconda misura venisse presa dopo un overflow? Come si fa? Analizziamo (esempio dell’orologio):   

Immaginiamo un orologio con due lancette A e B. A è prima di mezzo giorno, la seconda è dopo mezzogiorno. La distanza tra le lancette è la stessa ma B raffigura un overflow del timer perchè ha superato mezzo giorno.  
**Questo caso viene magicamente risolto dall’ALU**: i designer del micro avevano previsto la possibilità di ovf. Quindi, sapendo che l’ALU fa le somme con il complemento a due (CPL2),  l’ ALU fa diventare il secondo operando negativo e poi somma i due operandi. Però così facendo ho bisogno di 17 bit. L’ALU conserva quest'ultimo nel bit di carry (vedi lo status).  

Unità di misura del timer fosc/4, in tempo ogni colpo avviene ogni 125 ns (sapendo che fosc= 32MHz).
dt= timer*125ns
In un microsecondo ci stanno circa 8 volte 125 nanosecondi. Usiamo adeguatamente lo shift per ottenere la misura corretta.

## APPUNTI ADC LEZIONE 6
ADC. L’ADC funziona a 8/10 bit. I registri dell’ADC sono ADCON0, ADCON1,ADCON2.
ADCON0 serve a determinare con i pin da 6-2 il pin scelto per il trasferimento da analogico a digitale. In particolare se pensi di lavorare col sonar a p.22 del datasheet si vede la conversione RC3/AN15 che corrisponde ad 01111. Il pin ADC_GO_NOT_DONE è il bit 1 e quando è alto incomincia la conversione, quando la conversione è completata torna a zero. Il bit 0 di questo registro invece è legato alla abilitazione dell’ADC e deve essere 1.
ADCON1 è invece il registro delle alimentazioni. Noi di solito lo alimentiamo 0/+5V e quindi lo poniamo semplicemente tutto a 0.
ADCON2 invece contiene il bit 7 che è legato alla giustificazione, ovvero, dove vengono salvati i bit più significativi e dove i meno significativi. Considera che abbiamo due registri ADRESH e ADRESL e che il massimo dei bit da salvare sarà 10. Se usiamo ADC ad 8 bit allora se giustificato a destra o a sinistra non è importante, basta copiare dal registro corretto il dato. Se invece a 10 bit, 2 bit saranno o nell’High o nel registro Low e come vanno salvati dipende dalla giustificazione. I bit da 5 a 3 permettono di settare il tempo d’acquisizione (tempo di delay dopo che il condensatore del S&H viene bloccato). Nel caso venga settato a 0  appena viene settato il GO dell’ADC parte subito la conversione, la cui lunghezza verrà data da Tad. Attenzione che se l’acquisition time è zero e Tad troppo corto, la conversione non andrà a buon fine. Quindi è sconsigliatissimo tenere l’acquisition time a zero. Di solito si piazza ad un valore che superi almeno 7us quindi se ho un Tad di 1us posso settare l’acquisition time a 16Tad, sul datasheet dice che servono almeno 11Tad per avere una conversione riuscita
Seleziona il prescaler di fosc per il modulo ADC e, come conseguenza setta la durata di 1 Tad (si può usare anche un clock di un oscillatore dedicato FRC che va a 600Khz
Nota: Tacqt è il tempo in cui il S&H è ancora agganciato al pin del PIC e quindi il condensatore  è ancora libero di caricarsi prima che intervenga Tad per iniziare l’acquisizione del valore.
registri.
RICORDA DI CONFIGURARE BENE I PORT con ansel=1 e tris=1
Nota: potrei anche tenere il buffer input digitale acceso con ANSEL=0 però consumerebbe potenza a caso, la conversione avviene correttamente a priori)
Nota: ADIF è settato alla fine di ogni conversione a prescindere dall’interrupt abilitato o no
Nota: il GO/DONE non deve essere messo nella stessa istruzione in cui viene acceso l’ADC

## APPUNTI SUL PWM LEZIONE 7
PWM. Il PWM è un modulo che permette di generare un’onda quadra con duty cycle variabile. Viene spesso usato per alimentare a diverse potenze un carico.
Il pwm ha due comparatori HIGH/LOW. il comparatore sotto setta alto l’output, mentre quello sopra low. Un problema costruttivo di questa scheda è legato al fatto che abbiamo un PIC che lavora ad 8 bit ma abbiamo un pwm che formalmente lavora a 10. Come è possibile? Sono riusciti ad ottenere 1024 valori di quantizzazione possibile in questa maniera: Se il registro CCPRxH è da 8 bit, i due bit mancanti per renderlo da dieci bit vengono presi dai bit meno significativi di un altro registro e vengono affiancati nella parte meno significativa del nostro registro. Parliamo della implicazione di frequenza e come fanno i comparatori a comparare un valore a 10 bit: Anche al 	2, che di solito funziona ad 8 bit, sono stati aggiunti due bit per fare la comparazione con il valore nel registro CCPRxH. Ovviamente i bit aggiuntivi vanno ad una frequenza 4 volte superiore rispetto a quella a cui funzionava il timer. Se invece li avessimo messi nei bit più significativi, clockati a 256 volte superiore se bit MSB e frequenza più bassa di 4 volte. La max frequenza è fosc/4, per questo motivo per coordinare i due bit del timer c’è un contatore dedicato che lavora ad fosc. Inoltre per questo motivo, legato al coordinamento in frequenza dei bit LSB del TIMER2, il prescaler è disponibile solo ad 1, 4 o 16. CCPRxL è l’unico registro. Solo sul Ton o Tup bisogna utilizzare questa manovra. Tup è CCPRxL, concatenazione AND con due bit del CCPxCON in questa posizione 4,5 che da 8 è diventato 10 bit (?). Il risultato è che il nostro “timer” è come se andasse a fosc grazie ai due bit aggiuntivi.

Facciamo un esempio: 	 PRx=5			CCPRxL=2
Supponiamo che TMRx=0 e siamo in uscita con HIGH. Il timer conta e arriva a 5 (valore di PR). Il comparatore sotto scatta, latcha l’uscita a 5, fa copiare CCPRxL in CCPRxH, resetta TMRx.
Quindi, con l’uscita alta, TMRx riparte a contare. Ad un certo punto conta fino a 2 che è il valore di CCPRxH. L’uscita viene quindi latchata verso il basso grazie al flip flop. TMRx continua quindi a contare fino a che non raggiunge 5, istante in cui l’onda viene riportata alta e quindi si ripete il ciclo appena descritto. Il tempo totale del ciclo è PRx+1 perché l’onda ci mette un colpo di clock a tornare alta.
Come aumento la frequenza? Diminuisco PRx  la risoluzione diminuisce.
La risoluzione del CCP viene dettata da PRx (se ho PR=5 ho solo 5 passi modificabili). Ho risoluzione massima con prescaler a 1 f=fosc/4. Di conseguenza ogni passo è 125ns con un osc di 32M. Avessimo un prescaler di 16 allora 125ns *16=2us.
Definiamo il duty cycle=CCPL/PRx (in cui PRx è un registro a 8 bit)

# PARTE SU ASSEMBLY  
## QUALI ISTRUZIONI AFFLIGGONO LO STATUS?
- Tutte le operazioni di addizione (ADDWF ADDLW), sottrazione (SUBWF SUBLW) affliggono i bit C (carry) DC (digit carry) Z (Zero bit)
- Le rotate left e rotate right (RLF RRF) affliggono il bit C (carry) perché ho bisogno di 1 bit da salvare da qualche parte mentre sto shiftando

- Tutte le operazioni di incremento e decremento non condizionali affliggono il bit Z
- Le operazioni logiche ANDWF CLEARF CLEARW COMF IORWF MOVF XORWF ANDLW IORLW  XORLW affliggono il bit Z
- Tutte le operazioni rimanenti non affliggono lo status (esempio lo swap dei nibbles)

## COS’È LO STATUS?
Lo STATUS è uno dei registri più importanti del microcontrollore. Esso contiene dei bit legati alle operazioni effettuate dall’ALU, il bit dello stato di RESET e i bit del controllo del paging dei banchi di memoria.



## SPIEGAZIONE ALCUNE ISTRUZIONI ASM (RIVEDERE GOTO E BRA CON PIC16 PIC18)
GOTO XXX: aggiorna il PC ad un’etichetta designata. La dimensione dell’indirizzo dell’etichetta è di 11 bit È sensibile al paging, infatti il bit 12 e 13 vengono acchiappati da PCLATH. Quindi usare BANKSEL. Nei PIC18 il PC è a 21 bit assoluti, il goto è a 20bit quindi può spostarti ovunque in 2M di memoria (in due cicli), ma c’è il problema del paging con PCLATU (non più H)
BRA XXX: è un salto (un po’ come il goto) relativo all’istruzione di partenza. Posso spostarmi di massimo +-1023 posizioni dall’indirizzo di partenza. Essendo un salto condizionale partendo dal valore del PC corrente, non è affetto dal paging (non è disponibile nei PIC16).
RETFIE: specifica il return dall’interrupt (quindi sposta quello che c’era nel Top Of Stack nel PC) e riattiva il general interrupt GIE (che è stato disattivato all’ingresso della routine dell’interrupt).
CALL: va all’etichetta, esattamente come il goto, quindi è affetto da paging (USARE BANKSEL). Si salva nello stack il PC corrispondente alla posizione di chiamata. Appena la routine chiamata dal CALL, il PC viene rishiftato al chiamante.


## DIRETTIVE ASM
#include: tale e quale a C++. Si usa con #include <p18f452.inc>
UDATA: dichiara l’inizio di una sezione di dati non inizializzati. Per riservare lo spazio in questa sezione bisogna utilizzare la direttiva RES. L’utilizzo è “LABEL res #byte”.Se non vengono specificate label e indirizzo, (es VARIABLES_IN_BANK udata 0x20) si scrive .udata e il linker fa tutto da sé.
BANKSEL XXX: Serve per selezionare automaticamente il bank in base all’etichetta desiderata. Quindi se voglio selezionare un TRISB e non so dove sono, scrivo BANKSEL TRISB. Questo mi permette di poter mettere (con le dovute precauzioni della scelta del micro) lo stesso codice su un altro micro senza preoccuparmi del Bank da selezionare.
CODE XXXX : (indirizzo opzionale) significa che il linker può piazzare il codice nella program memory all’indirizzo specifico XXXX segnalato dall’utente, oppure viene lasciata libera la scelta dell’indirizzo al linker se XXXX non viene specificato.
END: specifica all’assembler che questa è la fine del file asm. Ogni file asm deve necessariamente finire con la direttiva END. Se così non fosse, l’assembler continuerebbe a passare tutta la memoria.
equ: analogo del define del c++. Si scrive come “PORTCACCA equ PORTD” (etichetta eq nome_indirizzo_in_RAM).
DIFFERENZE TRA PSEUDO-ISTRUZIONI, MACRO, DIRETTIVE
Direttiva: una direttiva assembler è un comando utilizzato a livello software che compare nel source code ma non è direttamente traducibile come opcode. Di conseguenza una direttiva non compare nell’instruction set del datasheet del PIC ma nella User’s guide del MPASM™ Assembler.
Pseudo-istruzione: istruzione ASM scritta con parole diverse in modo tale da agevolare la memorizzazione. Per esempio, nel PIC16 , c’è MOVWF, mentre la sua “complementare” dovrebbe essere MOVFW. Nelle istruzioni del datasheet però esiste solo MOVF, w.
L’assemblatore via software permette di tradurre direttamente MOVF,w tramite la pseudo-istruzione MOVFW, in modo tale da avere meno confusione nel codice e migliore memorizzazione.
Macro: può essere considerata come “un’istruzione” a livello di sviluppo software, ma in realtà è una routine di istruzioni unificate sotto un unico nome. A differenza delle funzioni in linguaggio C, la macro è una sostituzione in-line. Questo significa che non viene effettuata una call, non viene cambiato il PC, non viene allocato spazio nello stack. Il contenuto della macro viene semplicemente inserito in quel punto del programma. Per l’utilizzo leggi la User’s guide del MPASM™ Assembler.
