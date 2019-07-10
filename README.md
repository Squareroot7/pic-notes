# pic-notes
Appunti del corso di microcontrollori, Politecnico di Milano, 2018-19

<!-- TOC depthFrom:2 depthTo:6 withLinks:1 updateOnSave:1 orderedList:1 -->

1. [Lezione 1 - GPIO](#lezione-1-gpio)
			NaN. [USO PORT O LAT IN READ O WRITE?](#uso-port-o-lat-in-read-o-write)
			NaN. [EASYPIC BOARD SETUP ( usi tipici delle PORTx )](#easypic-board-setup-usi-tipici-delle-portx-)
2. [Lezione 2 - Timer 0](#lezione-2-timer-0)
3. [Lezione 2 - TIMER1 modalità 16Bit vs 2*8](#lezione-2-timer1-modalit-16bit-vs-28)
4. [Lezione 2 - LCD](#lezione-2-lcd)
5. [Lezione 4 - CCP (Sonar)](#lezione-4-ccp-sonar)
6. [Lezione 6 - ADC](#lezione-6-adc)
7. [Lezione 7 - PWM](#lezione-7-pwm)
8. [ASSEMBLY](#assembly)
	1. [STATUS register](#status-register)
		1. [Quali istruzioni affliggono lo STATUS register?](#quali-istruzioni-affliggono-lo-status-register)
	2. [Salti e return in assembly](#salti-e-return-in-assembly)
	3. [Direttive ASM](#direttive-asm)
		1. [Differenze tra pseudo-istruzioni, macro, direttive](#differenze-tra-pseudo-istruzioni-macro-direttive)

<!-- /TOC -->

## Lezione 1 - GPIO

NB: pin settati in Input per default. Quando si setta un digital out scrivere sempre LATx=0 per reset.

| Uso         | TRIS          | ANSEL          |
|:------------|:--------------|:---------------|
| Digital Out | TRISx.pin = 0 | dontcare       |
| Digital In  | TRISx.pin = 1 | ANSELx.pin = 0 |
| Analog In   | TRISx.pin = 1 | ANSELx.pin = 1 |


**NB**: se dei pin non vengono definitivamente utilizzati nel progetto, vengono settati come Output e poi il **LAT viene impostato a 0**. Questo per prevenire cambi della porta che comportano consumo dovuto alla potenza dinamica di switching dell’ingresso.
Ogni port ha diversi registri:

| Registro | 0 =                       | 1 =                        | Al reset               |
|:---------|:--------------------------|:---------------------------|:-----------------------|
| TRISx    | 0 = Output                | 1 = Input                  | Input                  |
| PORTx    | 0 = Reading low           | 1 = Reading high           | Undefined              |
| LATx     | 0 = Writing low           | 1 = Writing high           | Undefined              |
| ANSELx   | 0 = Digital buffer **ON** | 1 = Digital buffer **OFF** | Digital buffer **OFF** |


##### USO PORT O LAT IN READ O WRITE?
- **WRITE**: **LATx** e **PORTx** effettuano la stessa identica operazione di scrittura
- **READ**: **PORTx** rappresenta lo stato fisico del pin, mentre **LATx** rappresenta il registro che comanda il data latch.  

**Quando avvengono casini?** Prendo per esempio **RB1** come digital Output. Faccio tutte le operazioni che mi pare con **PORTB.RB1** oppure **LATB.RB1** (non fa differenza).  
Ora cambio **RB1** da digital Output a digital Input. Come risultato finale suppongo di avere **LATB.RB1=1**. Un bottone esterno impone ora il pin fisico con stato a 0. Se ora uso **PORTB.RB1**, leggo che effettivamente il pin è a zero. Se invece leggo LATB.RB1 mi ritrovo che il pin è a 1! Di conseguenza cosa devo usare?
- **digital Input usare sempre e solo PORTB** in READ  
- **digital Output** usare **PORTB** e **LATD** in WRITE a piacere


##### EASYPIC BOARD SETUP ( usi tipici delle PORTx )
- **PORTA**: usata per aggiungere pulsanti e per fare polling.
- **PORTB**: quasi interamente dedicata al LCD, attenzione che potrebbero venire usati RB6 e RB7 come pulsanti per interrupt on change (IOCB). **NB**: RB6 e RB7 vanno fuori uso con il debugger ON.
- **PORTC**: poco spesso usata per altri scopi se non per il modulo sonar.
- **PORTD**: spesso usata per operazioni con i led (tutta la porta).
- **PORTE**: usata per un PWM, PWM software o per dei led di segnalazione

## Lezione 2 - Timer 0
Il TIMER0 è per definizione un contatore che subisce un overflow dopo un tempo definito dall'utente attraverso setup dei registri dedicati.
Si parte da un oscillatore esterno o interno. Mediante un multiplexer possiamo selezionare la frequenza di incremento del contatore. **la nostra frequenza tipica è Fosc/4 in cui, grazie al PLL, Fosc è 32MHz (8MHz di default)**.

**T0CKI** pin non lo useremo mai, è il pin per comunicare con l'esterno che se abilitato come sorgente fornisce un impulso edge sensitive in base alla impostazione selezionata con T0SE (variazione sul rising/falling edge).

Successivamente alla selezione della sorgente c'è la scelta del sì o no prescaler, **il prescaler è un divisore di frequenza**. Il modo più facile per creare un prescaler è fare un contatore. All'interno del prescaler c'è quindi un contatore che quando supera una certa soglia dà in uscita un altro impulso. Questo prescaler può essere settato solo a potenze di 2 da 4 a 256 (massimo prescaler-> massimo tempo-> minima frequenza).

Ultima selezione, tralascando il sync clock, che dice semplicemente che devono passare due colpi di clock per aggiornare il timer, abbiamo il registro **TMR0L (registro low del timer 0)**. È un registro a 8 bit, i meno significativi del nostro registro di timer. **Questo registro è un altro contatore che setta la flag dell'interrupt TMR0IF quando va in overflow**. Se disabilito l’interrupt posso usarlo anche solo come contatore. **Il registro principale per configurare è T0CON**.

Con il TIMER0 quindi usiamo il suo l’interrupt per temporizzare e salviamo o incrementiamo la variabile di conta ad ogni interrupt.  
Per la precisione la **frequenza di interrupt è frequenza d’ingresso (Fosc/4) diviso il prescaler che va da non attivo a 256** .  
Successivamente tutto dipende dal **valore iniziale di TMR0L** perché altrimenti non servirebbero 256 impulsi per far avvenire l’overflow se non ci fosse il valore del **TMRL0** a zero. Ma ovviamente meno di 255 per la formula scritta successivamente. La configurazione con cui usiamo la nostra scheda è 32MHz. La frequenza di interrupt minima (che corrisponde al massimo tempo tra un interrupt e l’altro), è:  

<code> **f_min = f_osc / ( 4 * 256 * ( 256 - TMR0L ) )** </code>

Ovviamente la frequenza minima ottenibile tenendo conto di **TMR0L** . Il tempo di interrupt si ottiene semplicemente ribaltando la formula:  

<code> **t_max = ( 4 * 256 * ( 256 - TMR0L ) ) / f_osc** </code>

**4/32MHz = 1/8MHz che corrisponde a 125 ns. Questo è fisso, a meno di disabilitare il PLL**. Il risultato finale del tempo massimo dà 8,192 ms. 8 come approssimazione all’esame è più che sufficiente ma se volessi fare meglio hai 2 opzioni: la prima è salvare bene il valore del tempo di interrupt e/o sistemare il prescaler oppure modulare TMR0L in modo che dia un numero intero sempre sempre con l’aiuto del prescaler (Ho fatto il calcolo, **se TMR0L==6 allora il tempo è precisamente 8 millisecondi)**. Come funziona la seconda soluzione? Il valore iniziale del TMR0L, ogni volta che c’è un overflow, si attiva un interrupt ed è lì che dovrei impostare il valore che voglio nel TMR0L. **Per Timer0 impostare registri T0CON e INTCON.**


## Lezione 2 - TIMER1 modalità 16Bit vs 2*8  
**NB:** solo i timer pari possono essere a 8 bit.  
Detto ciò notiamo immediatamente che all' interno del registro TxCON è presente un bit (TxRD16) che parla di **modalità 16bit o 2x8**.  

È possibile usare **TMR** dispari a 8 bit, ma **è necessario o precaricare il byte alto a FF per avere interrupt corretti, o usare solo il byte alto tenendo conto del prescaling fatto da L di 256.**

**Modalità 2x8**  
Questa è la modalità più semplice per usare il timer, **TMRx** è semplicemente diviso in **TMrxH** e **TMRxL**, la lettura chiaramente viene fatta un bit alla vota e il byte flag di interrupt viene alzato all' overflow di **TMRxH**  

Questa modalità però **presenta dei problemi sostaziali**, che potrebbero portare a errori nel programma.

Facciamo un esempio: immaginiamo **TMR1** acceso, **TMR1L=0xFF TMR1H=0x00**  
Procediamo con la lettura di **TMR1L**, salviamo il dato in una variabile, risultato sarà pari a **0x00FF** ( a 16Bit)
A questo punto leggiamo il Byte alto, ma il timer non è stato spento, ed è andato avanti nel conteggio: **MR1L=0x00 TMR1H=0x01**
Il programma procede a leggere il Byte successivo e il risultato sarà pari a

<code>**0x00FF + 0x0100 = 0x01FF** </code>

diverso sostanzialmente da 0x00FF che avremmo dovuto ottenere.

**Facciamo un esempio di lettura nell' altro senso:**
1. TMR1=0x00FF
2. leggo TMR1H -> Res sarà 0x0000;
3. il timer conta -> TMR1= 0x0100;
4. Leggo TMR1L -> Res sarà 0x0000; =/= 0x00FF che ci aspettavamo.

**Quando allora questa modalità è utile?**
-	Quando non ci interessa il contenuto del timer ma solo l'overflow
-	Quando ci interessa un solo byte del timer che sia H o L ( utile ad esempio in un **PWM software**, vedi gli appunti più avanti)

Per correggere questo problema è stato introdotta l'altra modalità
**1x16**
La modalità che chiameremo **1x16** viene chiamata ufficialmente **16Bit** read/Write e quello che fa di fatto è **introdurre un buffer su TMR1H:**
Il byte alto del timer a questo punto **on sarà più direttamente controllabile o leggibile**( lo chiameremo **TMR1H**), ma potremo accedere solamente al buffer( di fatto il PIC sposta fisicamente l'idirizzo, quindi a livello di codice noi accediamo lo stesso registro)
**Il buffer viene copiato** quando viene toccato **TMR1L**:
-	Leggo **TMR1L** -> **contemporaneamente TMR1H** verrà copiato in **TMR1H**, in questo modo il dato è salvato in sincronia, e anche se viene letto successivamente il conteggio del timer non lo influisce.
-	Scrivo **TMR1H** -> nello stesso istante viene copiato **TMR1H** in **TMR1H**, il caricamento del valore iniziale viene fatto in sincronia e non c'è rischio di problemi in caricamento.

**Problemi relativi a questo:**
-	Se devo accedere solo a **TMR1H** Devo per forza leggere **TMR1L**, perdendo di fatto tempo.
-	Se dovessi sbagliare l'ordine di lettura/scrittura otterrei dati/funzionamento non valido.
## Lezione 2 - LCD

**Lo schermo LCD è composto da due righe e 16 colonne** .  
La cosa importante da capire a riguardo è che per comunicare con questo display c’è un protocollo apposito, altrimenti dovremmo comunicare con la memoria interna dell’LCD. Per semplificarci la vita noi useremo la sua libreria che è ad un “livello di astrazione più alto” rispetto al manipolare l’LCD noi.

Per comandare la memoria del dispositivo LCD bisogna **rispettare tempi precisi**, quindi nella libreria ci dovrà essere qualcosa che mi genera questi ritardi, non è importante sapere come generarli ma sapere che ci sono. Quando andiamo a scrivere sull’LCD una stringa questa funzione della libreria impiega tempo ( circa 1000 cicli macchina), **più che il quanto però l’importante è sapere che non è una operazione immediata come fare una somma scrivere qualcosa sull’LCD** .  

**Come fanno a generarsi dei ritardi fissi?** Con una funzione come il **delay_ms()**. Attenzione che usare delay_ms() vuol dire sapere con sicurezza a che frequenza si sta andando. Per questo dovremmo impostare in un altro modo l’LCD se volessimo gestirlo ad una frequenza diversa dalla 32MHz (alla quale lo gestiamo noi). **Se imposti a 32 MHz la frequenza e non abiliti il PLL infatti, l’IDE fa i calcoli precisi con la frequenza che gli abbiamo impostato noi**.  

Attenzione anche allo switch pin 6 (dallo schema e che dovrebbe essere già settato hardware giusto). Il display LCD è collegato alla porta B, sui primi 6 pin della porta B per la precisione, dalla 0 alla 5.  

**Rimangono liberi B6 e B7**. Se abbiamo il display collegato da B0 a B5 non possono essere usati per altro se non per l’LCD. L’IOCB era dal B7 al B4. Ma se abbiamo intenzione di usare come In Circuit Debugger, **quindi se vogliamo usare il debugger per capire cosa non va nel nostro programma, anche B6 e B7 risulteranno fuori uso**. Per questo motivo l’IOCB non lo useremo più dai prossimi laboratori, ma ci concentreremo sulle altre porte, precisamente sul quando ci sarà una transazione di stato su qualunque pin.  

C’è una parte di codice fissa da impostare se vuoi accendere l’LCD:

```
// Lcd module connections
sbit LCD_RS at LATB4_bit;
sbit LCD_EN at LATB5_bit;
sbit LCD_D4 at LATB0_bit;
sbit LCD_D5 at LATB1_bit;
sbit LCD_D6 at LATB2_bit;
sbit LCD_D7 at LATB3_bit;

sbit LCD_RS_Direction at TRISB4_bit;
sbit LCD_EN_Direction at TRISB5_bit;
sbit LCD_D4_Direction at TRISB0_bit;
sbit LCD_D5_Direction at TRISB1_bit;
sbit LCD_D6_Direction at TRISB2_bit;
sbit LCD_D7_Direction at TRISB3_bit;
// End Lcd module connections

void main(){
  Lcd_init();
  Lcd_Cmd(_LCD_CLEAR);
  Lcd_Cmd(_LCD_CURSOR_OFF);
  //
  //main code and while(1);
  //
}
```

**NB**: ricordati di attivare la libreria LCD e la parte di conversione delle char nell sezione delle librerie nell'IDE MikroC.

In particolare il comando principale è **Lcd_Out(riga, colonna, “cosa vuoi scrivere”)**. Attenzione che **le stringhe in C sono un semplice array di char**, quindi una volta dichiarata la dimensione, quella è, non modificabile in runtime.  
Quindi mi raccomando rifletti bene sulla dimensione di questo array, che risulta essere un puntatore alla prima cella della stringa.  
In C non esiste nessun metodo a runtime per conoscere la lunghezza di un vettore.  
Quando dichiariamo queste stringhe in C dobbiamo usare per forza quello che si chiama carattere di terminazione \0 e indica la terminazione di una stringa. **Quindi appena passo una stringa come variabile, il compilatore legge fino a \0 e poi si ferma** .  
È importante questa cosa perché se gli passiamo una stringa senza carattere di terminazione lui continua a leggere anche oltre la terminazione di memoria che non appartiene più alla nostra variabile. **Lascia sempre lunghezzastringa+ 1 di spazio o rischi di sbagliare a stampare sull’LCD**.

Riguardo all’LCD all’esame c'è il **file con la intestazione e la manipolazione base della stringhe**.

Vediamo come mai posso **rischiare di entrare due volte nella ISR dato un interrupt on change** .    
Premo un bottone. Il valore logico di tensione che è fisicamente su PORTB hardware è 1 perché io sto premendo il bottone -> **XOR=1** -> scatta l'interrupT -> **RBIF=1**, il main si ferma ed entriamo in ISR.  
Ora che siamo nella ISR **INTCON.RBIF==1** ? Sì, la flag è partita alla pressione del bottone.

Se ora resetto subito la flag ma non faccio nessuna lettura della **PORTB**, la **XOR** è ancora a 1, il PIC rialza la flag: questo perché la Q del flip flop non è stata cambiata (non ho letto niente ancora), è ancora 0, ma su **PORTB** il pulsante è ancora premuto quindi c’è un mismatch e l’uscita dello XOR è ancora 1. Ma quindi, dato che la flag è ancora alta, appena esco dal primo ISR rientro subito ed eseguo ISR una seconda volta, tutto questo mentre io sto ancora premendo il pulsante. Ancora una volta INTCON.RBIF==1, resetto la flag alla prima riga della ISR ma la seconda volta non è più vero che c’è il mismatch perché l’uscita del flip flop è stata aggiornata dalla ISR precedente (abbiamo fatto una read sulla PORTB tramite la condizione if (PORTB.RB6) appena dopo aver buttato giù la flag). L’uscita della XOR ora è a 0 e non rientrerò più nel ISR.
Riassumendo, mi ritrovo che se butto giù subito la flag ma non aggiorno la PORTB mi frego e rientro due volte nella stessa ISR ad ogni pressione del pulsante dell’interrupt on change.

```
  // ******ISR SCORRETTO******
  void interrupt(){
    if(INTCON.REIF){ //flag alzata da PORTB
    INTCON.RBIF=0; //Reset della flag
    // a questo punto accade il mismatch
    // non ho aggiornato PORTB prima di buttare giù la flag
    if(PORTB.RB6)} i++; // leggo PORTB quindi alla isr dopo
    if(PORTB.RB) i--; // avrò aggiornato il flip flop
  }

  // ******ISR CORRETTO******
  void interrupt(){
    if(INTCON.REIF){ // flag alzata da PORTB
    if(PORTB.RB6)} i++; // leggo PORTB ->Flip Flop aggiornato!
    if(PORTB.RB) i--; // altro refresh di PORTB
    INTCON.RBIF=0; //Reset della flag
    // a questo punto ho PORTB refreshato -> NO mismatch
    // non entro nella ISR una seconda volta  
  }
```


## Lezione 4 - CCP (Sonar)

Utilizzo del sonar. Quando si intende utilizzare il sonar, le cose importanti da ricordare sono principalmente:  
1.	Il sonar è collegato a **PORTC**. Se il sonar viene utilizzato in modalità **Pulse Width Output**, devo impostare **RC2 digital Input**  (ANSELC=0 TRISC=1); sennò uso **RC3** come **Analog Input** -> è necessario usare l’**ADC**.
2.	Il sonar si appoggia al Timer **TXCON** change sovrascrive i dati nel registro del capture **CCPXCON** da 16 bit, suddiviso nei registri **CCPXH** e **CCPXL** (8 bit), high e low rispettivamente. I timer dispari vengono usati **Capture and Compare** mentre per i timer pari vengono usati per il **PWM**. Si imposta il timer con il registro **CCPTMRS0**.
3.	È una periferica del uC, quindi per attivarne l’interrupt **INTCON.PEIE=1** (non dimenticare il general **INTCON.GIE=1**)
4.	L’attivazione degli interrupt non è solo legata al registro **INTCON** ma anche ai registri **PIEX** e **PIRX**. Bisogna cercare il numero corretto del registro a cui sostituire la X. **Esistono cinque registri per gli interrupt**.
5.	Se vuoi lo stream continuo dei dati imposta **LATC.RC6=1**, inoltre deve essere digital Output, quindi TRISC del bit 6 è sempre; Gli altri bit si possono mettere benissimo in modalità Input, in particolare **RC2** o **RC3** (digial/analog)
6.	Non dimenticare la routine di interrupt che è sempre identica:


```
if(PIR1.CCP1IF){
  if(CCP1CON.CCP1M0){	          // If we are sensing the Rising-Edge
    ta = ( CCPR1H << 8 ) + CCPR1L; // merge 8 and 8 bit in 16 bit
    CCP1CON.CCP1M0 = 0;            // Set Sense to Falling
	}
  else{	                        // If we are sensing the Falling-Edge
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

**Vediamo ora un esempio** dell'utilizzo del Sonar con l'LCD:

```
// Lcd module connections
sbit LCD_RS at LATB4_bit;
sbit LCD_EN at LATB5_bit;
sbit LCD_D4 at LATB0_bit;
sbit LCD_D5 at LATB1_bit;
sbit LCD_D6 at LATB2_bit;
sbit LCD_D7 at LATB3_bit;

sbit LCD_RS_Direction at TRISB4_bit;
sbit LCD_EN_Direction at TRISB5_bit;
sbit LCD_D4_Direction at TRISB0_bit;
sbit LCD_D5_Direction at TRISB1_bit;
sbit LCD_D6_Direction at TRISB2_bit;
sbit LCD_D7_Direction at TRISB3_bit;
// End Lcd module connections

// fosc/4= 125ns quindi nel registro devo fare
// 1mm è 1us, in 1us ci stanno 8 volte 125ns, quindi uso uno shift
// delta tempo= differenza contatore * 125ns

// VARIABILI GLOBALI
unsigned int       t_rise=0, t_diff=0;

void main() {
	char txt_num[7], txt_lcd[17];
	//-----PORTS INIT-----
	ANSELC=        0b11111011; // setto l'input da leggere dove il sonar mi dà l'impulso
	TRISC=         0b10111111; // settare l'uscita per dire al sonar di continuare a leggere
	LATC.RC6=1;				// imposto l'uscita alta

	//-----CONFIG TIMER 1-----
	T1CON=         0b00000011; // pagina 166 impostato fosc/4, senza prescaler, senza secondo osc dedicato, timer abilitato e scrittura LOW/HIGH abilitata

	//-----CONFIG CCP1-----
	CCP1CON=       0b00000101; // pagina 198 100 capture mode falling edge, 101 rising edge, basta fare ++ e -- nella isr per cambiare modalità
	CCPTMRS0=      0x0;        // pagina 201 capture compare mode using TMR1

	//-----INTERRUPT CONFIG-----
	INTCON=        0b11000000; // pagina 109 global e peripheral abilitati
	PIR1=          0x0;        // pagina 112 CONTIENE flag del CCP1
	PIE1=          0b00000100; // pagina 117 abilitato interrupt CCP1
	//-----LCD INIT -----
	Lcd_init();
	Lcd_Cmd(_LCD_CLEAR);
	Lcd_Cmd(_LCD_CURSOR_OFF);
	Lcd_Out(1, 1, "hello world");

	while(1){
		if(t_diff != 0){
			t_diff = 0;						//resetta conta temporale
			IntToStr(t_diff, txt_num);     	//converti distanza in stringa
			strcpy(txt_lcd, "dist = ");		//inserisci dist = nella stringa txt_lcd
			strcat(txt_lcd, txt_num);		  //unisci la stringa txt_num nella stringa txt_lcd
			Lcd_Out(1, 1, txt_lcd);			//stampa stringa txt_lcd
		}
	}
}
void interrupt(){
	if(PIR1.CCP1IF){ 							//se c'è stato un interrupt ccp
		if(CCP1CON.CCP1M0){        			  //if rising
			t_rise = (CCPR1H<<8) + CCPR1L;	   //inserisci i due registri nel tempo di salita
			CCP1CON=0b00000100; 			 	//switch ccp mode to falling
		}
		else{
			t_diff = (CCPR1H<<8)  + CCPR1L;      // inserisci i due registri nel tempo di discesa
			t_diff -= t_rise;    				// calcolo il delta tempo (se fa ovf non c'è problema per la storia del Complemento a due e non mi interesso del bit ovf)
			t_diff= t_diff>>3;   				// delta tempo= differenza contatore * 125ns che sarà anche la nostra distanza in mm
			CCP1CON=0b00000101; 				 // switch ccp mode to rising
		}
		PIR1.CCP1IF=0;          			 	// disattivo flag interrupt
	}
}
```

### Cosa succede se TIMER1 va in overflow più volte durante una misura di CCP?
Di certo questo non è il caso del Sonar dato che il **tempo di overflow del TIMER1 è molto maggiore di quello del gradino in uscita dal sonar** .  
Il caso di un overflow singolo è già stato trattato e, da come sappiamo, viene **"risolto" completamente dall'ALU** .  
Quello che non viene gestito dall'ALU è invece un **overflow multiplo durante la misura**.  
Infatti se mantenessimo il sistema come per il SONAR, la misura temporale finale sarebbe **errata**. Vediamo perchè succede questo:
Se non tenessimo conto del numero di volte di overflow del **TIMER1** la misura finale corrisponderebbe a due intervalli:
1. Tempo passato dal **primo rising edge** al **primo overflow**
2. Tempo dall'**ultimo overflow** al **falling edge**  

Tutto ciò, da come possiamo immaginare, **non tiene assolutamente conto degli ovf in mezzo**. Come possiamo risolvere questo caso?
È necessario innanzitutto **contare gli ovf di TIMER1** quindi nella ISR avremo qualcosa simile a:  
```
void interrupt(){
  if(PIR1.TMR1IF){
    //inserire condizione che abilita ovf_number++ quando ho intercettato il rising edge
    ovf_number++;
  }
  ...
  //inserire interrupt CCP
}
```
Bene, ora abbiamo nella variabile <code> ovf_number</code> il nostro numero di overflow. **Dobbiamo aggiungere questo tempo alla conta finale**. Nella routine dell'interrupt CCP dovremo modificare <code>ta</code> e <code>tb</code> in modo tale da avere:

<code>ta = (((CCPRxH<<8) + CCPRxL ) + ovf_number<<16); </code>  
Ovviamente al momento dell'altro fronte dovremo fare  la stessa operazione con la variabile <code>tb</code> :  
<code>tb = (((CCPRxH<<8) + CCPRxL ) + ovf_number<<16); </code>  

Da notare che la variabile <code> ovf_number</code> al primo aggiornamento (quello di <code>ta</code>) , rispetto al secondo aggiornamento (quello di <code>tb</code>), **sarà stata incrementata in caso di ovf**.
**Worst case scenario:** anche  <code> ovf_number</code> va in overflow! Basta porla uguale a zero dopo aver fatto l'aggiornamento di tb, così **si escludono tutti gli ovf** avvenuti nel frattempo.


**NB: le parentesi sono importanti** .
Quando si eseguono operazioni di shift e somma che potrebbero essere fatte in più righe di codice come:
```
   ta=CCPRxH<<8; //ta è a 24 bit;
   ta+=CCPRxL;
   ta+=ovf_number<<16;
```
è buona regola essere abbondanti di parentesi per **essere** sicuri che il compilatore stia effettivamente facendo le operazioni desiderate e che non stia mescolando operazioni portando a errori che spesso e volentieri sono **ostici da debuggare**.  

**NB: attenzione alla dimensione delle variabili** .  
Se non trattassimo gli ov allora la variabile <code>ta</code> sarà (intuitivamente) di 16 bit (8 bit di CCPRxL + 8 bit CCPRxH). L'aggiunta della conta degli ovf però corrisponde ad una variabile di 8 bit shiftata di 16 -> **sono necessari 24 bit totali**  
A seconda del compilatore, <code>ta</code> dovrà essere dichiarata come uint oppure <code>unsigned long int</code> (spesso 32 bit).



## Lezione 6 - ADC
L’**ADC funziona a 8 oppure 10 bit** .  
I registri dell’**ADC** da settare sono **ADCON0 ADCON1 ADCON2** .  
- **ADCON0** serve a determinare attraverso i bit da 6 a 2 il pin scelto del uC per la conversione.  
In particolare se voglio lavorare con il sonar uso **RC3(AN15)** che corrisponde ad **01111**.  Inoltre il bit **ADCON0.ADC_GO_NOT_DONE** fa partire la conversione se alto. Quando la conversione è completata torna a zero. Il bit **ADCON0.ADON** abilita l'**ADC** e deve essere impostato a 1 per accenderlo.

- **ADCON1** è invece il registro delle alimentazioni. Noi di solito lo alimentiamo 0/+5V e quindi semplicemente **ADCON1=0**.
- **ADCON2** invece contiene il bit 7 che è legato alla **giustificazione**(dove vengono salvati i bit più significativi e dove i meno significativi). Considera che abbiamo due registri **ADRESH** e **ADRESL** e che il massimo dei bit da salvare sarà 10. Esso contiene anche la selezione del numero di **Tad** (vedi sotto) e la frequenza che comanda l'**ADC**.

Se usiamo l'**ADC** ad 8 bit non è importante giustificare a destra o sinistra, basta ricordare il registro corretto (HIGH/LOW, 8 bit ciascuno) che contiene il dato.  

**Se invece lo voglio a 10 bit**, 2 bit saranno o nell’High o nel registro Low e viceversa, a seconda di come viene giustificato.  

**I bit da 5 a 3 di ADCON2** permettono di settare **il tempo d’acquisizione** (tempo di delay dopo che il condensatore del S&H viene bloccato, scandito in numero di Tad). Nel caso venga settato a 0 appena viene settato il bit GO dell’**ADC** parte immediatamente la conversione, la cui lunghezza verrà data da Tad.  

**NB: attenzione che se l’acquisition time è zero e Tad troppo corto, la conversione non andrà a buon fine**. Quindi è sconsigliatissimo tenere l’acquisition time a zero.  
Di solito si piazza ad un valore che superi almeno 7us quindi se ho un Tad di 1us posso settare l’acquisition time a **16Tad**, sul datasheet dice che servono almeno **11Tad per avere una conversione riuscita**.

Seleziona il prescaler di fosc per il modulo **ADC** e, come conseguenza setta la durata di 1 Tad (si può usare anche un clock di un oscillatore dedicato FRC che va a 600Khz.  

**NB:** Tacqt è il tempo in cui il S&H è ancora agganciato al pin del PIC e quindi il condensatore  è ancora libero di caricarsi prima che intervenga Tad per iniziare l’acquisizione del valore.

**NB:** ricorda di configurare bene i **PORT** con **ANSELx=1** e **TRISx=1**

**NB:** potrei anche tenere il buffer Input digitale acceso con **ANSELx=0** però consumerebbe potenza a caso, la conversione avviene correttamente a priori).

**NB: ADIF** è settato alla fine di ogni conversione a prescindere dall’interrupt abilitato o no

**NB:** il **ADC_GO_NOT_DONE** non deve essere messo nella stessa istruzione in cui viene acceso l’**ADC**.  

Esempio di configurazione dell'**ADC**:

```
// set TRIS here if needed
ANSELC = 0b00001000;    //RC3 --> AN15
ADCON2 = 0b10101110;
//         +---------Right justified
//           +++-----Tacquisition = 12Tad
//              +++--Fosc/64  
ADCON1 = 0b00000000;    // Use all internal voltage supply range
ADCON0 = 0b00111101;    // DO NOT SET ADC.GO_NOT_DONE HERE
//          +++++----RA15
//                +--ADC ON
PIR1 = 0;               // reset all flags (adc,ccp,tmr1 ecc)
PIE1.ADIE = 1;          // enable ADC interrupt
INTCON = 0b11000000;    // enable GIE and PIE
ADCON0.GO_NOT_DONE = 1; // Start ADC Acquisition
```

## Lezione 7 - PWM
Il **PWM** è un modulo che permette di generare un’onda quadra con duty cycle variabile. Viene spesso usato per alimentare a diverse potenze un carico.
**Il PWM ha due comparatori HIGH/LOW** .  
Il comparatore sotto setta l’Output, mentre il comparatore sopra lo resetta.

Un problema costruttivo di questa scheda è legato al fatto che **abbiamo un PIC che lavora ad 8 bit ma abbiamo un pwm che formalmente lavora a 10** . Come è possibile?  
Sono riusciti ad ottenere 1024 valori di quantizzazione possibile in questa maniera: Se il registro **CCPRxH**  è da **8 bit, i due bit mancanti per renderlo da dieci bit vengono presi dai bit meno significativi di un altro registro e vengono affiancati nella parte meno significativa del nostro registro** .  
Parliamo della implicazione di frequenza e come fanno i comparatori a comparare un valore a 10 bit: Anche al 	2, che di solito funziona ad 8 bit, sono stati aggiunti due bit per fare la comparazione con il valore nel registro **CCPRxH** .  
Ovviamente **i bit aggiuntivi vanno ad una frequenza 4 volte superiore rispetto a quella a cui funzionava il timer** (se invece li avessimo messi nei bit più significativi, clockati a 256 volte superiore se bit MSB e frequenza più bassa di 4 volte).  

**La max frequenza è fosc/4** , per questo motivo per coordinare i due bit del timer **c’è un contatore dedicato che lavora ad fosc** .  
Inoltre per questo motivo, legato al coordinamento in frequenza dei bit LSB del TIMER2, **il prescaler è disponibile solo ad 1, 4 o 16** .  
**CCPRxL** è l’unico registro. **Tup (parte alta del duty cycle) è segnato da CCPRxL** che consiste in un AND con i due bit del CCPxCON. Il risultato è che il nostro “timer” è come se fosse 10 bit e andasse a fosc grazie ai due bit aggiuntivi.

Facciamo un esempio settando 	 **PRx=5			CCPRxL=2**  :  
supponiamo **TMRx=0** e l'uscita è HIGH. Il timer conta e arriva a 5 (valore di PR).
1. **Il comparatore sotto scatta**
2. **latcha l’uscita alta** quando raggiunge 5
3. copia **CCPRxL** in **CCPRxH**
4. resetta **TMRx**.  

Quindi, con l’uscita alta, TMRx riparte a contare. Ad un certo punto conta fino a 2 che è il valore di **CCPRxH**.
1. **latcha l'uscita bassa** grazie al flip flop
2. **TMRx continua a contare** fino a che non raggiunge 5
3. **l’onda viene riportata alta** e quindi si ripete il ciclo appena descritto.

**Il tempo totale del ciclo è PRx+1** perché l’onda ci mette un colpo di clock a tornare alta.

**Come aumento la frequenza** Diminuisco PRx -> **la risoluzione diminuisce** .  
La risoluzione del **CCP** viene dettata da PRx (Nota: se ho PR=5 ho solo 5 passi modificabili). Ho risoluzione massima con prescaler a 1 f=fosc/4. Di conseguenza ogni passo è 125ns con un osc di 32M. Avessimo un prescaler di 16 allora:  
<code>125ns * 16 = 2us</code>  

Vado ora a definire il duty cycle  
<code>duty cycle=CCPL/PRx </code> (in cui PRx è un registro a 8 bit)

Per il PWM ho solo due moduli CCP (4/5) che ne permettono l'utilizzo. Come posso sviare da questo? Mi costruisco un **PWM software** utilizzabile su praticamente qualsiasi pin del uC, come nell'esempio seguente:

```
unsigned short int pwm_cnt=0;
unsigned int time_cnt1=0, time_set1 = 100;
void main() {  
  unsigned short int pwm_period=255, //set del periodo del pwm
  hi_period=5;           //set della parte HI del pwm (  1  <  hi_period  <  pwm_period  )
  TRISD=0; 		      //setto la porta D come output
  LATD=1;  		      //butto tutto a zero
  T0CON=  0b11000000;    //attiva timer, attivalo a 8 bit e attivalo a 256 prescaler
  INTCON= 0b10100000;    //attiva timer 0 interrupt
  while(1){
	   if(pwm_cnt<=hi_period) LATD.RD0=1; //se sono ancora nella parte alta del pwm
           else LATD.RD0=0;                   // toggle del pin verso la parte bassa
           if(pwm_cnt==pwm_period) pwm_cnt=0; // reset del periodo pwm
           if(time_cnt1==time_set1) {         // cambio del duty cycle
              time_cnt1=0;
              hi_period++;                    // incrementa duty cycle
              if(hi_period==pwm_period) hi_period=0;    //se il periodo hi raggiunge il periodo completo, torna a zero
           }
	}
}

void interrupt(){        // ISR
   if(INTCON.TMR0IF) {   // con questo prescaler, entra circa ogni 8ms
      pwm_cnt++;         // incrementa contatore del periodo del pwm
      time_cnt1++;       // incrementa contatore del ciclo di cambio duty cycle
      INTCON.TMR0IF=0;   // resettare i flag sempre alla fine del codice
   }
}
```

Se invece volessi usare entrambi i moduli PWM, avrei l'uscita su **RE2** e **RD1**. Ecco l'esempio che fa utilizzo dei **TIMER2/TIMER4** (**Nota**: posso anche associare due moduli PWM allo stesso timer)
```
void main() {
     unsigned short int dir=1;
     //vedi pinout table nel datasheet per vedere a quale pin viene associato quale CCP
     TRISE.RE2=1;  //DISATTIVO output per settare ccp5
     TRISD.RD1=1;  //DISATTIVO output per settare ccp4
     //----TMR2----
     T2CON=0b00000100;
     //      +++++-----postscaler (lascia sempre 0)
     //            +----timer on   (sempre a 1 per accendere il timer)
     //             ++--prescaler  (se c'è bisogno di cambio frequenza, cambiare prescaler)
     PR2=255;   // periodo pwm
     //----TMR4----
     T4CON=0b00000110;
     PR4=255;
     //----CCP MODULES SETUP----
     CCP4CON=  0B00001100;
     //              ++----SETTARE QUESTI DUE PER PWM MODE, TUTTI GLI ALTRI BIT A ZERO
     CCP5CON=0x0F; //settato bit0/1 HIGH perchè non vengono usati

     //dobbiamo configurare quale timer abbiamo associato al nostro CCP. Setta timer2 in ccptmrs
     CCPTMRS1= 0b00000001;   //associo timer2 a CCP55,  associo timer4 a CCP4

     CCPR5L= 64; // un quarto di duty cycle
     CCPR4L= 64; // un quarto di dyty cycle

     TRISE.RE2=0;     //risetto i pin in uscita per attivare il pwm
     TRISD.RD1=0;

     while(1){ // fade in e out dei led
       if(dir==0){
         CCPR5L++;
         CCPR4L++;
       }
       else {
       CCPR5L--;
       CCPR4L--;
       }
       if(CCPR5L==255) dir=1;
       else if(CCPR5L==0) dir=0;
       delay_ms(4);
     }
}
```

## ASSEMBLY  


### STATUS register
Lo STATUS è uno dei registri più importanti del microcontrollore. Esso contiene dei bit legati alle operazioni effettuate dall’ALU, il bit dello stato di RESET e i bit del controllo del paging dei banchi di memoria.

#### Quali istruzioni affliggono lo STATUS register?
- Tutte le operazioni di addizione (ADDWF ADDLW), sottrazione (SUBWF SUBLW) affliggono i bit C (carry) DC (digit carry) Z (Zero bit)
- Le rotate left e rotate right (RLF RRF) affliggono il bit C (carry) perché ho bisogno di 1 bit da salvare da qualche parte mentre sto shiftando

- Tutte le operazioni di incremento e decremento non condizionali affliggono il bit Z
- Le operazioni logiche ANDWF CLEARF CLEARW COMF IORWF MOVF XORWF ANDLW IORLW  XORLW affliggono il bit Z
- Tutte le operazioni rimanenti non affliggono lo status (esempio lo swap dei nibbles)



### Salti e return in assembly
- **`GOTO XXX`**: Salto assoluto. Aggiorna il PC ad un’etichetta designata. La dimensione dell’indirizzo dell’etichetta è di 11 bit È sensibile al paging, infatti il bit 12 e 13 vengono acchiappati da PCLATH. Quindi usare BANKSEL. Nei PIC18 il PC è a 21 bit assoluti, il goto è a 20bit quindi può spostarti ovunque in 2M di memoria (in due cicli), ma c’è il problema del paging con PCLATU (non più H)
- **`BRA XXX`**: Salto relativo. Posso spostarmi di massimo +-1023 posizioni dall’indirizzo di partenza. Essendo un salto condizionale partendo dal valore del PC corrente, non è affetto dal paging (non è disponibile nei PIC16).
- **`RETFIE`**: specifica il return dall’interrupt (quindi sposta quello che c’era nel Top Of Stack nel PC) e riattiva il general interrupt GIE (che è stato disattivato all’ingresso della routine dell’interrupt).
- **`CALL`**: va all’etichetta, esattamente come il goto, quindi è affetto da paging (USARE BANKSEL). Si salva nello stack il PC corrispondente alla posizione di chiamata. Appena la routine chiamata dal CALL, il PC viene rishiftato al chiamante.


### Direttive ASM
- **`\#include`**: tale e quale a C++. Si usa con #include <p18f452.inc>
UDATA: dichiara l’inizio di una sezione di dati non inizializzati. Per riservare lo spazio in questa sezione bisogna utilizzare la direttiva RES. L’utilizzo è “LABEL res #byte”.Se non vengono specificate label e indirizzo, (es VARIABLES_IN_BANK udata 0x20) si scrive .udata e il linker fa tutto da sé.
- **`BANKSEL XXX`**: Serve per selezionare automaticamente il bank in base all’etichetta desiderata. Quindi se voglio selezionare un TRISB e non so dove sono, scrivo BANKSEL TRISB. Questo mi permette di poter mettere (con le dovute precauzioni della scelta del micro) lo stesso codice su un altro micro senza preoccuparmi del Bank da selezionare.
- **`CODE XXXX`**: (indirizzo opzionale) significa che il linker può piazzare il codice nella program memory all’indirizzo specifico XXXX segnalato dall’utente, oppure viene lasciata libera la scelta dell’indirizzo al linker se XXXX non viene specificato.
- **`END`**: specifica all’assembler che questa è la fine del file asm. Ogni file asm deve necessariamente finire con la direttiva END. Se così non fosse, l’assembler continuerebbe a passare tutta la memoria.
equ: analogo del define del c++. Si scrive come “MYPORT equ PORTD” (etichetta eq nome_indirizzo_in_RAM).
#### Differenze tra pseudo-istruzioni, macro, direttive
- **Direttiva**: una direttiva assembly è un comando utilizzato a livello software che compare nel source code ma non è direttamente traducibile come opcode. Di conseguenza una direttiva non compare nell’instruction set del datasheet del PIC ma nella User’s guide del MPASM™ Assembler.
- **Pseudo-istruzione**: istruzione ASM scritta con parole diverse in modo tale da agevolare la memorizzazione. Per esempio, nel PIC16 , c’è `MOVWF`, mentre la sua “complementare” dovrebbe essere `MOVFW`. Nelle istruzioni del datasheet però esiste solo `MOVF, w`. L’assemblatore via software permette di tradurre direttamente `MOVF,w` tramite la pseudo-istruzione `MOVFW`, in modo tale da avere meno confusione nel codice e migliore memorizzazione.
- **Macro**: può essere considerata come “un’istruzione” a livello di sviluppo software, ma in realtà è **un gruppo di istruzioni unificate sotto un unico nome**. A differenza delle funzioni in linguaggio C, la macro è una sostituzione **in-line**. Questo significa che non viene effettuata una call, non viene cambiato il PC, non viene allocato spazio nello stack. Il contenuto della macro viene semplicemente inserito in quel punto del programma. Per l’utilizzo leggi la User’s guide del MPASM™ Assembler.

### Scrivere programmi in pseudo codice
Un processo alla base della scrittura di algoritmi per i neofiti della programmazione è **lo sviluppo iniziale in pseudo-codice** .  
Pur essendo all'apparenza "troppo base e scontato", questo è un ottimo metodo sia per pensare all'algoritmo che per debuggare in caso di ricontrollo del codice.  
Facciamo immediatamente un esempio (tenendo a mente le istruzioni del PIC16):

Vogliamo comparare due variabili <code>PIPPO</code>  e <code>PLUTO</code>, ponendo come condizione vera quando PIPPO è maggiore di PLUTO. In linguaggio C tutto ciò è intuitivo:
<code>if(PIPPO>PLUTO)</code>. Ma nelle istruzioni del PIC16 non c'è il confronto maggiore o minore, **possiamo usare solo addizioni, sottrazioni e operazioni logiche o bit a bit**.  
Di conseguenza andiamo a scrivere un po' di pseudo codice utilizzando solo queste operazioni:  
<code>prendi risultato            
carica PIPPO su risultato   
sottrai PLUTO a risultato   
...   
se risultato è negativo
->PIPPO è minore di PLUTO  
...  
se risultato è positivo  
->PIPPO è maggiore di PLUTO
</code>

Ora dobbiamo convertire tutto questo pseudo codice in qualcosa che il compilatore può capire.  
**In questo esempio ci sono due criticità** :  
1. è necessario avere un'ulteriore variabile risultato -> **uso il working register**
2. devo controllare il segno di risultato -> lo **STATUS** contiene il bit carry che viene portato a 1 nel caso un'operazione restituisca una variabile negativa

Vado dunque a scrivere il codice:

```
MOVFW PIPPO ; sposto PIPPO nel working
SUBFW PLUTO ; sottraggo PLUTO al working
BANKSEL STATUS ; vado a selezionare il bank in cui sta STATUS
BTFSS STATUS, c; Se PIPPO-PLUTO<0  ALLORA SKIPPA
GOTO PIPPO_MAGGIORE  ;se il bit test non non skippa allora PIPPO è maggiore
GOTO PIPPO_MINORE; se il bit test skippa allora PIPPO è minore

```
