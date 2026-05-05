# Semaphore

## Motivace

V kontextu souběžného programování v Javě je Semaphore (semafor) synchronizační nástroj, který slouží k řízení přístupu k omezenému počtu sdílených prostředků. Na rozdíl od klasického zámku (`Lock` nebo `synchronized`), který obvykle povoluje přístup pouze jednomu vláknu najednou, semafor udržuje sadu povolenek (permits).

Představte si semafor jako recepci v knihovně, která má k dispozici pouze 5 studoven.

1. Pokud přijde student a je volná studovna, dostane klíč (povolenku) a jde pracovat.
2. Pokud přijde další student a všech 5 studoven je obsazených, musí počkat v hale, dokud někdo jiný klíč nevrátí.
3. Jakmile jeden student odejde a vrátí klíč, první čekající v řadě si jej může vzít a obsadit studovnu.

Hlavní využití:

* Omezení propustnosti (Throttling): Ochrana externích služeb nebo databází před zahlcením příliš velkým množstvím požadavků.
* Správa fondů (Resource Pooling): Řízení přístupu k pevnému počtu připojení, hardwarových zařízení (tiskárny) nebo licencí.

## Implementace

V Javě je semafor implementován jako třída `java.util.concurrent.Semaphore`, která vnitřně udržuje čítač dostupných povolenek. Klíčovými metodami jsou `acquire()` pro získání povolenky a `release()` pro její vrácení.

**Konstruktory**

* `new Semaphore(int permits)`: Vytvoří semafor s daným počtem povolenek.
* `new Semaphore(int permits, boolean fair)`: Pokud je nastaveno `fair = true`, semafor zaručuje, že vlákna budou obsluhována v pořadí, v jakém o povolenku požádala (FIFO).

**Klíčové metody**

* `acquire()`: Vlákno požádá o povolenku. Pokud není žádná volná, vlákno se zablokuje a čeká.
* `release()`: Vlákno vrátí povolenku zpět semaforu. Tím se uvolní místo pro jedno z čekajících vláken.
* `tryAcquire()`: Pokusí se získat povolenku bez čekání. Pokud není volná, vrátí ihned `false`.
* `availablePermits()`: Vrátí aktuální počet volných povolenek.

## Příklad

Uvažujme jednoduchý příklad, kdy může spousta vláken přistupovat do databáze, ale máme pouze omezený maximální počet otevřených připojení (například z důvodu free-tier databáze).

#### Princip

* `Semaphore(10)` → maximálně 10 vláken může současně pokračovat
* `acquire()` → čekej, dokud není volné místo
* `release()` → uvolni místo pro další

V tomto příkladu máme servis, který by mohl být volán stovkami vláken, ale my chceme zajistit, aby do databáze v jeden okamžik přistupovalo maximálně 10 vláken, aby nedošlo k jejímu přetížení.

```java
import java.util.concurrent.Semaphore;

public class DatabaseThrottler {
    // Definujeme semafor s 10 povolenkami
    private final Semaphore semaphore = new Semaphore(10);

    public void fetchData(String query) {
        try {
            // Vlákno se pokusí získat povolenku
            System.out.println(Thread.currentThread().getName() + " čeká na přístup k DB...");
            semaphore.acquire(); 
            
            try {
                // Kritická sekce - přístup k omezenému zdroji
                System.out.println(Thread.currentThread().getName() + " PRACUJE S DB na dotazu: " + query);
                Thread.sleep(2000); // Simulace náročné operace
            } finally {
                // Velmi důležité: Povolenku musíme VŽDY vrátit ve finally bloku
                System.out.println(Thread.currentThread().getName() + " uvolňuje DB.");
                semaphore.release();
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }

    public static void main(String[] args) {
        DatabaseThrottler throttler = new DatabaseThrottler();

        // Simulujeme 6 vláken, která chtějí přistoupit k DB
        for (int i = 1; i <= 6; i++) {
            final int id = i;
            new Thread(() -> throttler.fetchData("Query #" + id)).start();
        }
    }
}
```

Důležité poznatky:

1. Binární semafor: Pokud vytvoříte semafor s 1 povolenkou, funguje podobně jako `Lock` (vzájemné vyloučení – Mutex). Rozdíl je v tom, že **u semaforu může povolenku uvolnit i jiné vlákno, než které ji získalo**, což u klasických zámků (`ReentrantLock`) nelze.
2. Zapomenuté uvolnění: Pokud vlákno získá povolenku a zapomene zavolat `release()` (např. kvůli neošetřené výjimce), povolenka je navždy ztracena. Proto se `release()` musí vždy volat v bloku `finally`.
3. Virtuální vlákna: V moderní Javě (21+) jsou semafory doporučovaným způsobem, jak omezit souběžnost (concurrency) u virtuálních vláken (vysvětleno později v [Broken link](/broken/pages/TzCVhX4fNiGwDdW9Pdld "mention")) místo používání pevných vláknových fondů (Fixed Thread Pools - [Broken link](/broken/pages/PqD3painQyxNMc3lEb6C "mention")).
