# Phaser

## Motivace

Phaser je nejpružnější a nejpokročilejší synchronizační nástroj v Javě, který kombinuje (a rozšiřuje) schopnosti `CountDownLatch` a `CyclicBarrier`. Jeho název je odvozen od „fází“ – je určen pro synchronizaci vláken v libovolném počtu po sobě jdoucích kroků.

Hlavní motivací je potřeba synchronizace v situacích, kde se počet účastníků (vláken) může v průběhu času měnit. Zatímco u bariéry je počet vláken fixní, Phaser umožňuje vláknům se dynamicky registrovat, odregistrovat nebo se „přihlásit“ k odběru dalších fází.

Kdy se typicky používá?

* Dynamické paralelní algoritmy: Například prohledávání webu (web crawling), kde vlákno narazí na další odkazy a dynamicky vytvoří (zaregistruje) nová vlákna pro další fázi stahování.
* Vícefázové pipeline procesy: Zpracování dat, kde se v každém kroku (fázi) může měnit počet aktivních výpočetních jednotek.
* Nahrazení bariér a závory: Phaser lze nakonfigurovat tak, aby se choval jako kterýkoliv z předchozích nástrojů, ale s nižší režií a vyšší flexibilitou.

## Implementace

V Javě je tento mechanismus implementován třídou `java.util.concurrent.Phaser`. Na rozdíl od bariéry nepoužívá termín „vlákna“, ale „účastníci“ (parties). Phaser udržuje číslo aktuální fáze (začíná od 0).

#### Základní operace

Třída nabízí tyto klíčové operace:

* Konstruktor `new Phaser(int parties)`: Vytvoří phaser s počátečním počtem registrovaných účastníků.
* `register()` / `bulkRegister(int)`: Dynamicky přidá jedno nebo více účastníků k phaseru.
* `arriveAndAwaitAdvance()`: Signalizuje příchod k synchronizačnímu bodu a čeká, dokud nedorazí ostatní. (Ekvivalent `await()` u bariéry).
* `arriveAndDeregister()`: Signalizuje dokončení práce a odhlásí vlákno z budoucích fází. Vlákno už na nikoho nečeká a pokračuje/končí.
* `arrive()`: Signalizuje příchod, ale neblokuje vlákno – vlákno pokračuje dál, ale phaser ví, že tato část práce je hotová.

#### Výjimky a stavy při čekání

Phaser se chová k chybám velmi specificky a je odolnější než `CyclicBarrier`:

1\. Žádné poškození (No Broken State)

Phaser nemá stav „poškozené bariéry“. Pokud v jednom vlákně dojde k výjimce, ostatní vlákna mohou dál čekat. Programátor musí zajistit, aby i chybující vlákno zavolalo `arriveAndDeregister()`, jinak by ostatní čekali věčně.

2\. Terminace (Termination)

Phaser může být ukončen (metoda `forceTermination()`). V takovém případě všechny metody `await` okamžitě skončí. Stav phaseru lze sledovat metodou `isTerminated()`.

3\. Flexibilní bariérová akce

Místo předávání `Runnable` v konstruktoru lze přepsat metodu `onAdvance(int phase, int registeredParties)`. Ta se spustí při každém přechodu do další fáze. Pokud vrátí `true`, phaser se ukončí.

## Příklad

Představme si automatizovaný proces nasazení aplikace (CI/CD), který má dvě fáze: 1. Kompilace a Testování, 2. Distribuce na servery. V druhé fázi se k procesu může dynamicky přidat monitorovací vlákno.

```java
import java.util.concurrent.Phaser;

public class DeploymentSystem {

    public static void main(String[] args) {
        // Inicializujeme Phaser pro 1 hlavní řídící vlákno
        Phaser phaser = new Phaser(1);

        // Spuštění workerů pro fázi "Build & Test"
        startTask(new Worker("Kompilace modulu A", phaser));
        startTask(new Worker("Kompilace modulu B", phaser));

        // 1. Synchronizační bod - Build fáze
        int phase = phaser.getPhase();
        phaser.arriveAndAwaitAdvance();
        System.out.println(">>> Fáze " + phase + " (Build) dokončena.");

        // Dynamická registrace nového účastníka pro distribuci
        startTask(new Worker("Distribuce na Produkci", phaser));

        // 2. Synchronizační bod - Deployment fáze
        phase = phaser.getPhase();
        phaser.arriveAndAwaitAdvance();
        System.out.println(">>> Fáze " + phase + " (Deployment) dokončena.");

        // Odhlášení hlavního vlákna a ukončení
        phaser.arriveAndDeregister();
    }

    private static void startTask(Worker worker) {
        worker.phaser.register(); // Dynamická registrace nového úkolu
        new Thread(worker).start();
    }

    static class Worker implements Runnable {
        private final String taskName;
        private final Phaser phaser;

        Worker(String taskName, Phaser phaser) {
            this.taskName = taskName;
            this.phaser = phaser;
        }

        @Override
        public void run() {
            try {
                System.out.println("[" + taskName + "] probíhá...");
                Thread.sleep((long) (Math.random() * 2000));
                
                System.out.println("[" + taskName + "] hotovo. Hlásím příchod k bariéře.");
                // Signalizace dokončení a odhlášení (vlákno končí)
                phaser.arriveAndDeregister();
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
    }
}
```

## Shrnutí

<table data-header-hidden><thead><tr><th width="169"></th><th width="290"></th><th width="266"></th></tr></thead><tbody><tr><td><strong>Vlastnost</strong></td><td><strong>CyclicBarrier</strong></td><td><strong>Phaser</strong></td></tr><tr><td>Počet účastníků</td><td>Fixní (nastaven v konstruktoru).</td><td>Dynamický (lze měnit za běhu).</td></tr><tr><td>Opakovatelnost</td><td>Ano (automaticky).</td><td>Ano (postupné fáze 0, 1, 2...).</td></tr><tr><td>Flexibilita</td><td>Všechna vlákna čekají na všechny.</td><td>Vlákna mohou jen signalizovat bez čekání (<code>arrive</code>).</td></tr><tr><td>Stav po chybě</td><td>Rozbitá bariéra (Broken).</td><td>Robustní (pokračuje dál).</td></tr></tbody></table>











fa











## Představení

...

## Příklad

### Problém (reálný, rozšířený)

Simulace světa po **časových krocích (ticks)**:

* svět je rozdělen na **regiony**
* každý region se počítá paralelně
* po **každém kroku musí všichni regiony počkat**
* **během simulace může region vzniknout nebo zaniknout**
  * přidání / odebrání aktéra
  * dynamické rozdělení mapy
  * load balancing

👉 **Počet účastníků NENÍ konstantní**

➡️ `CyclicBarrier` = špatně\
➡️ `Phaser` = přesně na tohle

***

### Proč `CyclicBarrier` nestačí

`CyclicBarrier`:

* počet vláken **fixní**
* nelze dynamicky:
  * registrovat nové
  * deregistrovat zmizelé

Simulace tohle **běžně potřebují**.

***

### Mentální model

> Phaser = **opakovaný latch + dynamická bariéra**

Umí:

* fáze (jako `CyclicBarrier`)
* dynamické přihlašování / odhlašování vláken
* centrální řízení konce simulace

***

### Scénář

* začínáme se 3 regiony
* běžíme 5 časových kroků
* region 2 se po 3. kroku **odpojí**
* simulace pokračuje správně dál

```java
import java.util.concurrent.Phaser;

public class WeatherSimulationWithPhaser {

    static final int INITIAL_REGIONS = 3;
    static final int STEPS = 5;

    public static void main(String[] args) {

        Phaser phaser = new Phaser(INITIAL_REGIONS) {
            @Override
            protected boolean onAdvance(int phase, int registeredParties) {
                System.out.println(
                        "=== Dokončen krok " + phase +
                        " | aktivní regiony: " + registeredParties + " ===\n"
                );

                // simulace běží 5 kroků
                return phase >= STEPS - 1 || registeredParties == 0;
            }
        };

        for (int i = 1; i <= INITIAL_REGIONS; i++) {
            int regionId = i;
            new Thread(() -> simulateRegion(regionId, phaser)).start();
        }
    }

    static void simulateRegion(int regionId, Phaser phaser) {
        int step = 0;

        while (!phaser.isTerminated()) {
            System.out.println("Region " + regionId + " počítá krok " + step);

            try {
                Thread.sleep(400 + regionId * 200);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            }

            // Region 2 se po kroku 2 dynamicky odpojí
            if (regionId == 2 && step == 2) {
                System.out.println("Region 2 končí simulaci");
                phaser.arriveAndDeregister();
                return;
            }

            System.out.println("Region " + regionId + " hotov pro krok " + step);
            phaser.arriveAndAwaitAdvance();

            step++;
        }
    }
}
``
```

### Co se tu děje (stručně, technicky)

#### 1. Každý region je **registrovaná party**

Javanew Phaser(INITIAL\_REGIONS)\
\`\`\
Zobrazit více řádků

Na rozdíl od `CyclicBarrier`:

* party **nejsou spojené s konkrétním vláknem**
* můžeš je přidávat/ubírat za běhu

***

#### 2. Synchronizace po krocích

Javaphaser.arriveAndAwaitAdvance();\
Zobrazit více řádků

\=

* „já jsem hotov“
* „počkáme, až budou hotovi všichni“

Stejně jako `CyclicBarrier.await()`\
**ale** bez fixního počtu účastníků.

***

#### 3. Dynamické odhlášení

Javaphaser.arriveAndDeregister();\
Zobrazit více řádků

Tohle:

* sníží počet aktivních regionů
* **nezablokuje ostatní**
* bariéra se přepočítá

➡️ s `CyclicBarrier` **nemožné**

***

#### 4. Centrální řízení simulace

Javaprotected boolean onAdvance(int phase, int parties)\
Zobrazit více řádků

* zavolá se **jednou po každém kroku**
* můžeš:
  * logovat
  * commitovat data
  * rozhodnout o ukončení simulace
