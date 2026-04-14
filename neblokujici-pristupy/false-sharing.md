# False Sharing

## Princip False Sharing

False Sharing (falešné sdílení) je jeden z nejzákeřnějších výkonnostních problémů v moderních vícevláknových systémech. Je to v podstatě "neviditelný zabiják" škálovatelnosti, protože k němu dochází na úrovni hardwaru, konkrétně v L1/L2 cache procesoru.

Abychom to pochopili, musíme se podívat, jak procesor čte data z paměti RAM.

#### 1. Cache Lines: Jednotka, ve které procesor myslí

Procesor nikdy nenačítá z paměti RAM jen jeden bajt nebo jednu proměnnou. Vždy načte celý blok dat zvaný Cache Line.

* Ve většině moderních architektur (x86\_64, ARM) má jedna Cache Line velikost 64 bajtů.
* Pokud máš v paměti dvě proměnné typu `long` (každá 8 bajtů) hned vedle sebe, s vysokou pravděpodobností skončí ve stejné Cache Line.

#### 2. Protokol koherence (MESI)

Procesory spolu musí neustále mluvit, aby zajistily, že všechna jádra vidí stejná data. Pokud jedno jádro (Jádro 1) změní hodnotu v Cache Line, musí o tom informovat ostatní jádra (Jádro 2).

Jádro 2 musí svou kopii téže Cache Line zneplatnit (invalidate) a načíst si ji znovu z L3 cache nebo RAM.

#### 3. Jádro problému (Ten "False" moment)

Mějme dvě vlákna:

1. Vlákno A neustále inkrementuje `counter1`.
2. Vlákno B neustále inkrementuje `counter2`.

Logicky jsou tyhle dvě proměnné na sobě nezávislé. Žádný zámek (`synchronized`) tam není - není důvod; vlákna si do proměnných navzájem nepíší. Ale protože v paměti leží vedle sebe, jsou ve stejné Cache Line.

Když Vlákno A změní `counter1`, procesor "označí" celou 64bajtovou linku jako změněnou. Vlákno B, které běží na jiném jádře a chce změnit svůj `counter2`, najednou zjistí, že jeho linka je neplatná. Musí čekat na synchronizaci, i když s `counter1` nemá nic společného.

Výsledek: Vlákna se neustále přetlačují o "vlastnictví" téže cache linky. Výkon pak může být paradoxně horší, než kdyby celý program běžel jen na jednom vlákně.

## Mitigrace False Sharing

Dříve se to řešilo šílenými hacky, kdy programátoři mezi dvě důležité proměnné vkládali 7 nevyužitých `long` proměnných (8 \* 7 = 56 bajtů), aby je od sebe "odsunuli".

Od Javy 8 (nyní však pouze pro interní potřeby JDK) existuje řešení anotací `@Contended`, kdy JVM automaticky přidá kolem daného pole dostatečný padding (vycpávku), aby se zaručilo, že bude v samostatné Cache Line. Anotace je však dnes pouze po vnitřní potřebu JVM a navrhují se jiné způsoby.

```
public class OptimisedCounters {
    @jdk.internal.vm.annotation.Contended
    public long counter1;

    @jdk.internal.vm.annotation.Contended
    public long counter2;
}
```

_(Poznámka: Pro použití v běžném kódu je často potřeba spustit JVM s parametrem `-XX:-RestrictContended`)._

Moderní Java proto nabízí robustnější a čistší abstrakce pro řešení vysokého souběhu na společných čítačích, které jsou postaveny na principu rozkladu (striping). Namísto snahy o fyzické oddělení dvou nesouvisejících proměnných se tyto třídy zaměřují na rozklad jedné logické proměnné do více vnitřních buněk, aby se zabránilo kolizím procesorových cache při zápisu z mnoha vláken.

Základem této rodiny tříd je abstraktní třída `Striped64`. Ta implementuje vnitřní logiku pole "buněk" (Cells), kde každá buňka je v podstatě samostatný `long` s vynuceným paddingem, který zabraňuje tomu, aby se dvě sousední buňky ocitly ve stejné cache line. Když více vláken přistupuje k objektům jako `LongAdder`, knihovna se pokusí přiřadit každému vláknu jinou buňku na základě hash kódu daného vlákna. Tím se drasticky snižuje frekvence instrukcí CAS (Compare-And-Swap) nad stejnou paměťovou adresou.

#### LongAdder v praxi

`LongAdder` je nejběžnějším zástupcem tohoto přístupu. Je ideální pro scénáře, kde se hodnota pouze přičítá (např. globální čítač HTTP požadavků nebo statistik). Vnitřně udržuje základní hodnotu a pole buněk. Pokud dochází ke kolizi na základní hodnotě, vlákna začnou zapisovat do buněk. Celkový součet se vypočítá až v okamžiku volání metody `sum()`, která projde základní hodnotu i všechny buňky a sečte je.

{% code lineNumbers="true" %}
```java
import java.util.concurrent.atomic.LongAdder;

public class MetricsCollector {
    // LongAdder vnitřně řeší padding a rozptyl zápisů mezi jádra CPU
    private final LongAdder requestCounter = new LongAdder();

    public void onRequest() {
        // Operace add/increment je neblokující a minimalizuje False Sharing
        requestCounter.increment();
    }

    public long getTotalRequests() {
        // Sumace proběhne až na vyžádání; v ten moment je čtení pomalejší, 
        // ale zápisy jsou extrémně rychlé
        return requestCounter.sum();
    }
}
```
{% endcode %}

#### LongAccumulator pro obecné operace

Pokud nestačí prosté přičítání, ale je vyžadována obecná funkce (např. hledání maxima nebo násobení), nastupuje `LongAccumulator`. Ten funguje na stejném principu jako `LongAdder`, ale umožňuje definovat vlastní binární operaci v podobě lambda výrazu. To je užitečné pro udržování extrémně výkonných statistik bez nutnosti explicitního zamykání nebo řešení paddingu uživatelem.

{% code lineNumbers="true" %}
```java
import java.util.concurrent.atomic.LongAccumulator;

public class HighScoreTracker {
    // Akumulátor používá Striped64 pro uložení maximální nalezené hodnoty
    private final LongAccumulator maxScore = new LongAccumulator(Long::max, 0);

    public void updateScore(long currentScore) {
        // Vlákna aktualizují maxima ve svých buňkách nezávisle na sobě
        maxScore.accumulate(currentScore);
    }

    public long getMax() {
        // Vrátí globální maximum ze všech vnitřních buněk
        return maxScore.get();
    }
}
```
{% endcode %}

Zatímco `AtomicLong` je v podstatě jediná proměnná, u které se vlákna neustále perou o vlastnictví cache line, `LongAdder` a `LongAccumulator` využívají dynamicky rozšiřované pole, které se přizpůsobuje počtu jader a intenzitě souběhu. Tento přístup efektivně obchází nutnost používat nízkoúrovňové techniky jako ruční padding nebo `@Contended`, protože řešení problému False Sharing je v těchto třídách zabudováno přímo na úrovni implementace JDK. Za vyšší propustnost při zápisu se platí pouze mírně vyšší paměťovou náročností a o něco pomalejším čtením finální sumární hodnoty.

## Jednoduchý demonstrační benchmark

Vytvoříme dvě vlákna. Každé bude inkrementovat svou proměnnou `long`.

Dále vytvoříme dva typy objektů:

1. Rozložený (`Padded`): Kde mezi proměnné vložíme "vatu" z dalších `long` proměnných.
2. Na těsno (`Unpadded`): Kde proměnné leží v paměti hned u sebe.

{% code lineNumbers="true" %}
```java
public class ObjectFalseSharingBenchmark {
    private static final long ITERATIONS = 500_000_000L;

    // Objekt, kde jsou proměnné hned vedle sebe
    static class UnpaddedData {
        public volatile long counterA;
        public volatile long counterB;
    }

    // Objekt s "vatou" (paddingem)
    static class PaddedData {
        public volatile long counterA;
        // 64 bajtů / 8 (velikost longu) = 8 longů. 
        // Vložíme 7 longů jako vycpávku, aby counterB byl v další Cache Line.
        public long p1, p2, p3, p4, p5, p6, p7; 
        public volatile long counterB;
    }

    public static void main(String[] args) throws InterruptedException {
        UnpaddedData unpadded = new UnpaddedData();
        PaddedData padded = new PaddedData();

        System.out.println("Benchmark spuštěn...");

        // 1. TEST: False Sharing
        long start = System.currentTimeMillis();
        runTest(() -> unpadded.counterA++, () -> unpadded.counterB++);
        System.out.println("Bez paddingu (False Sharing): " + (System.currentTimeMillis() - start) + " ms");

        // 2. TEST: S Paddingem
        start = System.currentTimeMillis();
        runTest(() -> padded.counterA++, () -> padded.counterB++);
        System.out.println("S paddingem (Clean): " + (System.currentTimeMillis() - start) + " ms");
    }

    private static void runTest(Runnable task1, Runnable task2) throws InterruptedException {
        Thread t1 = new Thread(() -> {
            for (long i = 0; i < ITERATIONS; i++) task1.run();
        });
        Thread t2 = new Thread(() -> {
            for (long i = 0; i < ITERATIONS; i++) task2.run();
        });
        t1.start(); t2.start();
        t1.join(); t2.join();
    }
}
```
{% endcode %}

Na většině moderních vícejádrových procesorů bude propastný rozdíl:

* S False Sharingem: Může to trvat např. 2500 ms.
* Bez False Sharingu: Může to trvat např. 400 ms.

#### Co se děje v paměti?

V případě `UnpaddedData` vypadá rozložení v paměti (zjednodušeně) takto: `[Hlavička Objektu (12b)][counterA (8b)][counterB (8b)]` Vše se vejde do prvních 28 bajtů. Celé se to tedy načte do jedné 64bajtové linky.

V případě `PaddedData`: `[Hlavička (12b)][counterA (8b)][p1..p7 (56b)][counterB (8b)]` Tady `counterB` začíná až hluboko za hranicí 64 bajtů od začátku `counterA`. Procesor je nucen je načíst do dvou různých Cache Lines.

Procesory dvou jader se přetahují o Cache Line. Pokaždé, když Jádro 1 inkrementuje proměnnou, pošle po sběrnici signál Jádru 2: _"Tvoje cache je neplatná, zahoď ji!"_. Jádro 2 musí počkat, načíst si data znova, jen aby o nanosekundu později udělalo to samé Jádru 1.

Většinu času tak procesor netráví počítáním, ale čekáním na synchronizaci cache (tzv. Cache Coherency Protocol).

