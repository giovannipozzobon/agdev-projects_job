# Analisi del Programma FlAgon - Dispositivo Agonlight e Console8

## Panoramica

FlAgon è un clone del gioco Flappy Bird sviluppato per il dispositivo Agonlight, utilizzando le funzionalità grafiche avanzate della Console8. Il programma è scritto in C e sfrutta una libreria custom chiamata `acurses` (Agon Curses) che emula le funzionalità della libreria ncurses standard, ma ottimizzata per il VDP (Video Display Processor) dell'Agonlight.

## Architettura del Sistema Grafico

### 1. Modalità Video e Double Buffering

Il programma utilizza la **modalità video 132** ([acurses.h:215](flagon/src/acurses.h#L215)):

```c
vdp_mode(132); // Modalità 80x30, 16 colori con double buffering
```

Questa modalità offre:
- Risoluzione: 80 colonne × 30 righe
- 16 colori (ridotti da 64 per supportare il double buffering)
- **Double buffering integrato** per eliminare il flickering

#### Funzionamento del Double Buffering

Il double buffering è gestito attraverso la funzione `refresh()` ([acurses.h:406-410](flagon/src/acurses.h#L406-L410)):

```c
int refresh(void) {
    vdp_swap();
    return true;
}
```

La funzione `vdp_swap()` scambia i buffer video:
- Il programma disegna su un buffer nascosto (back buffer)
- Al momento del `refresh()`, il buffer nascosto diventa visibile (front buffer)
- Questo elimina il flickering che si vedrebbe disegnando direttamente sullo schermo

### 2. Sistema di Bitmap Mappate a Caratteri

Una caratteristica innovativa di FlAgon è l'uso del sistema di **character-to-bitmap mapping** della Console8, che permette di mappare caratteri ASCII a bitmap grafiche personalizzate.

#### Caricamento delle Bitmap

Il processo avviene nella funzione `init_graphics()` ([driver.c:359-678](flagon/src/driver.c#L359-L678)):

**Esempio per il tubo verticale (pipe):**

```c
// Seleziona il buffer bitmap 0x80
vdp_adv_select_bitmap(0x80);

// Carica i dati bitmap (56 pixel larghezza × 8 pixel altezza)
vdp_load_bitmap(56, 8, pipedata);

// Mappa il carattere 0x80 alla bitmap 0x80
vdp_map_char_to_bitmap(0x80, 0x80);
```

#### Bitmap Caricate

Il programma carica le seguenti bitmap:

1. **0x80**: Sezione verticale del tubo (56×8 pixel)
2. **0x81**: Estremità superiore del tubo (56×8 pixel)
3. **0x82**: Estremità inferiore del tubo (56×8 pixel)
4. **0x83-0x86**: Animazioni dell'uccello (16×18 pixel):
   - 0x83: Uccello in caduta
   - 0x84-0x86: Frame di animazione del volo
5. **0x88-0x8B**: Tile del pavimento animato (8×24 pixel)

### 3. Gestione della Palette Colori

Il programma ridefinisce la palette per ottimizzare i 16 colori disponibili ([driver.c:365-394](flagon/src/driver.c#L365-L394)):

```c
init_color(0, 0, 0, 0);           // Nero
init_color(1, 0x00, 0x55, 0x00);  // Verde scuro
init_color(2, 0x55, 0xAA, 0x00);  // Verde medio
init_color(3, 0x00, 0xAA, 0x00);  // Verde
init_color(4, 0x00, 0xFF, 0x00);  // Verde brillante
// ... altre definizioni colore
init_color(14, 0x55, 0xAA, 0xFF); // Azzurro cielo
init_color(15, 255, 255, 255);    // Bianco
```

Il formato RGB utilizza i livelli FabGL: 0x00, 0x55, 0xAA, 0xFF.

## Gestione dello Scrolling

### 1. Scrolling Orizzontale dei Tubi

Lo scrolling è implementato tramite un sistema di coordinate virtuali. Ogni tubo ha una struttura `vpipe` ([driver.c:26-41](flagon/src/driver.c#L26-L41)):

```c
typedef struct vpipe {
    float opening_height;  // Altezza dell'apertura (frazione dell'altezza schermo)
    int center;           // Posizione X del centro del tubo
} vpipe;
```

#### Aggiornamento della Posizione

La funzione `pipe_refresh()` gestisce il movimento ([driver.c:188-204](flagon/src/driver.c#L188-L204)):

```c
void pipe_refresh(vpipe *p) {
    // Se il tubo esce dallo schermo a sinistra
    if(p->center + PIPE_RADIUS < 0) {
        p->center = NUM_COLS + PIPE_RADIUS;  // Riposiziona a destra
        p->opening_height = rand() / ((float) INT_MAX) * 0.5 + 0.25;
        score++;  // Incrementa il punteggio
    }
    p->center--;  // Muove il tubo a sinistra
}
```

**Caratteristiche dello scrolling:**
- I tubi si muovono di **1 colonna a sinistra** per frame
- Quando escono dallo schermo, vengono **riciclati** riapparendo a destra
- Ogni tubo riciclato genera una nuova altezza casuale dell'apertura
- Il sistema usa **2 tubi** (`p1` e `p2`) distanziati per creare un flusso continuo

### 2. Scrolling del Pavimento

Il pavimento crea un'illusione di movimento attraverso **4 frame di animazione** ([driver.c:137-181](flagon/src/driver.c#L137-L181)):

```c
void draw_floor_and_ceiling(int i) {
    switch (i) {
        case 0: mvaddstr(NUM_ROWS+1, 0, floor4); break;
        case 1: mvaddstr(NUM_ROWS+1, 0, floor3); break;
        case 2: mvaddstr(NUM_ROWS+1, 0, floor2); break;
        case 3: mvaddstr(NUM_ROWS+1, 0, floor1); break;
    }
}
```

Chiamata nel main loop come:
```c
draw_floor_and_ceiling(frame & 3);  // Cicla tra 0-3
```

Le stringhe `floor1-floor4` contengono pattern alternati di caratteri 0x88-0x8B, che sono mappati a bitmap che creano un effetto di scorrimento quando alternate.

### 3. Fisica di Flappy

Il movimento verticale segue una **traiettoria parabolica** ([driver.c:259-261](flagon/src/driver.c#L259-L261)):

```c
int get_flappy_position(flappy f) {
    return f.h0 + V0 * f.t + 0.5 * GRAV * f.t * f.t;
}
```

Dove:
- `h0`: Altezza al momento dell'ultimo salto
- `V0`: Velocità iniziale (-0.5, verso l'alto)
- `GRAV`: Accelerazione gravitazionale (0.05)
- `t`: Tempo trascorso dall'ultimo salto

## Ciclo di Aggiornamento dello Schermo

### Sequenza del Main Loop

Il ciclo principale ([driver.c:748-813](flagon/src/driver.c#L748-L813)) segue questa sequenza:

```c
while(!leave_loop) {
    // 1. TIMING: Attende per mantenere 20 FPS
    napms((unsigned int) (1000 / TARGET_FPS));  // ~50ms di delay

    // 2. INPUT: Legge i comandi da tastiera
    ch = wgetch(stdscr);
    switch (ch) {
        case KEY_UP: // Salto
            f.h0 = get_flappy_position(f);
            f.t = 0;
            break;
        default:
            f.t++;  // Continua la caduta
    }

    // 3. CLEAR: Pulisce il buffer nascosto
    clear();

    // 4. DRAW FLOOR: Disegna il pavimento animato
    draw_floor_and_ceiling(frame & 3);

    // 5. DRAW PIPES: Disegna i tubi
    draw_pipe(p1, 0x80, 0x81, 0x82, 0, NUM_ROWS - 1);
    draw_pipe(p2, 0x80, 0x81, 0x82, 0, NUM_ROWS - 1);

    // 6. UPDATE PIPES: Aggiorna posizioni dei tubi
    pipe_refresh(&p1);
    pipe_refresh(&p2);

    // 7. DRAW FLAPPY: Disegna l'uccello con animazione
    if(draw_flappy(f)) {
        restart = 1;  // Collision detected
    }

    // 8. DRAW UI: Disegna punteggio
    mvprintw(NUM_ROWS + FLOOR_ROWS, SCORE_START_COL - bdigs - sdigs,
             " Score: %d  Best: %d", score, best_score);

    // 9. REFRESH: Scambia i buffer (visualizza il frame)
    refresh();

    // 10. INCREMENT: Avanza al prossimo frame
    frame++;
}
```

### Ottimizzazioni Chiave

1. **Double Buffering**: Elimina completamente il flickering
2. **Character-to-Bitmap**: Permette grafica complessa con API testuali
3. **Full Redraw**: Ogni frame viene ridisegnato completamente sul buffer nascosto
4. **Frame Rate Fisso**: 20 FPS garantisce movimento fluido

## Rendering dei Tubi

La funzione `draw_pipe()` ([driver.c:231-250](flagon/src/driver.c#L231-L250)) disegna un tubo:

```c
void draw_pipe(vpipe p, char vch, char hcht, char hchb,
               int ceiling_row, int floor_row) {
    int i, start_column;
    start_column = p.center - PIPE_RADIUS;

    if ((start_column) >= 0 && (start_column) < NUM_COLS - 1) {
        // Disegna parte superiore del tubo
        for(i = ceiling_row; i < get_orow(p, 1); i++) {
            mvaddch(i, start_column, vch);  // 0x80: sezione verticale
        }
        mvaddch(i, start_column, hcht);  // 0x81: estremità superiore

        // Disegna parte inferiore del tubo
        for(i = floor_row - 1; i > get_orow(p, 0); i--) {
            mvaddch(i, start_column, vch);  // 0x80: sezione verticale
        }
        mvaddch(i, start_column, hchb);  // 0x82: estremità inferiore
    }
}
```

**Nota importante**: Il tubo è largo solo 1 carattere (56 pixel), ma la bitmap è sufficientemente dettagliata da sembrare un tubo 3D.

## Animazione dell'Uccello

L'animazione di Flappy utilizza 4 frame diversi ([driver.c:326-348](flagon/src/driver.c#L326-L348)):

```c
int draw_flappy(flappy f) {
    int h = get_flappy_position(f);

    // Se in caduta: mostra sprite statico
    if (GRAV * f.t + V0 > 0) {
        mvaddch(h, FLAPPY_COL - 1, 0x83);
    }
    // Se in salita: anima le ali
    else {
        mvaddch(h, FLAPPY_COL, 0x83 + ((frame & 6)/2));
        // Cicla tra 0x83, 0x84, 0x85, 0x86 ogni 2 frame
    }

    return 0;
}
```

## Gestione della Collision Detection

La collisione è verificata in tempo reale ([driver.c:271-281](flagon/src/driver.c#L271-L281)):

```c
int crashed_into_pipe(flappy f, vpipe p) {
    // Verifica se Flappy è orizzontalmente allineato con il tubo
    if (FLAPPY_COL >= p.center - PIPE_RADIUS - 1 &&
        FLAPPY_COL <= p.center + PIPE_RADIUS + 1) {

        // Verifica se Flappy è verticalmente dentro il tubo (non nell'apertura)
        if (get_flappy_position(f) < get_orow(p, 1) + 1 ||
            get_flappy_position(f) > get_orow(p, 0) - 1) {
            return 1;  // Collisione!
        }
    }
    return 0;
}
```

## Libreria Acurses

La libreria `acurses.h` fornisce un'astrazione completa delle funzionalità VDP:

### Funzioni Chiave per la Grafica

1. **Inizializzazione**:
   - `initscr()`: Inizializza il sistema, imposta modalità 132
   - `start_color()`: Inizializza il sistema colori
   - `init_color()`: Definisce colori RGB personalizzati

2. **Disegno**:
   - `mvaddch(y, x, ch)`: Disegna un carattere/bitmap alla posizione
   - `mvaddstr(y, x, str)`: Disegna una stringa
   - `mvprintw(y, x, fmt, ...)`: Disegna testo formattato

3. **Controllo Schermo**:
   - `clear()`: Pulisce il buffer corrente
   - `refresh()`: Scambia i buffer (double buffering)
   - `vdp_swap()`: Funzione VDP per lo swap dei buffer

4. **Input**:
   - `wgetch()`: Legge input da tastiera (non-blocking)
   - `timeout(ms)`: Imposta timeout per l'input

## Conclusioni

FlAgon dimostra un uso efficiente delle capacità della Console8 dell'Agonlight:

1. **Double Buffering**: Rendering senza flickering
2. **Character-to-Bitmap Mapping**: Grafica avanzata con API testuali
3. **Palette Ottimizzata**: 16 colori ben bilanciati per il gioco
4. **Scrolling Efficiente**: Movimento fluido con riciclaggio degli oggetti
5. **Animazione Fluida**: 20 FPS costanti con timing preciso

Il codice è ben strutturato e separa chiaramente la logica di gioco dalla gestione grafica tramite la libreria acurses, rendendo il programma mantenibile e facilmente estendibile.
