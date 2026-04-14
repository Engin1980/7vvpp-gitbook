# Princip, CAS operace

## Rozdíl mezi blokujícími a neblokujícími přístupy

Konceptuální rozdíl mezi blokujícími a neblokujícími přístupy spočívá především ve způsobu, jakým vlákno nakládá s časem, kdy čeká na dostupnost prostředku nebo dokončení operace jiného aktéra. V blokujícím paradigmatu je vlákno, které nemůže pokračovat v práci, operačním systémem suspendováno. To znamená, že přestává spotřebovávat procesorový čas a uvolňuje místo pro jiná vlákna. Tento proces je však vykoupen režií spojenou s přepínáním kontextu (context switch), kdy systém musí uložit kompletní stav stávajícího vlákna a obnovit stav jiného, což v moderních vysoce výkonných systémech představuje nezanedbatelnou latenci.

Neblokující přístupy naproti tomu volí strategii aktivního čekání nebo optimistického předpokladu úspěchu. Namísto toho, aby se vlákno nechalo uspat, neustále se v rychlém cyklu dotazuje na stav prostředku nebo se pokouší provést operaci pomocí atomických instrukcí, jako je CAS (Compare-And-Swap). Pokud operace selže z důvodu souběžného zásahu jiného vlákna, neblokující algoritmus se nezastaví, ale okamžitě pokus opakuje s novými daty. Tato technika minimalizuje latenci a odstraňuje rizika spojená s uváznutím (deadlock), protože žádné vlákno nedrží exkluzivní zámek nad prostředkem, který by mohl zablokovat zbytek systému při havárii nebo neočekávaném zpoždění.

Volba mezi těmito dvěma světy se obvykle odvíjí od předpokládané míry konfliktu a požadavků na propustnost systému. Blokující přístupy jsou vhodnější pro scénáře, kde je konkurence o zdroje vysoká nebo kde by čekání mohlo trvat dlouho, neboť šetří energii a výpočetní zdroje CPU. Neblokující algoritmy dominují v nízkoúrovňových infrastrukturních komponentách, jako jsou fronty zpráv nebo finanční systémy s nízkou latencí, kde je prioritou okamžitá reakce. Za extrémní rychlost se zde však platí vyšší spotřebou procesoru při nečinnosti a značně vyšší komplexností návrhu, kde musí vývojář precizně ošetřovat stavy, které by v blokujícím světě jednoduše vyřešil zámek.

| **Kritérium**      | **Blokující přístup (Blocking)**            | **Neblokující přístup (Non-blocking)**    |
| ------------------ | ------------------------------------------- | ----------------------------------------- |
| Mechanismus čekání | Vlákno je suspendováno OS (spánkový režim). | Vlákno aktivně rotuje v cyklu (spin/CAS). |
| Využití CPU        | Nízké při čekání (uvolňuje procesor).       | Vysoké při čekání (vlákno stále běží).    |
| Latence (reakce)   | Vyšší (režie na přepnutí kontextu).         | Minimální (okamžitá reakce na změnu).     |
| Rizika             | Uváznutí (deadlock), inverze priorit.       | Žádný deadlock, riziko livelocku.         |
| Vhodné pro...      | Dlouhé čekání, vysoká konkurence.           | Krátké operace, low-latency systémy.      |
| Složitost kódu     | Jednodušší (standardní zámky).              | Vysoká (precizní atomické operace).       |

Tato tabulka demonstruje, že volba přístupu není otázkou "lepšího" řešení, ale spíše hledáním kompromisu mezi efektivitou využití hardwarových prostředků a požadavky na rychlost odezvy systému. Zatímco blokující přístup šetří energii a výpočetní čas pro ostatní procesy, neblokující přístup upřednostňuje rychlost zpracování za cenu trvalého vytížení přiděleného jádra procesoru.

## Neblokující  přístupy v Java

Při implementaci neblokujících volání a struktur se v Javě opíráme o kombinaci hardwarových instrukcí, specifických tříd z balíku `java.util.concurrent` a moderních asynchronních API. Cílem je umožnit vláknům vykonávat práci i v momentě, kdy čekají na data, a minimalizovat režii spojenou se správou zámků.

#### 1. Atomické operace a Compare-And-Swap (CAS)

Základem všech neblokujících struktur v Javě jsou atomické třídy jako `AtomicInteger`, `AtomicReference` nebo `LongAdder`. Tyto třídy uvnitř nevyužívají klíčové slovo `synchronized`, ale spoléhají na nízkoúrovňovou operaci CAS. Princip funguje tak, že vlákno přečte hodnotu, vypočítá novou a při zápisu se atomicky zeptá procesoru: „Pokud je v paměti stále tato původní hodnota, nahraď ji mou novou hodnotou.“ Pokud se hodnota mezitím změnila jiným vláknem, operace selže a vlákno se v cyklu (tzv. spin-lock) pokusí o totéž znovu s aktuálními daty. Tento přístup eliminuje suspendování vlákna operačním systémem.

#### 2. Neblokující datové struktury

Pro sdílení dat mezi vlákny bez zamykání nabízí Java kolekce navržené pro vysoký souběh. Příkladem je `ConcurrentLinkedQueue`, která využívá algoritmus založený na atomických pointech. Zatímco u standardní `LinkedList` byste museli celou frontu při každém zápisu zamknout, neblokující fronta umožňuje více vláknům přidávat prvky současně pomocí zřetězení atomických operací. Dalším významným prvkem jsou struktury typu Copy-On-Write (např. `CopyOnWriteArrayList`), které řeší souběh tak, že při každé modifikaci vytvoří novou kopii pole, čímž umožňují neomezenému počtu čtenářů přistupovat k datům bez jakékoliv synchronizace.

#### 3. Asynchronní programování a CompletableFuture

Na úrovni aplikační logiky se neblokující volání v Javě realizuje pomocí rozhraní `CompletableFuture`. Namísto toho, aby vlákno čekalo na výsledek dlouhotrvající operace (např. volání databáze), odešle požadavek a okamžitě obdrží objekt, který reprezentuje budoucí výsledek. K tomuto objektu lze pomocí metod jako `thenApply()` nebo `thenAccept()` řetězit další úkoly, které se spustí automaticky v momentě, kdy jsou data k dispozici. Tím se uvolňuje hlavní vlákno pro obsluhu dalších požadavků, což je princip hojně využívaný v reaktivních frameworkech jako Spring WebFlux.

#### 4. Java NIO a selektory

Pro efektivní neblokující vstup a výstup (I/O) disponuje Java balíkem `java.nio`. Hlavním konceptem je zde Selector, který umožňuje jedinému vláknu monitorovat více komunikačních kanálů (např. síťových socketů) najednou. Vlákno se neblokuje na čtení z jednoho konkrétního kanálu, ale dotazuje se selektoru, které kanály již mají data připravená k odběru. Tento model „poháněný událostmi“ (event-driven) je základem vysoce škálovatelných serverů, protože dramaticky snižuje počet potřebných vláken oproti tradičnímu blokujícímu I/O modelu, kde každé spojení vyžaduje vlastní vlákno.

## CAS operace v Java

CAS (Compare-And-Swap) je základní atomická instrukce moderních procesorů, která tvoří srdce všech neblokujících algoritmů a datových struktur (nejen) v Javě. Umožňuje aktualizovat hodnotu v paměti bez použití těžkopádných zámků (mutexů), přičemž garantuje integritu dat i při extrémním souběhu vláken.

#### Princip fungování CAS

Operace CAS pracuje se třemi parametry:

1. Místo v paměti ($$ $V$ $$): Adresa proměnné, kterou chceme aktualizovat.
2. Očekávaná stará hodnota ($$ $A$ $$): Hodnota, o které si vlákno myslí, že je v paměti aktuálně uložena.
3. Nová hodnota ($$ $B$ $$): Hodnota, kterou chceme do paměti zapsat.

Celý proces probíhá na úrovni hardwaru následovně: Procesor porovná hodnotu na adrese $$ $V$ $$ s očekávanou hodnotou $$ $A$ $$. Pokud se shodují, znamená to, že žádné jiné vlákno mezitím hodnotu nezměnilo, a procesor na toto místo okamžitě zapíše novou hodnotu $$ $B$ $$. Pokud se neshodují, operace selže a nic se nezapíše. Celá tato kontrola a zápis proběhnou jako jedna nedělitelná (atomická) operace.

#### Implementace v Javě (Optimistické zamykání)

V Javě se CAS nepoužívá jako přímý příkaz k zastavení ostatních vláken, ale jako základ tzv. optimistického zamykání. Vlákno se "optimisticky" pokusí provést změnu, a pokud zjistí, že mu do toho jiné vlákno zasáhlo (CAS selže), jednoduše celý proces zopakuje s novými daty. Tento cyklus se nazývá CAS loop nebo spin-wait.

**Příklad: Implementace atomického čítače**

Níže je zjednodušená ukázka toho, jak vnitřně funguje například `AtomicInteger`. Namísto klíčového slova `synchronized` používá metodu `compareAndSet`.

Java

```java
import java.util.concurrent.atomic.AtomicInteger;

public class MyAtomicCounter {
    private final AtomicInteger value = new AtomicInteger(0);

    public void increment() {
        int oldValue;
        int newValue;
        
        // CAS Loop: opakuj, dokud se nepodaří hodnotu bezpečně zapsat
        do {
            // 1. Přečteme aktuální stav z paměti
            oldValue = value.get();
            // 2. Vypočítáme novou hodnotu lokálně v registru
            newValue = oldValue + 1;
            
            // 3. Pokusíme se o atomický zápis. 
            // Metoda vrátí 'true', pokud v paměti stále bylo 'oldValue'.
            // Pokud jiné vlákno mezitím hodnotu změnilo, vrátí 'false' a cyklus jede znovu.
        } while (!value.compareAndSet(oldValue, newValue));
    }

    public int getValue() {
        return value.get();
    }
}
```

#### Výhody a omezení

Hlavní výhodou tohoto přístupu je, že vlákna nejsou nikdy suspendována operačním systémem. Nedochází k nákladnému přepínání kontextu (context switch), což vede k extrémně nízké latenci, pokud je konflikt mezi vlákny krátkodobý.

Nevýhodou je tzv. ABA problém. Ten bude vysvětlen později.

## Základní typy

Použití atomických tříd z balíku `java.util.concurrent.atomic` představuje v moderním vývoji v Javě standard pro realizaci neblokujících operací. I když je mechanika CAS (Compare-And-Swap) teoretickým základem těchto struktur, přímá implementace vlastních CAS smyček vývojářem je ve většině případů redundantní a neefektivní. Java toto nízkoúrovňové chování zapouzdřuje do optimalizovaných metod, které zajišťují jak integritu dat, tak maximální využití hardwarového potenciálu. Typy lze je rozdělit do několika základních skupin podle typu dat, se kterými pracují.

#### 1. Skalární atomické typy

Jedná se o nejčastěji používané třídy, které obalují základní datové typy nebo reference.

* AtomicInteger a AtomicLong: Slouží pro práci s celočíselnými hodnotami. Kromě základního čtení a zápisu nabízejí metody pro aritmetické operace.
  * `incrementAndGet()` / `decrementAndGet()`: Zvýší/sníží hodnotu o 1 a vrátí nový výsledek (ekvivalent `++i`).
  * `getAndAdd(delta)`: Přičte zadanou hodnotu a vrátí tu původní.
  * `updateAndGet(operator)`: Aplikuje na hodnotu lambda funkci a uloží výsledek.
* AtomicBoolean: Zajišťuje atomickou práci s logickou hodnotou.
  * `compareAndSet(expected, update)`: Změní hodnotu na `update` pouze tehdy, pokud je aktuální stav roven `expected`.
* AtomicReference\<V>: Umožňuje atomicky aktualizovat odkaz na libovolný objekt. Je klíčová pro stavbu neblokujících datových struktur (např. zřetězených seznamů).

#### 2. Atomická pole

Tyto třídy umožňují pracovat s jednotlivými prvky pole atomicky, což je efektivnější než zamykání celého pole při každé změně indexu.

* AtomicIntegerArray / AtomicLongArray / AtomicReferenceArray\<E>: Poskytují stejné operace jako skalární typy, ale s přidáním parametru indexu.
  * `addAndGet(index, delta)`: Atomicky přičte hodnotu k prvku na daném indexu.
  * `compareAndSet(index, expect, update)`: Provede CAS operaci na konkrétním prvku pole.

#### 3. Akumulátory a sčítače (Striped64)

Pro scénáře s extrémně vysokým zápisem z mnoha vláken jsou určeny specializované třídy, které minimalizují False Sharing (falešné sdílení - bude vysvětleno později) rozkladem hodnoty do více buněk.

* LongAdder / DoubleAdder: Optimalizované pro situace, kdy se hodnota pouze přičítá (např. čítače statistik). Jsou výrazně rychlejší než `AtomicLong` pod velkým tlakem mnoha jader.
  * `add(value)` / `increment()`: Přidá hodnotu bez nutnosti okamžité synchronizace všech jader.
  * `sum()`: Sečte vnitřní buňky a vrátí aktuální celkový výsledek.
* LongAccumulator / DoubleAccumulator: Rozšířená verze sčítačů, která umožňuje definovat vlastní akumulační funkci (např. pro hledání maxima).

#### Souhrn hlavních operací

Většina atomických typů sdílí společné rozhraní pro manipulaci s daty:

1. get() / set(): Čtení a zápis s garancí viditelnosti (jako u `volatile`).
2. lazySet(): Zápis, který garantuje změnu, ale neprodleně ji nevynucuje ostatním jádrům (vhodné pro optimalizaci zápisů, které nespěchají).
3. getAndSet(newValue): Atomicky zapíše novou hodnotu a vrátí tu původní.
4. compareAndSet(expect, update): Základní CAS operace – pokud je v paměti `expect`, zapiš `update` a vrať `true`, jinak nedělej nic a vrať `false`.
