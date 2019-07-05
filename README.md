# pic-notes
Appunti del corso di microcontrollori, Politecnico di Milano, 2018-19

<!-- TOC depthFrom:2 depthTo:6 withLinks:1 updateOnSave:1 orderedList:1 -->

1. [GPIO SETUP](#gpio-setup)
2. [APPUNTI TIMER0 LEZIONE 2](#appunti-timer0-lezione-2)
3. [APPUNTI LCD LEZIONE 2](#appunti-lcd-lezione-2)

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
