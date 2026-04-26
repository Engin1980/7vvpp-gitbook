# CountDownLatch

## Popis

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
