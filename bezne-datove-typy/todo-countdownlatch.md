# CountDownLatch

## Motivace

`CountDownLatch` (v překladu „odpočítávací závora“) protipól k semaforu. Zatímco semafor omezuje počet vláken, `CountDownLatch` slouží k synchronizaci, kdy jedno nebo více vláken musí počkat na dokončení operací v jiných vláknech.

Hlavní motivací pro použití `CountDownLatch` je koordinace startu nebo ukončení sady paralelních úloh. Představte si jej jako „startovní bránu“ nebo „kontrolní seznam“ (checklist), který musí být kompletně odškrtán, než se může pokračovat dále.

Kdy se typicky používá?

* Čekání na inicializaci: Hlavní aplikace nesmí začít obsluhovat požadavky, dokud není dokončena inicializace všech subsystémů (např. připojení k DB, načtení konfigurace, kontrola souborů).
* Paralelní testování: Testovací vlákno čeká, dokud 100 simulovaných uživatelů nedokončí svou akci, aby mohl test vyhodnotit celkový výsledek.
* Dostihový start: Synchronizace skupiny vláken tak, aby všechna začala pracovat přesně v jeden okamžik (všechna čekají na uvolnění jedné závory).

## Implementace

V Javě je tento mechanismus implementován třídou `java.util.concurrent.CountDownLatch`. Funguje na principu vnitřního čítače, který se postupně snižuje.

#### Základní operace

Třída nabízí základní operace:

1. Konstruktor `new CountDownLatch(int count)`: Vytvoří závoru s počáteční hodnotou čítače.
2. `await()`: Zablokuje volající vlákno (např. `main`), dokud čítač nedosáhne nuly. Existuje i verze s časovým limitem: `await(long timeout, TimeUnit unit)`.
3. `countDown()`: Sníží hodnotu čítače o jedna. Tuto metodu volají pracovní vlákna poté, co dokončí svou část práce.

### Výjimky při čekání na \`await\`

Metoda `await()` v Javě reaguje na výjimky a stavy vláken následovně:

#### 1. InterruptedException (Přerušení čekajícího vlákna)

Metoda `await()` je tzv. blocking call. Pokud je vlákno, které na závoře čeká, přerušeno jiným vláknem (pomocí `thread.interrupt()`), `await()` okamžitě vyhodí `InterruptedException`.

* Důsledek: Čekající vlákno se "probudí" dříve, než čítač dosáhne nuly.
* Doporučení: V bloku `catch` je nutné buď obnovit příznak přerušení (`Thread.currentThread().interrupt()`), nebo provést korektní ukončení operace.

#### 2. Výjimky v pracovních vláknech

Pozor: Pokud v pracovním vlákně (které má udělat `countDown()`) dojde k `RuntimeException`, toto vlákno zemře. Pokud k tomu dojde před zavoláním `countDown()`, čítač se nesníží.

* Chování: `await()` o této chybě vůbec neví. Bude dál trpělivě čekat na hodnotu 0, která ale kvůli havárii pracovníka nikdy nenastane.

#### 3. Časový limit (Timeout) jako pojistka

Nejlepším způsobem, jak se bránit uváznutí v důsledku výjimek, je používat variantu `await(long timeout, TimeUnit unit)`.

* Návratová hodnota: Tato metoda vrací `boolean`.
  * `true` – čítač dosáhl nuly (vše je v pořádku).
  * `false` – vypršel čas, ale čítač je stále kladný (něco se pokazilo nebo trvalo příliš dlouho).

## Příklad

Představme si start mikroslužby, která vyžaduje úspěšné spuštění tří nezávislých modulů (Databáze, Cache, Messaging), než začne přijímat provoz. Aplikace **nesmí** začít obsluhovat requesty, dokud není všechno hotové.

```java
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class SystemStartup {

    public static void main(String[] args) {
        // Musíme počkat na 3 moduly
        CountDownLatch latch = new CountDownLatch(3);
        ExecutorService executor = Executors.newFixedThreadPool(3);

        System.out.println("Hlavní systém: Čekám na inicializaci modulů...");

        // Spuštění 3 úloh
        executor.execute(new ServiceModule("Databáze", 2000, latch));
        executor.execute(new ServiceModule("Cache", 1000, latch));
        executor.execute(new ServiceModule("Messaging", 3000, latch));

        try {
            // Hlavní vlákno zde čeká, dokud čítač nebude 0
            latch.await();
            System.out.println("Hlavní systém: Všechny moduly jsou připraveny. STARTUJI.");
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            executor.shutdown();
        }
    }
}

class ServiceModule implements Runnable {
    private final String name;
    private final int startupTime;
    private final CountDownLatch latch;

    public ServiceModule(String name, int startupTime, CountDownLatch latch) {
        this.name = name;
        this.startupTime = startupTime;
        this.latch = latch;
    }

    @Override
    public void run() {
        try {
            Thread.sleep(startupTime); // Simulace zavádění modulu
            System.out.println("Modul " + name + " byl úspěšně inicializován.");
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            // Klíčová operace: Odškrtnutí ze seznamu
            latch.countDown();
        }
    }
}
```
