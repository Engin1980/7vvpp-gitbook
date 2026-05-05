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









...

## Příklad

### Reálný scénář: **start aplikace po inicializaci všech závislostí**

Tohle je **klasický backendový případ**.

#### Kontext

Startuješ serverovou aplikaci a máš několik **nezávislých init úloh**:

* připojení k databázi
* inicializace cache
* načtení konfigurace
* warm‑up nějaké služby

👉 **Aplikace NESMÍ začít obsluhovat requesty, dokud není všechno hotové.**

To je **jednorázová synchronizace** → přesně `CountDownLatch`.

***

### Proč je `CountDownLatch` ideální

* počet úloh je **známý dopředu**
* každá úloha:
  * běží paralelně
  * po dokončení „odškrtne hotovo“
* hlavní thread:
  * **čeká**
  * po dokončení všech pokračuje

```java
import java.util.concurrent.CountDownLatch;

public class AppStartup {

    private static final int TASKS = 3;

    public static void main(String[] args) throws InterruptedException {

        CountDownLatch startupLatch = new CountDownLatch(TASKS);

        new Thread(() -> initDatabase(startupLatch)).start();
        new Thread(() -> initCache(startupLatch)).start();
        new Thread(() -> loadConfiguration(startupLatch)).start();

        System.out.println("Aplikace čeká na inicializaci...");
        startupLatch.await(); // BLOKUJE, dokud counter > 0

        System.out.println("✅ Vše inicializováno, aplikace startuje.");
        startServer();
    }

    static void initDatabase(CountDownLatch latch) {
        try {
            System.out.println("Inicializace DB...");
            Thread.sleep(1500);
            System.out.println("DB připravena");
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            latch.countDown();
        }
    }

    static void initCache(CountDownLatch latch) {
        try {
            System.out.println("Inicializace cache...");
            Thread.sleep(1000);
            System.out.println("Cache připravena");
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            latch.countDown();
        }
    }

    static void loadConfiguration(CountDownLatch latch) {
        try {
            System.out.println("Načítání konfigurace...");
            Thread.sleep(800);
            System.out.println("Konfigurace načtena");
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            latch.countDown();
        }
    }

    static void startServer() {
        System.out.println("🚀 Server běží");
    }
}
```

### Co je zde **důležité konceptuálně**

#### 1. `await()` = **čekám, dokud nebude všechno hotové**

JavastartupLatch.await();\
Zobrazit více řádků

* hlavní thread:
  * **nepustí aplikaci dál**
  * dokud `count == 0`

***

#### 2. `countDown()` = „já už jsem hotový“

Javalatch.countDown();\
Zobrazit více řádků

* každý task:
  * sníží počítadlo o 1
* **nezáleží na pořadí**
* poslední, kdo zavolá `countDown()`, **odblokuje `await()`**

***

#### 3. Proč je latch **jednorázový**

Jakmile:

```
count == 0
```

* latch je „mrtvý“
* **nejde resetovat**
* kdykoliv další `await()` projde okamžitě

➡️ to je záměr\
➡️ startup se děje **jen jednou**

### Další **reálné** použití `CountDownLatch`

Bez pohádek:

* čekání na:
  * dokončení batch jobů
  * paralelní validaci dat
  * dočtení více souborů
  * doběhnutí workerů před shutdownem
* testy:
  * počkej, až async operace proběhnou
  * pak teprve assert
