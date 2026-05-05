# Executor - ForkJoinPool

## Motivace

Při vývoji aplikací v jazyce Java se brzy narazí na potřebu spouštět úlohy souběžně. V raných verzích jazyka bylo nutné ručně vytvářet a spravovat instance třídy `Thread`, což při větším rozsahu projektu vedlo k nepřehlednému kódu a neefektivnímu nakládání se systémovými prostředky. Problémem nebylo pouze samotné spuštění vlákna, ale především jeho životní cyklus – tedy jak vlákno bezpečně ukončit, jak znovu využít již vytvořené vlákno pro jinou úlohu nebo jak omezit celkový počet běžících vláken, aby nedošlo k zahlcení operačního systému.

#### Proč byla zavedena abstrakce exekutorů

Koncept exekutorů (Executor Framework) byl zaveden za účelem oddělení správy vláken od samotné logiky úlohy. Namísto toho, aby programátor přímo volal `new Thread().start()`, definuje pouze to, co se má provést (objekt typu `Runnable` nebo `Callable`), a způsob provedení deleguje na specializovanou službu.

* Správa zdrojů: Umožňuje recyklaci vláken (Thread Pooling), čímž se snižuje režie spojená s neustálým vytvářením a ničením objektů vláken.
* Škálovatelnost: Pomocí exekutorů lze snadno definovat limity pro počet souběžně zpracovávaných úloh, což chrání aplikaci před pádem z důvodu nedostatku paměti.
* Oddělení odpovědnosti: Vývojář se soustředí na implementaci byznys logiky, zatímco exekutor řeší plánování, fronty úkolů a synchronizaci.

#### Klíčové komponenty a jejich hierarchie

Framework je postaven na několika rozhraních, která definují úroveň kontroly nad vykonávanými úlohami:

<table data-header-hidden><thead><tr><th width="249"></th><th width="471"></th></tr></thead><tbody><tr><td><strong>Komponenta</strong></td><td><strong>Charakteristika a účel</strong></td></tr><tr><td>Executor</td><td>Nejjednodušší rozhraní s jednou metodou <code>execute()</code>. Slouží k pouhému spuštění úlohy bez možnosti sledovat její výsledek.</td></tr><tr><td>ExecutorService</td><td>Rozšířené rozhraní, které přidává metody pro správu životního cyklu (např. <code>shutdown()</code>) a získávání výsledků pomocí objektů <code>Future</code>.</td></tr><tr><td>ScheduledExecutorService</td><td>Specializovaná varianta umožňující plánování úloh s časovou prodlevou nebo jejich opakované spouštění v intervalech.</td></tr><tr><td>ForkJoinPool</td><td>Specifická implementace navržená pro rekurzivní úlohy a algoritmy typu „rozděl a panuj“, která maximalizuje využití vícejádrových procesorů pomocí techniky work-stealing.</td></tr></tbody></table>

#### Koncept Work-Stealing a ForkJoinPool

Zatímco standardní `ExecutorService` využívá pro úkoly společnou frontu, ze které si vlákna berou práci, `ForkJoinPool` zavádí pokročilejší mechanismus. Každé vlákno má svou vlastní frontu úloh. Pokud některé vlákno dokončí svou práci dříve, může „ukrást“ (steal) úlohu z konce fronty jiného, vytíženějšího vlákna. Tím je zajištěno, že všechna procesorová jádra jsou rovnoměrně vytížena a žádné nezůstává zbytečně nečinné, zatímco ostatní jsou přetížena.

Tento framework tvoří technologický základ, na kterém jsou dnes postaveny i moderní prvky Javy, jako jsou paralelní streamy (Parallel Streams) a částečně i plánování dříve zmíněných virtuálních vláken.

## Základní datové typy

Z hlediska programátora je Executor Framework v Javě strukturován jako hierarchie rozhraní a tříd, které umožňují plynule škálovat od jednoduchého asynchronního spuštění až po komplexní správu fondů vláken. Hlavním cílem tohoto rozdělení je definovat úroveň kontroly, kterou má vývojář nad odeslaným úkolem a stavem systému.

#### Rozhraní Executor

Jedná se o nejvíce bazální rozhraní celého frameworku. Jeho smyslem je poskytnout jednotný způsob pro oddělení definice úkolu od mechanismu jeho vykonání.

* Metoda `execute(Runnable)`: Přijímá příkaz k provedení. Programátor nemá informaci o tom, kdy přesně bude úloha dokončena, ani nemůže získat její návratovou hodnotu. Je to typický model "vystřel a zapomeň" (fire-and-forget).

#### Rozhraní ExecutorService

Toto rozhraní představuje klíčový prvek, se kterým se v praxi pracuje nejčastěji. Rozšiřuje základní `Executor` o funkce pro správu životního cyklu a sledování úloh.

* Odesílání úloh s návratovou hodnotou: Pomocí metody `submit(Callable)` lze odeslat úlohu, která vrací výsledek nebo vyhazuje kontrolovanou výjimku. Návratovým typem je objekt `Future`, skrze který se lze dotázat na stav zpracování.
* Správa ukončení: Obsahuje metody jako `shutdown()` (dokončí běžící úlohy, ale nepřijme nové) a `shutdownNow()` (pokusí se zastavit i běžící úlohy). Bez korektního ukončení `ExecutorService` by aplikace mohla zůstat běžet i po skončení hlavní metody `main`, protože vlákna v poolu jsou standardně typu non-daemon.

#### Objekty Future a Callable

Aby bylo možné v asynchronním prostředí efektivně pracovat s daty, byla zavedena dvojice rozhraní, která doplňují klasické `Runnable`.

* `Callable`: Na rozdíl od `Runnable` vrací výsledek a umožňuje vyhazovat výjimky (checked exceptions).
* `Future`: Slouží jako "poukázka" na výsledek, který bude k dispozici v budoucnosti. Programátor může volat `get()`, což je blokující operace, která počká na dokončení úlohy, nebo `isDone()` pro zjištění stavu bez blokování.

#### Třída Executors (Tovární metody)

V běžné praxi se implementace exekutorů nevytvářejí pomocí konstruktorů, ale využívají se statické tovární metody třídy `Executors`. Tyto metody zapouzdřují konfiguraci nejčastějších typů vláknových fondů (Thread Pools).

<table data-header-hidden><thead><tr><th width="253"></th><th width="474"></th></tr></thead><tbody><tr><td><strong>Metoda</strong></td><td><strong>Popis chování</strong></td></tr><tr><td><code>newFixedThreadPool(n)</code></td><td>Vytvoří fond s pevným počtem vláken. Pokud jsou všechna vytížena, úlohy čekají ve frontě.</td></tr><tr><td><code>newCachedThreadPool()</code></td><td>Vytváří nová vlákna podle potřeby a znovu využívá ta stávající. Ideální pro velké množství krátkodobých úloh.</td></tr><tr><td><code>newSingleThreadExecutor()</code></td><td>Zajišťuje, že všechny úlohy budou vykonávány sekvenčně v jediném vlákně.</td></tr><tr><td><code>newScheduledThreadPool(n)</code></td><td>Umožňuje spouštět úlohy s prodlevou nebo periodicky.</td></tr></tbody></table>

#### ForkJoinPool

Tato implementace se od ostatních liší svým vnitřním uspořádáním. Zatímco běžné exekutory jsou vhodné pro nezávislé I/O úlohy, `ForkJoinPool` je navržen pro výpočetně náročné operace, které lze rozdělit na menší podúlohy (recursive tasks).

Programátor v tomto případě definuje úlohu dědící z `RecursiveTask` (vrací hodnotu) nebo `RecursiveAction` (nevrací hodnotu). Algoritmus uvnitř úlohy typicky kontroluje, zda je úkol dostatečně malý; pokud ne, rozdělí jej (`fork`) na dvě části a následně výsledky spojí (`join`). Tento přístup je základem pro paralelní zpracování kolekcí v Java Streams API.

## CommonPool vs vlastní executor

Koncept Common Pool představuje sdílenou instanci `ForkJoinPool`, která je v prostředí JVM dostupná pro všechny úlohy běžící v rámci jedné aplikace. Tento fond je navržen tak, aby minimalizoval režii spojenou s vytvářením nových vláken pro paralelní operace a automaticky škáloval svou velikost podle počtu dostupných procesorových jader.

Jedná se o statický fond vláken, který je implicitně využíván mnoha moderními funkcemi Javy. Pokud není specifikován vlastní exekutor, JVM automaticky deleguje práci právě sem.

* Paralelní Streamy: Metody jako `parallelStream()` využívají Common Pool k rozdělení datových sad.
* CompletableFuture: Metody typu `runAsync` nebo `supplyAsync` (vysvěleny později) bez parametru exekutoru běží v Common Poolu.
* Velikost fondu: Standardně je velikost nastavena na `n - 1`, kde `n` je počet jader procesoru. Toto nastavení předpokládá, že úlohy jsou výpočetně náročné a nebudou vlákna blokovat čekáním na I/O.

Přestože je Common Pool pohodlný, jeho sdílení napříč celou aplikací (včetně knihoven třetích stran) přináší značná rizika, která mohou vést k degradaci výkonu nebo úplnému zamrznutí aplikace:

**1. Riziko vyhladovění (Starvation)**

Jelikož je fond sdílený, jedna špatně navržená komponenta, která zahltí Common Pool dlouhotrvajícími úlohami, může zablokovat ostatní části systému. Pokud například paralelní stream nad velkou kolekcí obsadí všechna vlákna, přestanou v celé aplikaci fungovat asynchronní úkoly využívající `CompletableFuture`.

**2. Problém s blokujícími operacemi (I/O)**

Common Pool je optimalizován pro CPU-bound úlohy (výpočty). Pokud se v něm spustí úloha, která čeká na odpověď z databáze nebo sítě, vlákno zůstane zablokované. Vzhledem k malé velikosti fondu stačí několik takových volání a celý Common Pool se stane nepoužitelným pro ostatní paralelní výpočty.

**3. Izolace a kontrola**

Vlastní exekutor umožňuje definovat specifické parametry pro konkrétní doménovou oblast:

* Pojmenování vláken: Usnadňuje analýzu logů a ladění (debugging).
* Velikost fronty: Ochrana před zahlcením paměti při nárazovém zvýšení počtu úloh.
* Strategie odmítnutí: Definice toho, co se má stát, pokud je exekutor přetížen (např. vyhození výjimky nebo vykonání úlohy v hlavním vlákně).

#### Srovnání: Common Pool vs. Vlastní Executor

<table data-header-hidden><thead><tr><th width="143"></th><th width="285"></th><th width="294"></th></tr></thead><tbody><tr><td><strong>Vlastnost</strong></td><td><strong>Common Pool</strong></td><td><strong>Vlastní Executor</strong></td></tr><tr><td>Konfigurace</td><td>Globální, těžko ovlivnitelná.</td><td>Plně pod kontrolou vývojáře.</td></tr><tr><td>Izolace</td><td>Žádná (sdíleno celým JVM).</td><td>Vysoká (chyba v jedné části neovlivní jinou).</td></tr><tr><td>Vhodné pro</td><td>Krátké, čistě výpočetní operace.</td><td>I/O operace, kritické byznys procesy.</td></tr><tr><td>Monitorování</td><td>Obtížné rozlišení zdrojů úloh.</td><td>Snadné sledování vytížení konkrétního poolu.</td></tr></tbody></table>

V profesionálním vývoji je považováno za dobrou praxi vytvářet separátní exekutory pro různé typy zátěže. Například jeden exekutor s pevným počtem vláken pro komunikaci s databází, jiný pro generování PDF dokumentů a další (např. využívající virtuální vlákna) pro zpracování příchozích HTTP požadavků. Common Pool by měl zůstat vyhrazen pouze pro krátké a efektivní transformace dat v rámci Java Streams.

## Základní operace nad executory

Při práci s instancí vlastního exekutoru (typicky implementujícího rozhraní `ExecutorService`) se operace dělí na tři základní oblasti: odesílání úloh, sledování jejich stavu a řízení životního cyklu celého fondu vláken. Správné používání těchto operací je klíčové pro stabilitu aplikace a předcházení únikům paměti (memory leaks).

#### 1. Odesílání úloh k provedení

Existují dva základní způsoby, jak exekutoru předat práci, přičemž volba závisí na tom, zda je vyžadován výsledek operace.

* `execute(Runnable)`: Tato metoda pochází ze základního rozhraní `Executor`. Používá se pro asynchronní spuštění úloh typu „vystřel a zapomeň“ (fire & forget). Metoda nevrací žádnou hodnotu a neumožňuje snadno zjistit, zda úloha skončila úspěchem či chybou.
* `submit(Callable)` nebo `submit(Runnable)`: Rozšířená metoda, která vrací objekt typu `Future`. Ten slouží jako odkaz na budoucí výsledek. Pokud je předán `Callable`, může úloha vrátit hodnotu nebo vyhodit kontrolovanou výjimku.

#### 2. Sledování a řízení úloh skrze Future

Jakmile je úloha odeslána pomocí `submit()`, je možné s ní dále pracovat skrze získanou „poukázku“ na výsledek.

* `get()`: Blokující operace, která zastaví aktuální vlákno a čeká, dokud exekutor úlohu nedokončí. Poté vrátí výsledek.
* `isDone()`: Neblokující dotaz, který vrací logickou hodnotu `true`, pokud je úloha hotová, zrušená nebo skončila výjimkou.
* `cancel(boolean)`: Pokus o zrušení úlohy, která ještě nezačala nebo právě probíhá. Parametr určuje, zda má být vlákno přerušeno (interrupt), pokud již na úloze pracuje.

#### 3. Hromadné operace (Bulk Operations)

Vlastní exektutor umožňuje efektivně spravovat celou skupinu úloh najednou, což je užitečné pro paralelní agregaci dat.

* `invokeAll(Collection<Callable>)`: Odešle seznam úloh a čeká, dokud nejsou všechny dokončeny.
* `invokeAny(Collection<Callable>)`: Odešle seznam úloh a čeká pouze na první úspěšně dokončenou. Ostatní jsou automaticky zrušeny.

#### 4. Řízení životního cyklu exekutoru

U vlastních exekutorů je programátor zodpovědný za jejich korektní ukončení. Pokud exekutor není vypnut, vlákna uvnitř něj mohou bránit ukončení celé JVM (pokud nejsou typu daemon):

* `shutdown()`: Zahájí postupné vypínání. Exekutor přestane přijímat nové úlohy, ale dokončí ty, které jsou již ve frontě.
* `shutdownNow()`: Okamžitý pokus o zastavení. Snaží se přerušit běžící úlohy a zahodí ty, které čekají ve frontě.
* `isShutdown()`: Vrací `true`, pokud již byla vyvolána jedna z metod pro ukončení.
* `awaitTermination()`: Blokující volání, které čeká definovanou dobu, než se exekutor plně vypne.

## Vytvoření vlastního executoru

Při vytváření vlastního exekutoru je nezbytné zvolit takovou konfiguraci, která odpovídá povaze vykonávaných úloh. V jazyce Java lze exekutor vytvořit buď pomocí tovární třídy `Executors` pro standardní scénáře, nebo přímou instanciací třídy `ThreadPoolExecutor` pro detailní kontrolu nad parametry fondu.

#### Způsoby definice vlastního exekutoru

Konfigurace vlastního exekutoru umožňuje definovat klíčové aspekty, jako je počet vláken, chování fronty a pojmenování vláken, což je zásadní pro pozdější diagnostiku aplikací.

1. Využití ThreadFactory pro pojmenování: Je dobrou praxí vlákna v poolu pojmenovat, aby bylo v logu jasné, která komponenta úlohu vykonává.
2. Volba typu fondu:
   * FixedThreadPool: Pro úlohy s předvídatelnou zátěží.
   * CachedThreadPool: Pro velké množství krátce trvajících úloh.
   * VirtualThreadPerTaskExecutor: Pro moderní aplikace náročné na I/O operace.

#### Implementace: Vytvoření konfigurovaného exekutoru

Níže je uveden příklad vytvoření exekutoru s vlastní továrnou na vlákna, což zajišťuje, že všechna vlákna budou mít jednotný prefix.

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.ThreadFactory;
import java.util.concurrent.atomic.AtomicInteger;

public class ExecutorFactory {

    public static ExecutorService createCustomPool(int threads, String poolName) {
        // Definice vlastní továrny pro pojmenování vláken
        ThreadFactory factory = new ThreadFactory() {
            private final AtomicInteger count = new AtomicInteger(1);
            @Override
            public Thread newThread(Runnable r) {
                return new Thread(r, poolName + "-thread-" + count.getAndIncrement());
            }
        };

        // Vytvoření fondu s pevným počtem vláken a vlastní továrnou
        return Executors.newFixedThreadPool(threads, factory);
    }
}
```

#### Reálný příklad: Paralelní zpracování faktur

V tomto scénáři aplikace přijme seznam ID faktur, které je nutné vygenerovat a odeslat. Použití vlastního exekutoru zajistí, že generování faktur nezahltí celý systém a v logu bude jasně dohledatelné pod názvem `InvoiceService`.

```java
import java.util.List;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.TimeUnit;

public class InvoiceProcessor {

    // Vytvoření dedikovaného exekutoru pro fakturační modul
    private final ExecutorService invoiceExecutor = 
            ExecutorFactory.createCustomPool(3, "InvoiceService");

    public void processInvoices(List<String> invoiceIds) {
        for (String id : invoiceIds) {
            // Odeslání úlohy k paralelnímu zpracování
            invoiceExecutor.execute(() -> {
                String threadName = Thread.currentThread().getName();
                System.out.println("[" + threadName + "] Zahajuji generování faktury: " + id);
                
                simulateLongOperation(1500); // Simulace tvorby PDF a odeslání
                
                System.out.println("[" + threadName + "] Faktura " + id + " byla odeslána.");
            });
        }
    }

    public void shutdown() {
        invoiceExecutor.shutdown();
        try {
            if (!invoiceExecutor.awaitTermination(10, TimeUnit.SECONDS)) {
                invoiceExecutor.shutdownNow();
            }
        } catch (InterruptedException e) {
            invoiceExecutor.shutdownNow();
        }
    }

    private void simulateLongOperation(int ms) {
        try { Thread.sleep(ms); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
    }

    public static void main(String[] args) {
        InvoiceProcessor processor = new InvoiceProcessor();
        List<String> ids = List.of("FA-2024-001", "FA-2024-002", "FA-2024-003", "FA-2024-004");

        processor.processInvoices(ids);
        processor.shutdown();
    }
}
```

Proč je toto řešení lepší než Common Pool?

<table data-header-hidden><thead><tr><th width="120"></th><th></th><th></th></tr></thead><tbody><tr><td><strong>Faktor</strong></td><td><strong>Vlastní InvoiceService Pool</strong></td><td><strong>Common Pool (sdílený)</strong></td></tr><tr><td>Izolace</td><td>Zpracovává se max. 3 faktury najednou, zbytek čeká.</td><td>Mohlo by se spustit tolik faktur, kolik je jader, a zablokovat celou aplikaci.</td></tr><tr><td>Diagnostika</td><td>V logu vidíme <code>InvoiceService-thread-1</code>.</td><td>V logu vidíme anonymní <code>ForkJoinPool.commonPool-worker-1</code>.</td></tr><tr><td>Stabilita</td><td>Pokud generování zamrzne, jsou blokována jen 3 vlákna.</td><td>Pokud generování zamrzne, aplikace přestane obsluhovat paralelní streamy.</td></tr></tbody></table>

Z uvedeného příkladu vyplývá, že vlastní exekutor slouží jako bezpečnostní mantinel, který izoluje výpočetně nebo časově náročné moduly aplikace od zbytku systému.

#### Reálný příklad: Řízený zápis do souboru

Tento scénář je kritický v aplikacích, kde více asynchronních částí systému (např. různé moduly pro zpracování objednávek, plateb a dopravy) potřebuje zapisovat události do jednoho sdíleného textového souboru. Přímý paralelní zápis by mohl vést k proplétání textu (tzv. race conditions) nebo k chybám uzamčení souboru operačním systémem. Použitím exekutoru s jediným vláknem je zajištěna serializace požadavků, aniž by docházelo k blokování hlavních procesů aplikace.

#### Princip řešení

Všechny moduly aplikace odesílají své textové zprávy do exekutoru, který vnitřně spravuje frontu úloh. Jediné existující vlákno postupně vybírá zprávy z této fronty a provádí fyzický zápis na disk. Tím je zaručeno, že:

* Zápisy probíhají v přesném pořadí, v jakém byly exekutoru předány.
* V jeden okamžik je soubor otevřen pouze jednou instancí zapisovače.
* Zpoždění disku (I/O latence) neovlivňuje rychlost odezvy byznys logiky.

#### Implementace: Bezpečný Logger

V ukázce je demonstrován servisní objekt, který zapouzdřuje `SingleThreadExecutor` pro bezpečný a izolovaný zápis do logu.

```java
import java.io.BufferedWriter;
import java.io.FileWriter;
import java.io.IOException;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

public class AsyncFileLogger {

    // Exekutor s jediným vláknem pro serializaci zápisů
    private final ExecutorService diskWriter = Executors.newSingleThreadExecutor();
    private final String logFileName;

    public AsyncFileLogger(String logFileName) {
        this.logFileName = logFileName;
    }

    public void log(String level, String message) {
        // Úloha je odeslána k asynchronnímu zpracování
        diskWriter.execute(() -> {
            String logEntry = String.format("[%s] %s: %s%n", 
                java.time.LocalDateTime.now(), level, message);
            
            try (BufferedWriter writer = new BufferedWriter(new FileWriter(logFileName, true))) {
                // Simulace zápisu, který může trvat delší dobu
                writer.write(logEntry);
                System.out.println("Zapsáno do souboru: " + message);
            } catch (IOException e) {
                System.err.println("Chyba při zápisu do logu: " + e.getMessage());
            }
        });
    }

    public void close() {
        diskWriter.shutdown();
        try {
            if (!diskWriter.awaitTermination(5, TimeUnit.SECONDS)) {
                diskWriter.shutdownNow();
            }
        } catch (InterruptedException e) {
            diskWriter.shutdownNow();
        }
    }

    public static void main(String[] args) {
        AsyncFileLogger logger = new AsyncFileLogger("system.log");

        // Simulace volání z různých částí aplikace (např. z různých vláken)
        logger.log("INFO", "Uživatel se přihlásil.");
        logger.log("WARNING", "Pokus o přístup k neautorizovanému zdroji.");
        logger.log("ERROR", "Připojení k databázi bylo přerušeno.");

        logger.close();
    }
}
```

Přiřazení jediného vláknového fondu pro zápis do souboru přináší následující vlastnosti:

<table data-header-hidden><thead><tr><th width="165"></th><th></th></tr></thead><tbody><tr><td><strong>Prvek</strong></td><td><strong>Chování v single-threaded modelu</strong></td></tr><tr><td>Konzistence dat</td><td>Nemůže dojít k "přemazání" nebo prolnutí dvou řádků textu.</td></tr><tr><td>Fronta (Backlog)</td><td>Pokud disk nestíhá, zprávy se hromadí v RAM, ale neblokují aplikaci.</td></tr><tr><td>Thread-Safety</td><td>Třída <code>FileWriter</code> nemusí být synchronizována, protože k ní přistupuje vždy jen jedno vlákno.</td></tr><tr><td>Ukončení</td><td>Metoda <code>shutdown()</code> zajistí, že se do souboru dopíší i zprávy, které jsou právě ve frontě.</td></tr></tbody></table>

Tento model je vhodný i pro situace, kdy aplikace komunikuje přes sériový port nebo jiné zařízení, které z povahy věci neumožňuje paralelní přístup (tzv. "Single Resource Access" problém).

## Typické scénáře použití

### Jednorázové vykonání úlohy Fire & Forget

{% hint style="info" %}
**Fire & Forget** je typ úlohy, kdy potřebujeme zahájit vykonání nějaké činnosti, ale nečekáme (nebo vůbec neřešíme) její dokončení. Příkladem je zápis do logu či odeslání nějakého notifikačního e-mailu, kdy chceme, ať se operace provede, ale nechceme před posláním výsledku operace uživateli čekat na její dokončení.
{% endhint %}

Tento scénář je využíván v situacích, kdy je nezbytné provést vedlejší operaci, která není kritická pro okamžitou odpověď uživateli a jejíž výsledek hlavní vlákno nepotřebuje pro svou další činnost. Typicky se jedná o zápis do logů, odesílání notifikací nebo sběr telemetrických dat. Oddělením těchto úloh se výrazně zvyšuje odezva aplikace, protože hlavní exekuční proud není blokován potenciálně pomalým I/O přenosem.

#### Princip fungování

V tomto modelu je využíváno rozhraní `Executor` nebo `ExecutorService` a jeho metoda `execute(Runnable)`. Klíčovým aspektem je, že metoda nevrací žádnou referenci na budoucí výsledek. Jakmile je úloha předána exekutorovi, hlavní vlákno pokračuje v další instrukci. Pro tento účel se často volí `CachedThreadPool` (pokud jsou úlohy krátké a časté) nebo `SingleThreadExecutor` (pokud je vyžadováno zachování pořadí úloh, například u logování).

#### Implementace v Javě

Níže je uveden příklad simulující odeslání e-mailového potvrzení po dokončení nákupu. Zatímco se e-mail odesílá na pozadí, uživateli je okamžitě potvrzena objednávka.

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class EmailNotificationService {

    // Vytvoření exekutora pro zpracování vedlejších úloh
    private static final ExecutorService executor = Executors.newCachedThreadPool();

    public void processOrder() {
        System.out.println("Objednávka byla přijata a zpracována.");

        // Scénář Fire-and-forget: Odeslání e-mailu na pozadí
        executor.execute(() -> {
            try {
                // Simulace síťového zpoždění při odesílání e-mailu
                Thread.sleep(2000);
                System.out.println("E-mailové potvrzení bylo úspěšně odesláno na pozadí.");
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        // Hlavní vlákno nečeká a okamžitě pokračuje
        System.out.println("Uživateli bylo zobrazeno potvrzení na obrazovce.");
    }

    public static void main(String[] args) {
        EmailNotificationService service = new EmailNotificationService();
        service.processOrder();
        
        // V reálné aplikaci by se exekutor zavíral při ukončení aplikace
        // executor.shutdown();
    }
}
```

### Omezení podle systémových zdrojů

Tento scénář se uplatňuje v situacích, kdy je k dispozici omezený systémový zdroj (např. databázové připojení, propustnost disku nebo specifický hardware) a je nepřípustné, aby jej využívalo příliš mnoho vláken současně. Zatímco u virtuálních vláken se souběžnost omezuje pomocí semaforů, u platformových vláken se k tomuto účelu využívá Fixed Thread Pool. Tento fond udržuje konstantní počet aktivních vláken; pokud je odesláno více úloh, jsou tyto úkoly řazeny do interní fronty (typicky `LinkedBlockingQueue`) a čekají, dokud se některé z vláken neuvolní.

#### Výhody pevného fondu vláken

* Předvídatelnost: Spotřeba paměti pro zásobníky vláken je fixní a nedochází k nekontrolovanému nárůstu systémových prostředků.
* Ochrana zdroje: Cílový systém (např. souborový systém) není zahlcen nadměrným počtem požadavků, což zabraňuje degradaci výkonu v důsledku přepínání kontextu nebo fragmentace zápisu.
* Efektivita: Odpadá režie spojená s neustálým vytvářením a ničením vláken pro každý jednotlivý úkol.

#### Implementace v Javě

V následujícím příkladu je simulován systém pro zápis protokolů do sdíleného souboru. Přestože do systému přichází desítky požadavků na zápis, exekutor zajišťuje, že v jeden okamžik provádějí zápis maximálně dvě vlákna, čímž se předchází nadměrnému zamykání souborového systému.

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

public class ResourceThrottlingService {

    // Vytvoření fondu s pevným počtem 2 vláken
    private static final ExecutorService writerExecutor = Executors.newFixedThreadPool(2);

    public void logData(String message) {
        // Úloha je odeslána bez použití Future (Runnable)
        writerExecutor.execute(() -> {
            String threadName = Thread.currentThread().getName();
            System.out.println("[" + threadName + "] Zahajuji zápis: " + message);
            
            try {
                // Simulace náročného I/O zápisu na disk
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }

            System.out.println("[" + threadName + "] Zápis dokončen.");
        });
    }

    public void shutdownService() {
        writerExecutor.shutdown();
        try {
            if (!writerExecutor.awaitTermination(60, TimeUnit.SECONDS)) {
                writerExecutor.shutdownNow();
            }
        } catch (InterruptedException e) {
            writerExecutor.shutdownNow();
        }
    }

    public static void main(String[] args) {
        ResourceThrottlingService service = new ResourceThrottlingService();

        // Simulace náhlého příchodu 6 požadavků na zápis
        for (int i = 1; i <= 6; i++) {
            service.logData("Záznam události č. " + i);
        }

        service.shutdownService();
    }
}
```

Při použití `newFixedThreadPool` je důležité rozumět mechanismu, jakým jsou úlohy odbavovány, pokud počet úloh překročí počet vláken:

<table data-header-hidden><thead><tr><th width="230"></th><th width="478"></th></tr></thead><tbody><tr><td><strong>Stav systému</strong></td><td><strong>Chování exekutoru</strong></td></tr><tr><td>Počet úloh &#x3C; počet vláken</td><td>Každá úloha dostane okamžitě přiděleno volné vlákno.</td></tr><tr><td>Počet úloh = počet vláken</td><td>Všechna vlákna jsou vytížena, systém pracuje na maximální definované kapacitě.</td></tr><tr><td>Počet úloh > počet vláken</td><td>Nadbytečné úlohy jsou uloženy do fronty a čekají na uvolnění prvního vlákna.</td></tr><tr><td>Vlákno havaruje</td><td>Pokud vlákno skončí chybou, exekutor automaticky vytvoří nové, aby udržel cílový počet.</td></tr></tbody></table>

Tento model je ideální pro stabilní serverové aplikace, kde je kladen důraz na robustnost a ochranu před přetížením na úkor potenciálního zpoždění (latence) úloh čekajících ve frontě.

### Pravidelně se opakující úlohy

Tento scénář řeší potřebu spouštění úloh, které nejsou iniciovány přímou akcí uživatele, ale jsou vázány na časový harmonogram. V prostředí Javy se k tomuto účelu využívá specializované rozhraní `ScheduledExecutorService`. To umožňuje překonat omezení jednoduchých smyček typu `while(true) { Thread.sleep() }`, které jsou náchylné k chybám, špatně se ukončují a blokují vlákno i v době nečinnosti.

#### Klíčové koncepty plánování

Při plánování úloh se rozlišují tři základní režimy vykonávání:

* Zpožděné spuštění (Schedule): Úloha je vykonána pouze jednou po uplynutí definovaného časového intervalu.
* Fixní sazba (Fixed Rate): Úloha je spouštěna v pravidelných intervalech bez ohledu na to, jak dlouho trvalo předchozí vykonání. Pokud je interval 10 minut a úloha trvá 2 minuty, další spuštění nastane přesně 10 minut po začátku té předchozí.
* Fixní prodleva (Fixed Delay): Interval se začíná odpočítávat až po dokončení předchozí úlohy. Pokud je prodleva 10 minut a úloha trvá 2 minuty, další spuštění nastane za 12 minut od začátku té předchozí. Tento režim je bezpečnější, pokud hrozí, že by se úlohy mohly překrývat.

#### Implementace v Javě

V následujícím příkladu je demonstrován servis pro automatické čištění mezipaměti (cache), který se spouští se zpožděním a následně se opakuje v pravidelných intervalech.

```java
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;

public class CacheCleanupService {

    // Vytvoření plánovaného exekutoru s jedním vláknem
    private final ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);

    public void startMaintenance() {
        final Runnable cleanupTask = () -> {
            System.out.println("Provádím údržbu: Čištění mezipaměti a dočasných souborů...");
            // Logika čištění
        };

        // Naplánování úlohy:
        // - počáteční zpoždění: 5 sekund
        // - opakování každých 10 sekund (Fixed Delay pro zajištění rozestupu)
        scheduler.scheduleWithFixedDelay(cleanupTask, 5, 10, TimeUnit.SECONDS);
        
        System.out.println("Plánovač údržby byl spuštěn.");
    }

    public void stopMaintenance() {
        System.out.println("Zastavuji plánovač...");
        scheduler.shutdown();
    }

    public static void main(String[] args) throws InterruptedException {
        CacheCleanupService service = new CacheCleanupService();
        service.startMaintenance();

        // Necháme simulaci běžet 35 sekund
        Thread.sleep(35000);
        service.stopMaintenance();
    }
}
```

Při použití tohoto exekutoru je důležité zajistit, aby vnitřní logika úloh byla ošetřena bloky `try-catch`. Pokud by totiž uvnitř plánované úlohy došlo k nevyrovnané výjimce (Runtime Exception), vykonávající vlákno by ukončilo svou činnost a plánovač by přestat další běhy úlohy vykonávat bez jakéhokoliv upozornění.

### Rekurzivní dělení výpočetně náročné úlohy

Tento scénář se zaměřuje na efektivní využití výpočetního výkonu moderních vícejádrových procesorů při řešení úloh, které lze dekomponovat na menší, samostatně řešitelné podúlohy. Na rozdíl od běžných exekutorů, které jsou optimalizovány pro nezávislé úkoly, je Fork/Join framework (představený v Javě 7) navržen pro algoritmy typu „rozděl a panuj“ (divide and conquer).

Základem je specifická implementace exekutoru nazvaná ForkJoinPool, která využívá mechanismus work-stealing. Tento princip zajišťuje, že pokud jedno vlákno dokončí své úkoly dříve, dokáže „ukrást“ práci z fronty jiného, stále vytíženého vlákna, čímž se minimalizuje nečinnost procesorových jader.

#### Mechanismus Fork a Join

Proces zpracování probíhá ve třech fázích, které jsou rekurzivně opakovány:

1. Analýza prahu (Threshold): Úloha prověří, zda je dostatečně malá na to, aby byla vykonána sekvenčně v aktuálním vlákně.
2. Fork (Rozvětvení): Pokud je úloha příliš velká, je rozdělena na dvě nebo více podúloh, které jsou asynchronně odeslány do fronty k vykonání.
3. Join (Spojení): Vlákno počká na dokončení podúloh a následně agreguje (spojí) jejich dílčí výsledky do finální odpovědi.

#### Implementace v Javě

V následujícím příkladu je demonstrován výpočet součtu prvků v rozsáhlém poli. Úloha dědí z třídy `RecursiveTask<Long>`, což naznačuje, že výpočet vrací hodnotu typu `Long`.

```java
import java.util.concurrent.RecursiveTask;
import java.util.concurrent.ForkJoinPool;

public class ArraySumTask extends RecursiveTask<Long> {
    private static final int THRESHOLD = 10_000; // Mez pro sekvenční zpracování
    private final int[] array;
    private final int start;
    private final int end;

    public ArraySumTask(int[] array, int start, int end) {
        this.array = array;
        this.start = start;
        this.end = end;
    }

    @Override
    protected Long compute() {
        // 1. Fáze: Kontrola prahu
        if ((end - start) <= THRESHOLD) {
            long sum = 0;
            for (int i = start; i < end; i++) {
                sum += array[i];
            }
            return sum;
        } else {
            // 2. Fáze: Rozdělení úlohy (Fork)
            int middle = start + (end - start) / 2;
            ArraySumTask leftSubtask = new ArraySumTask(array, start, middle);
            ArraySumTask rightSubtask = new ArraySumTask(array, middle, end);

            // Spuštění první podúlohy asynchronně
            leftSubtask.fork();

            // Druhá podúloha se spustí v aktuálním vlákně (optimalizace)
            long rightResult = rightSubtask.compute();

            // 3. Fáze: Spojení výsledků (Join)
            long leftResult = leftSubtask.join();

            return leftResult + rightResult;
        }
    }

    public static void main(String[] args) {
        int[] data = new int[100_000];
        // Inicializace pole dat...

        // Použití v try-with-resources pro automatické zavření poolu
        try (ForkJoinPool pool = new ForkJoinPool()) {
            Long finalSum = pool.invoke(new ArraySumTask(data, 0, data.length));
            System.out.println("Výsledný součet: " + finalSum);
        }
    }
}
```

{% hint style="info" %}
&#x20;Tento model je klíčový pro fungování paralelních proudů (Parallel Streams) v Javě, kde je `ForkJoinPool.commonPool()` využíván na pozadí pro automatickou paralelizaci operací nad kolekcemi.
{% endhint %}

##

### 🧠 Co to znamená

* sdílený pool pro celou JVM
* omezený počet vláken (≈ počet CPU jader)
* není izolovaný pro tvůj app design

***

## 💥 3. Problém common poolu

👉 typické problémy:

* blokující IO ho zabije
* sdílený s jinými knihovnami
* těžko se debuguje
* neřídíš ho ty

***

## 🧠 4. Kdy použít vlastní ExecutorService

Použij když:

* UI thread (Swing/JavaFX/WPF-like simulace)
* IO operace
* separace logiky (např. network vs compute)
* chceš determinismus

***

## ⚡ 5. ForkJoinPool vs ExecutorService (mentálně)

| vlastnost            | ForkJoinPool | ExecutorService |
| -------------------- | ------------ | --------------- |
| optimalizace pro CPU | ✔            | ❌               |
| blocking IO friendly | ❌            | ✔               |
| sdílený JVM          | ✔            | ❌               |
| izolace              | ❌            | ✔               |
| work-stealing        | ✔            | ❌               |
