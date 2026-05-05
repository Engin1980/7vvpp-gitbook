# Virtuální vlákna

## Motivace

Při návrhu vysoce škálovatelných aplikací v jazyce Java se po dlouhou dobu naráželo na omezení spojená s modelem, kde jedno vlákno aplikace přímo odpovídá jednomu vláknu operačního systému (tzv. Platform Threads). Vzhledem k tomu, že vlákna operačního systému jsou nákladná na paměť i na režii spojenou s přepínáním kontextu, bývá jejich počet v rámci běžného stroje limitován na řády tisíců. V moderních architekturách, jako jsou mikroslužby, je však běžné zpracovávat desítky tisíc souběžných požadavků, což při modelu „jedno vlákno na jeden požadavek“ vede k rychlému vyčerpání systémových prostředků.

#### Omezení platformových vláken

Hlavním problémem platformových vláken je jejich neefektivita v situacích, kdy dochází k blokujícím I/O operacím (např. čekání na odpověď z databáze nebo HTTP volání). Během tohoto čekání zůstává drahé vlákno operačního systému nevyužito (v čekajícím stavu na výsledek), ale přesto blokuje alokovanou paměť stacku.

#### Navržené řešení

Mechanismus virtuálních vláken je založen na konceptu kooperativního plánování úloh v rámci uživatelského prostoru (JVM), čímž je eliminována závislost na drahém přepínání kontextu v jádře operačního systému. Celé řešení se opírá o vztah mezi virtuálními vlákny, nosnými vlákny a plánovačem.

Základem navrženého řešení je hierarchie, ve které jsou **virtuální vlákna (Virtual Threads) mapována na malý počet platformových vláken**, jež jsou v tomto kontextu označována jako nosná vlákna **(Carrier Threads)**.

* Plánovač (Scheduler): JVM využívá k distribuci virtuálních vláken na nosná vlákna standardní mechanismy plánování (viz `ForkJoinPool` představený později). Počet nosných vláken obvykle odpovídá počtu dostupných procesorových jader, což zajišťuje maximální efektivitu využití CPU.
* Parkování a odpoutání (Mount/Unmount): Pokud virtuální vlákno začne vykonávat kód, je „připoutáno“ (mounted) k nosnému vláknu. Ve chvíli, kdy narazí na blokující operaci (např. `Thread.sleep()`, čtení ze soketu nebo čekání na zámek), je stav jeho zásobníku (stack) uložen z nosného vlákna zpět do haldy (heap). Tímto procesem se virtuální vlákno „odpoutá“ (unmount) a nosné vlákno je okamžitě volné pro obsluhu jiného virtuálního vlákna.
* Obnova vykonávání: Jakmile je blokující operace dokončena (např. data ze sítě dorazila), plánovač označí virtuální vlákno jako připravené a při nejbližší příležitosti jej znovu „připoutá“ k libovolnému volnému nosnému vláknu, kde pokračuje v práci přesně tam, kde skončilo.

## Virtuální vlákna

Virtuální vlákna byla představena v rámci projektu Loom (plně dostupná od Java 21) jako řešení, které odděluje programové vlákno od vlákna operačního systému. Jsou spravována přímo prostředím JVM, což umožňuje vytvářet miliony instancí bez rizika pádu systému.

* Efektivní blokování: Pokud virtuální vlákno narazí na blokující operaci, je „odsunuto“ (unmounted) z nosného vlákna (carrier thread) a toto nosné vlákno může okamžitě obsluhovat jinou úlohu. Jakmile je I/O operace dokončena, virtuální vlákno je znovu naplánováno k vykonání.
* Zachování jednoduchosti: Na rozdíl od reaktivního programování umožňují virtuální vlákna psát kód synchronním, sekvenčním stylem, který se snadno čte a ladí, přičemž pod kapotou je dosaženo vysoké míry asynchronicity.
* Kompatibilita: Jsou navržena tak, aby byla plně kompatibilní s existujícím rozhraním `java.lang.Thread`, což umožňuje jejich snadné nasazení do stávajících ekosystémů bez nutnosti kompletního přepracování aplikací.

Důvodem jejich vzniku byla tedy snaha o dosažení maximální propustnosti serverových aplikací při zachování srozumitelného programového modelu, který není omezen fyzickými limity operačního systému. Virtuální vlákna jsou ideální pro úlohy náročné na I/O, zatímco pro výpočetně náročné operace (CPU-bound) zůstávají platformová vlákna i nadále relevantní.

{% hint style="info" %}
S ohledem na zavedený pojem _virtuální vlákno_ se klasická vlákna nyní budeme odkazovat jako na _platformova vlákna - platform threads_.
{% endhint %}

#### Správa paměti a stacku

Tradiční vlákna mají fixní velikost zásobníku alokovanou operačním systémem (často 1 MB), což je pro miliony vláken neudržitelné. Virtuální vlákna tento problém řeší dynamicky:

* zásobník je u místěn na haldě (heap) jako běžný objekt,
* stack roste a zmenšuje se podle skutečné potřeby aplikace,
* kopírování stacku mezi haldou a nosným vláknem je optimalizováno na úrovni JVM.

#### Omezení a specifické stavy

I přes vysokou efektivitu existují situace, kdy k uvolnění nosného vlákna dojít nemůže. Tento stav se označuje jako přišpendlení (Pinning). K tomuto jevu dochází ve dvou hlavních případech:

1. Nativní kód: Pokud vlákno vykonává kód uvnitř bloku `native` nebo volá funkce přes JNI.
2. Synchronizované bloky: Pokud je blokující operace vykonávána uvnitř bloku nebo metody s klíčovým slovem `synchronized`. V takovém případě je doporučováno nahradit tyto bloky konstrukcí `ReentrantLock`, která "přišpendlení" nezpůsobuje a umožňuje plnou využitelnost virtuálních vláken.

{% hint style="warning" %}
Pozor - zejména druhá odrážka je zásadním omezením při práci s virtuálními vlákny. Práce se `synchronized` je běžná a často používaná technika synchronizace v Javě, která však při propojení s virtuálními vlákny může způsobit zásadní výkonnostní propad.
{% endhint %}

## Virtuální vs Platformová vlákna

Při vývoji aplikací v jazyce Java je klíčové porozumět tomu, že virtuální vlákna nebyla navržena jako úplná náhrada za platformová vlákna, ale jako jejich vysoce efektivní alternativa pro specifické scénáře. Přestože obě entity sdílejí společné API, jejich chování pod zátěží a způsob správy prostředků vykazují značné rozdíly.

#### Společné rysy (Podobnosti)

Z pohledu programátora je zachována kontinuita, což usnadňuje adopci nové technologie bez nutnosti učit se zcela nové programovací paradigma.

* Identické API: Virtuální vlákno je stále instancí třídy `java.lang.Thread`. Metody jako `start()`, `join()`, `isAlive()` nebo `interrupt()` fungují u obou typů identicky.
* ThreadLocal úložiště: Oba typy vláken podporují mechanismus `ThreadLocal` pro uchovávání dat specifických pro dané vlákno, i když u virtuálních vláken je doporučeno k tomuto přistupovat s opatrností kvůli jejich obrovskému počtu.
* Zpracování výjimek: Šíření výjimek a práce s objektem `StackTraceElement` zůstává zachována, což umožňuje standardní ladění a logování chyb.

#### Klíčové odlišnosti

Hlavní rozdíly spočívají v životním cyklu vlákna a v tom, jakým způsobem jsou alokovány systémové prostředky. Zatímco platformová vlákna jsou považována za "těžké" objekty, virtuální vlákna jsou extrémně "lehká".

<table data-header-hidden><thead><tr><th width="125"></th><th width="291"></th><th></th></tr></thead><tbody><tr><td><strong>Parametr</strong></td><td><strong>Platform Threads</strong></td><td><strong>Virtual Threads</strong></td></tr><tr><td>Vytváření (Creation)</td><td>Nákladná operace vyžadující alokaci v OS. Často se využívají pooly (Thread Pools).</td><td>Velmi levná operace. Doporučuje se vytvářet nové vlákno pro každý úkol; poolování je kontraproduktivní.</td></tr><tr><td>Priorita (Priority)</td><td>Podporována a vynucována plánovačem operačního systému.</td><td>Ignorována. Virtuální vlákna běží vždy se střední prioritou (NORM_PRIORITY).</td></tr><tr><td>Démonství (Daemon status)</td><td>Mohou být daemon i non-daemon.</td><td>Jsou vždy a pouze daemon vlákny. Nelze vytvořit non-daemon virtuální vlákno.</td></tr><tr><td>Náklady na blokování</td><td>Vysoké. Blokované vlákno drží 1 MB RAM a prostředky jádra OS.</td><td>Minimální. Blokované vlákno je pouze datovou strukturou v haldě (heap).</td></tr></tbody></table>

#### Rozdíly v praktickém nasazení

Při práci s těmito entitami je nutné změnit některé zažité postupy, které byly u platformových vláken považovány za nejlepší praxi (best practices).

* Strategie přidělování úkolů: U platformových vláken je standardem limitovat jejich počet pomocí `ExecutorService` s pevnou velikostí, aby nedošlo k vyčerpání paměti. U virtuálních vláken je strategií vytvářet nové vlákno pro každou logickou jednotku práce (např. jeden HTTP požadavek = jedno nové virtuální vlákno).
* Vliv na propustnost vs. latenci: Platformová vlákna jsou vhodnější pro CPU-bound operace (matematické výpočty, kódování videa), kde je cílem nízká latence jednoho výpočtu. Virtuální vlákna jsou navržena pro optimalizaci propustnosti (throughput) systému u I/O-bound operací, kde systém musí zvládnout obrovské množství souběžných čekajících úloh.
* Omezení souběžnosti: Zatímco u platformových vláken se souběžnost omezuje velikostí poolu, u virtuálních vláken se k omezení přístupu k vzácným zdrojům (např. počet spojení do databáze) používají synchronizační primitiva jako `Semaphore`.

Z výše uvedeného vyplývá, že ačkoliv se navenek oba typy vláken tváří stejně, vnitřní mechanismy vyžadují odlišný přístup k návrhu architektury aplikace, zejména v oblasti správy životního cyklu a omezování zdrojů.

## Vytváření virtuálních vláken

Při vytváření virtuálních vláken v jazyce Java je kladen důraz na jednoduchost a čitelnost kódu. Na rozdíl od platformových vláken, kde je vyžadována komplexní správa pomocí omezujících poolů, jsou virtuální vlákna navržena pro styl „vytvoř a zapomeň“. Níže jsou uvedeny konkrétní programové přístupy doplněné o ukázky zdrojového kódu.

Při přechodu na virtuální vlákna je kladen důraz na odklon od tradičního způsobu vytváření vláken pomocí konstruktorů nebo děděním od třídy `Thread`. Virtuální vlákna se chovají odlišně z několika důvodů:

* Typová neviditelnost: Virtuální vlákna jsou interní implementací a navenek se stále jeví jako instance třídy `Thread`. Rozlišení mezi platformovým a virtuálním vláknem není prováděno skrze odlišnou třídu, ale skrze tovární metodu, která vlákno správně nakonfiguruje.
* Abstrakce správy: Použitím metod jako `Thread.ofVirtual()` je programátorovi umožněno pracovat s buildery, které lépe spravují vnitřní stav vlákna v haldě (heap), což je s klasickým konstruktorem nerealizovatelné.

Pro spuštění virtuálního vlákna je tedy třeba mít bezparametrickou metodu vracející `void`, která vykonává činnost přiřazenou danému vláknu.

Následuje přehled způsobů vytvoření a spuštění práce ve virtuálním vláknu.

#### 1. Přímé spuštění pomocí statické metody

Nejrychlejší způsob, jak delegovat úlohu do virtuálního vlákna, představuje metoda `startVirtualThread`. Tato metoda nevyžaduje žádnou složitou konfiguraci a je ideální pro jednoduché asynchronní úkoly.

```java
// Okamžité spuštění jednoduché úlohy
Thread vThread = Thread.startVirtualThread(() -> {
    System.out.println("Úloha vykonávána ve virtuálním vlákně: " + Thread.currentThread());
});

// Čekání na dokončení (volitelné)
vThread.join();
```

#### 2. Vytváření pomocí Builderu (Fluent API)

Pro případy, kdy je nutné vláknu přiřadit název nebo specifické vlastnosti pro účely monitoringu a ladění, je využíván builder `Thread.ofVirtual()`.

```java
// Konfigurace pojmenovaného virtuálního vlákna
Thread.Builder builder = Thread.ofVirtual().name("vypocet-", 1);

Thread t1 = builder.start(() -> {
    System.out.println("Běží vlákno: " + Thread.currentThread().getName());
});

// Vytvoření vlákna bez okamžitého spuštění
Thread t2 = builder.unstarted(() -> {
    System.out.println("Dodatečně spuštěné vlákno.");
});
t2.start();
```

#### 3. Využití ExecutorService pro hromadné úlohy

V moderním vývoji je preferován přístup využívající `ExecutorService`. S příchodem virtuálních vláken se používá metoda `newVirtualThreadPerTaskExecutor`, která pro každou úlohu vygeneruje nové vlákno. Doporučuje se použití v bloku try-with-resources, který zajistí automatické uzavření a synchronizaci.

```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (int i = 0; i < 10; i++) {
        int taskId = i;
        executor.submit(() -> {
            System.out.println("Zpracovávám úkol č. " + taskId + " ve vlákně " + Thread.currentThread());
            return "Hotovo";
        });
    }
} // Zde se automaticky volá close(), která počká na dokončení všech úloh
```

{% hint style="info" %}
Problematika plánování a exekutorů bude vysvětlena v [executor-forkjoinpool.md](../executor-forkjoinpool.md "mention").
{% endhint %}

#### 4. Implementace pomocí ThreadFactory

Pokud je nutné předat mechanismus vytváření vláken do externí knihovny nebo frameworku, využívá se tovární rozhraní `ThreadFactory`.

```java
// Definice továrny pro virtuální vlákna s prefixem názvu
ThreadFactory factory = Thread.ofVirtual().name("web-worker-", 0).factory();

// Ruční vytvoření vlákna z továrny
Thread threadFromFactory = factory.newThread(() -> {
    System.out.println("Vlákno vytvořené továrnou: " + Thread.currentThread().getName());
});
threadFromFactory.start();
```
