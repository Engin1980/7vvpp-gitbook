# Phaser

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
