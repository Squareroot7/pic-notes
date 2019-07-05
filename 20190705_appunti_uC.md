# GPIO SETUP

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

# APPUNTI TIMER0 LEZIONE 2
Il TIMER0 è per definizione un contatore che subisce un overflow dopo un tempo definito dall'utente attraverso setup dei registri dedicati.
Si parte da un oscillatore esterno o interno. Mediante un multiplexer possiamo la frequenza di incremento del contatore. **la nostra frequenza tipica è Fosc/4 in cui, grazie al PLL, Fosc è 32MHz (8MHz di default)**.

**T0CKI** pin non lo useremo mai, è il pin per comunicare con l'esterno che se abilitato come sorgente fornisce un impulso edge sensitive in base alla impostazione selezionata con T0SE (variazione sul rising/falling edge).

Successivamente alla selezione della sorgente c'è la scelta del sì o no prescaler, **il prescaler è un divisore di frequenza**. Il modo più facile per creare un prescaler è fare un contatore. All'interno del prescaler c'è quindi un contatore che quando supera una certa soglia dà in uscita un altro impulso. Questo prescaler può essere settato solo a potenze di 2 da 4 a 256 (massimo prescaler-> massimo tempo-> minima frequenza).

Ultima selezione, tralascando il sync clock, che dice semplicemente che devono passare due colpi di clock per aggiornare il timer, abbiamo il registro **TMR0L (registro low del timer 0)**. È un registro a 8 bit, i meno significativi del nostro registro di timer. **Questo registro è un altro contatore che setta la flag dell'interrupt TMR0IF quando va in overflow**. Se disabilito l’interrupt posso usarlo anche solo come contatore. **Il registro principale per configurare è T0CON**.
