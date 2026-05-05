# CyclicBarrier

## Motivace

CyclicBarrier (cyklická bariéra) je synchronizační nástroj, který umožňuje sadě vláken vzájemně na sebe počkat v určitém bodě (bariéře), než budou moci všechna pokračovat dále. Na rozdíl od `CountDownLatch`, který je určen pro čekání na událost, `CyclicBarrier` je určen pro čekání vláken na sebe navzájem.

Hlavní motivací je rozdělení velkého úkolu na menší části, které běží paralelně, ale v určitém bodě musí synchronizovat své mezivýsledky, než postoupí do další fáze. Přívlastek „cyklická“ znamená, že bariéru lze po průchodu všech vláken znovu použít.

Kdy se typicky používá?

* Vícefázové algoritmy: Simulace nebo výpočty, kde v kroku 1 všechna vlákna počítají svá data a v kroku 2 se tato data musí spojit, než se začne krok 3.
* Paralelní iterativní výpočty: Například matematické maticové operace, kde každé vlákno počítá řádek, ale další iterace závisí na hodnotách z předchozí iterace všech ostatních řádků.
* Synchronizace herních entit: Všechny postavy v multiplayer hře musí dokončit svůj tah, než se herní svět posune do dalšího kola.

## Implementace

V Javě je tento mechanismus implementován třídou `java.util.concurrent.CyclicBarrier`. Funguje na principu vnitřního čítače, který se po dosažení nuly automaticky resetuje na původní hodnotu.

### Základní operace

Třída nabízí tyto základní operace:

* Konstruktor `new CyclicBarrier(int parties)`: Vytvoří bariéru pro daný počet spolupracujících vláken.
* Konstruktor `new CyclicBarrier(int parties, Runnable barrierAction)`: Vytvoří bariéru, která po dosažení stanoveného počtu vláken spustí zadanou akci (bariérovou úlohu). Tuto akci provede poslední vlákno, které k bariéře dorazilo.
* `await()`: Vlákno signalizuje, že dorazilo k bariéře, a zablokuje se, dokud nedorazí i poslední zbývající vlákno. Vrací index příjezdu vlákna (0 je poslední).

### Výjimky při čekání na `await`

Chování `CyclicBarrier` je komplexnější než u `CountDownLatch`, protože chyba v jednom vlákně obvykle ovlivní všechna ostatní:

1\. InterruptedException a BrokenBarrierException

Pokud je jedno z čekajících vláken přerušeno (`interrupt`), nebo pokud vyprší timeout u jedné metody `await()`, bariéra se považuje za poškozenou (`broken`).

* Důsledek: Všechna ostatní vlákna, která na bariéře aktuálně čekají (nebo se k ní pokusí připojit), okamžitě dostanou výjimku `BrokenBarrierException`.
* Význam: Tím se zabrání nekonečnému čekání ostatních vláken v situaci, kdy jedno z nich už nemůže v práci pokračovat.

2\. Výjimky v bariérové akci

Pokud dojde k výjimce v rámci `barrierAction` (akce spuštěná po sestavení všech vláken), bariéra je rovněž označena jako poškozená a všechna čekající vlákna obdrží `BrokenBarrierException`.

3\. Resetování bariéry

Na rozdíl od `CountDownLatch` můžete zavolat metodu `reset()`. Ta bariéru vrátí do výchozího stavu, ale všechna vlákna, která na ní právě čekala, obdrží `BrokenBarrierException`.

## Příklad

V tomto příkladu máme 3 pracovní vlákna (`Worker`), která počítají chyby v různých částech logu. Po dokončení jejich práce bariéra automaticky spustí `Aggregator`, který výsledky sečte.

Java

```java
import java.util.concurrent.CyclicBarrier;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.Map;

public class LogProcessingSystem {

    // Mapa pro ukládání mezivýsledků z jednotlivých vláken
    private static final Map<String, Integer> partialResults = new ConcurrentHashMap<>();

    public static void main(String[] args) {
        int workerCount = 3;

        // Bariéra: Po dokončení všech 3 workerů se spustí agregační úloha
        CyclicBarrier barrier = new CyclicBarrier(workerCount, () -> {
            System.out.println("--- BARIÉRA: Všechna vlákna hotova. Spouštím agregaci výsledků ---");
            int totalErrors = partialResults.values().stream().mapToInt(Integer::intValue).sum();
            System.out.println("CELKOVÝ POČET CHYB V SYSTÉMU: " + totalErrors);
            partialResults.clear(); // Příprava na další cyklus (bariéra je znovupoužitelná)
        });

        ExecutorService executor = Executors.newFixedThreadPool(workerCount);

        // Simulace zpracování různých částí logů (např. Log-A, Log-B, Log-C)
        String[] logParts = {"Server-Log-A", "Server-Log-B", "Server-Log-C"};

        for (String logPart : logParts) {
            executor.execute(new LogWorker(logPart, barrier));
        }

        executor.shutdown();
    }

    static class LogWorker implements Runnable {
        private final String partName;
        private final CyclicBarrier barrier;

        LogWorker(String partName, CyclicBarrier barrier) {
            this.partName = partName;
            this.barrier = barrier;
        }

        @Override
        public void run() {
            try {
                // 1. FÁZE: Samostatná analýza logu
                System.out.println("[" + partName + "] Analyzuji data...");
                int errorsFound = (int) (Math.random() * 50); // Simulace nálezu chyb
                partialResults.put(partName, errorsFound);
                
                Thread.sleep((long) (Math.random() * 2000)); // Simulace I/O operace

                System.out.println("[" + partName + "] Dokončeno. Čekám na ostatní u bariéry.");
                
                // 2. FÁZE: Synchronizační bod
                barrier.await();

                // 3. FÁZE: Pokračování po agregaci (např. uvolnění prostředků)
                System.out.println("[" + partName + "] Pokračuji v čištění dočasných souborů.");

            } catch (Exception e) {
                System.err.println("[" + partName + "] Kritická chyba: " + e.getMessage());
            }
        }
    }
}
```

#### Proč je toto řešení v IT praxi robustní?

1. Atomické zpracování: Agregace se nespustí dříve, dokud není i ten nejpomalejší log analyzován.
2. Oddělení zodpovědnosti: Workeři se starají pouze o svá data, o finální součet se stará `barrierAction`.
3. Detekce chyb: Pokud jeden `LogWorker` havaruje (vyhodí výjimku nebo timeout), bariéra se "rozbije" (`BrokenBarrierException`) a ostatní vlákna nebudou nekonečně čekat na neexistující výsledek.

## Závěr

Porovnání CountDownLatch vs CyclicBarrier:

<table data-header-hidden><thead><tr><th width="169"></th><th width="277"></th><th width="266"></th></tr></thead><tbody><tr><td><strong>Vlastnost</strong></td><td><strong>CountDownLatch</strong></td><td><strong>CyclicBarrier</strong></td></tr><tr><td>Opakovatelnost</td><td>Jednorázový (nelze resetovat).</td><td>Cyklický (lze použít opakovaně).</td></tr><tr><td>Kdo na koho čeká</td><td>Vlákno (main) čeká na události v jiných vláknech.</td><td>Vlákna čekají na sebe navzájem.</td></tr><tr><td>Synchronizační bod</td><td>Pouze <code>countDown()</code> (neblokuje volajícího).</td><td><code>await()</code> (blokuje všechna vlákna).</td></tr><tr><td>Bariérová akce</td><td>Neexistuje.</td><td>Podporuje spuštění kódu po spojení vláken.</td></tr></tbody></table>
