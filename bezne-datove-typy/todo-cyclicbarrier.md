# CyclicBarrier

## Popis

...

## Příklad

### Konkrétní příklad: simulace počasí po krocích

* 3 regiony
* 5 časových kroků
* V každém kroku:
  * region spočítá **nový stav z předchozího**
  * pak čeká, než dopočítají všichni

```
import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CyclicBarrier;

public class WeatherSimulation {

    static final int REGIONS = 3;
    static final int STEPS = 5;

    public static void main(String[] args) {

        CyclicBarrier barrier = new CyclicBarrier(
                REGIONS,
                () -> System.out.println("=== Časový krok dokončen, commit stavu ===\n")
        );

        for (int i = 1; i <= REGIONS; i++) {
            int regionId = i;
            new Thread(() -> simulateRegion(regionId, barrier)).start();
        }
    }

    static void simulateRegion(int regionId, CyclicBarrier barrier) {
        try {
            for (int step = 1; step <= STEPS; step++) {

                // výpočet regionu pro daný časový krok
                System.out.println(
                        "Region " + regionId + " počítá krok " + step
                );

                // simulace různě náročných výpočtů
                Thread.sleep(300 + regionId * 200);

                System.out.println(
                        "Region " + regionId + " hotov pro krok " + step
                );

                // čekání na ostatní regiony
                barrier.await();
            }
        } catch (InterruptedException | BrokenBarrierException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

### Proč je tu `CyclicBarrier` **nutný**

#### Bez bariéry by se stalo:

```
Region 1: krok 2 ─────▶
Region 2:    krok 1 ─────▶
Region 3: krok 3 ─────▶
```

➡️ regiony by:

* počítaly s **nekompletními nebo budoucími daty**
* porušily invariant _„všichni jsou ve stejném čase“_

***

#### S bariérou:

```
Krok 1:
  R1 ─┐
  R2 ─┼── barrier ─▶ pokračují
  R3 ─┘

Krok 2:
  R1 ─┐
  R2 ─┼── barrier ─▶ pokračují
  R3 ─┘
```

➡️ **globální synchronní commit času**

Tohle přesně **odpovídá reálným simulátorům**:

* fyzikální enginy
* klimatické modely
* multi-agent systémy
* ekonomické simulace
* herní servery (tick-based logika)

### Proč ne `CountDownLatch`

* `CountDownLatch`:
  * pouze **jednorázový**
* tady:
  * **opakované kroky**
  * bariéra se **automaticky resetuje**

➡️ přesně definice `CyclicBarrier`.

***

### Proč ne `Semaphore`

* `Semaphore` řeší **kolik vláken může běžet**
* **neřeší fáze**
* tady nepotřebuješ omezovat paralelismus, ale **synchronizovat čas**
