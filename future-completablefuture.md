# Future

## Motivace

V tradičním synchronním modelu dochází k zastavení vykonávání programu (blokování), dokud není dokončena určitá operace, jako je například dotaz do databáze nebo stažení dat z internetu. Nevýhodnost blokování vykonávání programu spočívá především v neefektivním nakládání se systémovými prostředky, konkrétně s procesorovými vlákny. V tradičním modelu je pro každý blokující požadavek vyhrazeno jedno vlákno, které po dobu čekání na externí událost (vstupně-výstupní operace, např. čtení/zápis dat z/do databáze, komunikace s jiným serverem) nevykonává žádnou užitečnou práci, a přesto spotřebovává operační paměť pro svůj zásobník (stack) a vytěžuje jádro procesoru "čekáním". Při vysokém počtu souběžných uživatelů nebo úloh (řádově už nižší tisíce) může dojít k vyčerpání dostupných vláken operačního systému, což vede k neschopnosti aplikace přijímat nové požadavky, k drastickému nárůstu odezvy a v krajním případě k úplnému pádu systému z důvodu nedostatku paměti (tzv. Thread Starvation).

Dalším kritickým aspektem je degradace uživatelské zkušenosti a celkové propustnosti aplikace. Pokud je například v desktopové nebo mobilní aplikaci synchronně prováděn síťový dotaz v hlavním vlákně, dojde k „zamrznutí“ celého uživatelského rozhraní, které přestane reagovat na vstupy, jako je klikání nebo posouvání obsahu. V kontextu distribuovaných systémů a mikroslužeb pak synchronní blokování způsobuje kaskádovité šíření prodlev; pokud jedna služba v řetězci zpomalí, jsou postupně blokovány všechny služby na ni navázané, což znemožňuje efektivní škálování.

V oblasti asynchronního programování v jazyce Java jsou Future a CompletableFuture klíčovými mechanismy, které umožňují manipulaci s výsledky operací, jež probíhají na pozadí a nejsou dokončeny okamžitě. Tyto nástroje slouží k efektivnímu využívání procesorového času, neboť neblokují hlavní vlákno aplikace během čekání na dokončení pomalých úloh (např. síťová komunikace nebo výpočty).

## Rozhraní Future

Rozhraní `Future`, uvedené v Java 5, reprezentuje výsledek asynchronního výpočtu. Lze na něj nahlížet jako na „příslib“ hodnoty, která bude k dispozici v budoucnu. Jakmile je úloha odeslána k vykonání (typicky do `ExecutorService`), je vrácen objekt typu `Future`, pomocí něhož lze kontrolovat stav operace.

Hlavní omezení tohoto rozhraní spočívají v jeho pasivitě. Pro získání výsledku je nutné volat metodu `.get()`, která je blokující – vlákno tedy zastaví svou činnost a čeká, dokud není výsledek hotov. Stejně tak neexistuje jednoduchý způsob, jak řetězit více operací za sebou nebo reagovat na dokončení úlohy pomocí zpětného volání (callbacku).

## Třída CompletableFuture\<T>

S příchodem Java 8 byla představena generická třída `CompletableFuture`, která implementuje rozhraní `Future` a `CompletionStage`. Tento nástroj odstraňuje většinu nedostatků svého předchůdce a přináší do Javy prvky funkcionálního programování. Nejdůležitější vlastností je schopnost manuálně dokončit výpočet a definovat následné kroky, které se mají provést automaticky po získání výsledku.

Mezi hlavní výhody patří:

* Neblokující řetězení: Pomocí metod jako `.thenApply()` nebo `.thenAccept()` jsou definovány další operace, které se spustí ihned po dokončení předchozího kroku. Můžeme tedy jednoduše definovat sekvenci _udělej toto -> pak udělej toto -> pak udělej něco dalšího_.
* Kombinování úloh: Více asynchronních operací běžících současně může být snadno sloučeno (např. čekání na dokončení všech úloh pomocí `allOf()`).
* Zpracování chyb: Obsahuje vestavěné mechanismy pro deklarativní ošetření výjimek (metody `.exceptionally()` nebo `.handle()`).
* Manuální kompletace: Objekt lze vytvořit prázdný a jeho hodnota může být nastavena explicitně z jakékoliv části kódu.

### Generičnost

Třída `CompletableFuture` je definována jako generický typ, což v praxi znamená, že její definice obsahuje typový parametr, zpravidla označovaný jako `<T>`. Generičnost umožňuje, aby stejná implementační logika obsluhovala různé druhy dat, od jednoduchých čísel až po komplexní doménové objekty, podle toho, co má být výsledkem požadované operace.

Důvody pro využití generičnosti u tohoto konceptu jsou popsány v následujících bodech:

* Typová bezpečnost (Type Safety): Díky generikům je kompilátor schopen kontrolovat, zda jsou s výsledkem operace prováděny operace odpovídající jeho typu. Tím se předchází chybám typu `ClassCastException`, které by jinak mohly nastat při běhu programu, pokud by se pracovalo s obecným typem `Object`.
* Odstranění nutnosti přetypování: Při získávání výsledku z asynchronního výpočtu není vyžadováno manuální přetypování na konkrétní třídu. Výsledek metody `.get()` nebo parametr v metodě `.thenAccept(T action)` je již přímo požadovaného typu.
* Flexibilita a znovupoužitelnost: Jedna třída `CompletableFuture` může reprezentovat výsledek stažení textového souboru (`CompletableFuture<String>`), seznamu uživatelů z databáze (`CompletableFuture<List<User>>`) nebo výsledek matematického výpočtu (`CompletableFuture<Integer>`).

V případech, kdy asynchronní operace nevykazuje žádnou návratovou hodnotu a je prováděna čistě pro své vedlejší účinky (například zápis do logu, odeslání e-mailu nebo aktualizace stavu v databázi), je využíván generický typ `CompletableFuture<Void>`. I přes absenci návratové hodnoty je však stále zachována možnost sledovat stav dokončení úlohy, řetězit další akce pomocí metody `.thenRun()` nebo reagovat na případné výjimky, které během vykonávání nastaly.

{% hint style="info" %}
Jelikož v jazyce Java nelze použít klíčové slovo `void` jako parametr generické třídy, je k tomuto účelu určena speciální třída `Void`, která funguje jako zástupný symbol pro prázdný výsledek.
{% endhint %}

V následující tabulce je znázorněno, jak se mění datový typ objektu v závislosti na vykonávané operaci:

<table data-header-hidden><thead><tr><th width="251"></th><th width="226"></th><th></th></tr></thead><tbody><tr><td><strong>Definice typu</strong></td><td><strong>Význam</strong></td><td><strong>Příklad výsledku</strong></td></tr><tr><td><code>CompletableFuture&#x3C;String></code></td><td>Příslib textového řetězce</td><td>"Operace dokončena"</td></tr><tr><td><code>CompletableFuture&#x3C;Integer></code></td><td>Příslib celého čísla</td><td>42</td></tr><tr><td><code>CompletableFuture&#x3C;Void></code></td><td>Operace bez návratové hodnoty</td><td>(pouze signál o dokončení)</td></tr><tr><td><code>CompletableFuture&#x3C;Double></code></td><td>Příslib desetinného čísla</td><td>3.14</td></tr></tbody></table>

### Vytvoření instance

Existuje několik standardních způsobů, jak vytvořit instanci třídy CompletableFuture, přičemž volba konkrétní metody závisí na tom, zda má být operace spuštěna okamžitě, asynchronně, nebo zda má být vytvořen prázdný objekt pro manuální dokončení v budoucnu. Pro nejběžnější scénáře jsou využívány statické tovární metody, které automaticky zajišťují spuštění úlohy v rámci sdíleného fondu vláken (`ForkJoinPool.commonPool()`), pokud není specifikován jiný vykonavatel.

Asynchronní operace jsou nejčastěji zahajovány pomocí metod `supplyAsync` a `runAsync`. Rozdíl mezi nimi spočívá v tom, zda je od dané úlohy očekáván výsledek.

* `supplyAsync(Supplier supplier)`: Tato metoda je volena v situacích, kdy má asynchronní výpočet vrátit hodnotu. Jako parametr přijímá funkcionální rozhraní `Supplier`, které vrací výsledek definovaného typu.
* `runAsync(Runnable runnable)`: Tento způsob je využíván pro úlohy, které pouze vykonají určitou činnost bez návratové hodnoty (výsledkem je `CompletableFuture<Void>`).
* `completedFuture(T value)`: Používá se v případech, kdy je výsledek již znám a je třeba jej zabalit do objektu `CompletableFuture`, aby bylo možné využít jeho API pro další řetězení.
* Konstruktor `new CompletableFuture<>()`: Vytvoří se "neúplný" objekt, který neobsahuje žádnou logiku pro spuštění. Taková instance zůstává v nedokončeném stavu, dokud na ní není z jiného vlákna explicitně zavolána metoda `.complete(T value)`.

Níže jsou uvedeny konkrétní ukázky kódu pro jednotlivé způsoby inicializace. Každý příklad demonstruje odlišný přístup k zahájení asynchronní operace v závislosti na tom, zda je vyžadován výsledek a zda má být úloha spuštěna automaticky nebo řízena manuálně.

#### 1. Metoda supplyAsync

Tento způsob je využíván, pokud je cílem provést výpočet na pozadí a získat jeho výsledek. Metoda přijímá `Supplier`, což je funkce, která nebere žádný argument, ale vrací hodnotu.

```java
// Spustí se asynchronní úloha, která po 2 sekundách vrátí text
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    // Simulace náročného výpočtu nebo síťového dotazu
    return "Výsledek z databáze";
});
```

Úloha je automaticky odeslána do výchozího fondu vláken. Jakmile je kód uvnitř lambda výrazu dokončen, objekt `future` se naplní vráceným řetězcem.

#### 2. Metoda runAsync

Tato varianta je volena v situacích, kdy je nutné vykonat práci, která nemá žádný výstup (např. pouhý zápis do souboru).

```java
// Spustí se asynchronní úloha bez návratové hodnoty
CompletableFuture<Void> future = CompletableFuture.runAsync(() -> {
    System.out.println("Zápis do logu probíhá v jiném vlákně...");
});
```

Výsledkem je `CompletableFuture<Void>`. To znamená, že po dokončení nelze z objektu získat žádná data, ale lze na něj navázat další operace (např. „po dokončení zápisu vypiš potvrzení“).

#### 3. Metoda completedFuture

Tato statická metoda se používá tehdy, pokud je hodnota již k dispozici, ale architektura programu vyžaduje, aby byla předána jako objekt typu `CompletableFuture`.

```java
// Vytvoří se instance, která je již od začátku ve stavu "dokončeno"
CompletableFuture<Integer> future = CompletableFuture.completedFuture(42);
```

V tomto případě nedochází k žádnému asynchronnímu výpočtu. Objekt je okamžitě připraven k použití. To je užitečné například při testování nebo při implementaci rozhraní, kde je výsledek v některých případech znám ihned.

#### 4. Manuální vytvoření pomocí konstruktoru

Tento přístup je nejvíce flexibilní, neboť umožňuje vytvořit "prázdný" objekt a jeho dokončení nechat na libovolném jiném vlákně nebo události.

```java
// Vytvoření prázdného příslibu
CompletableFuture<String> manualFuture = new CompletableFuture<>();

// Někde jinde v kódu (např. v reakci na příchozí zprávu ze sítě)
manualFuture.complete("Data dorazila manuálně");
```

Objekt je vytvořen v neúplném stavu. Vlákna volající metodu `.get()` na tomto objektu budou blokována (nebo budou čekat na callback), dokud není odněkud zavolána metoda `.complete()`.

**Přehledové shrnutí syntaxe**

V tabulce je znázorněna korelace mezi použitou metodou a funkcionálním rozhraním, které je jí předáváno:

<table data-header-hidden><thead><tr><th width="165"></th><th width="193"></th><th></th></tr></thead><tbody><tr><td><strong>Metoda</strong></td><td><strong>Použité rozhraní</strong></td><td><strong>Návratový typ v příkladu</strong></td></tr><tr><td><code>supplyAsync</code></td><td><code>Supplier&#x3C;T></code></td><td><code>CompletableFuture&#x3C;T></code></td></tr><tr><td><code>runAsync</code></td><td><code>Runnable</code></td><td><code>CompletableFuture&#x3C;Void></code></td></tr><tr><td><code>completedFuture</code></td><td>(přímá hodnota)</td><td><code>CompletableFuture&#x3C;T></code></td></tr><tr><td><code>new</code></td><td>(žádné)</td><td><code>CompletableFuture&#x3C;T></code></td></tr></tbody></table>

## Řetězící operace

Řetězení operací představuje jednu z nejvýznamnějších vlastností třídy CompletableFuture, neboť umožňuje definovat sekvenci kroků, které mají být vykonány automaticky po dokončení předchozí úlohy. Tento mechanismus je založen na principu, kdy každá metoda pro řetězení vrací novou instanci `CompletableFuture`, čímž vzniká plynulý tok dat (pipeline). Namísto čekání na výsledek jsou v kódu pouze deklarovány transformace a reakce, které se aktivují asynchronně, což vede k výrazně čistší architektuře bez nutnosti vnořených zpětných volání.

#### Operace zpracování dat

V praxi jsou nejčastěji využívány tři typy operací v závislosti na tom, zda transformují výsledek, pouze jej spotřebovávají, nebo pouze signalizují dokončení.

* `thenApply(Function<T, R> fn)`: Tato metoda je využívána pro transformaci výsledku. Přijímá funkci, která vezme výstup předchozího kroku, provede nad ním výpočet a vrátí nový výsledek. Odpovídá operaci `map` známé ze Stream API.
* `thenAccept(Consumer action)`: Používá se v okamžiku, kdy je potřeba výsledek pouze zpracovat (např. vypsat do konzole nebo uložit do databáze), ale není vyžadována žádná návratová hodnota pro další kroky. Výsledkem je `CompletableFuture<Void>`.
* `thenRun(Runnable action)`: Tato metoda je volána, pokud má být po dokončení úlohy provedena akce, která ke své činnosti vůbec nepotřebuje výsledek předchozího kroku. Používá se pro spuštění čistě signálních nebo úklidových činností.

Níže jsou uvedeny modelové situace použití jednotlivých metod pro demonstraci jejich syntaxe:

```java
// 1. Transformace výsledku pomocí thenApply
CompletableFuture<Integer> lengthFuture = CompletableFuture.supplyAsync(() -> "Hello")
    .thenApply(text -> text.length()); // Transformuje String na Integer
    // vrací CompletableFuture<Integer>

// 2. Spotřebování výsledku pomocí thenAccept
CompletableFuture<Void> printFuture = CompletableFuture.supplyAsync(() -> "Data")
    .thenAccept(data -> System.out.println("Zpracováno: " + data)); // Spotřebuje data
    // vraci CompletableFuture<Void>

// 3. Spuštění akce po dokončení pomocí thenRun
CompletableFuture<Void> cleanupFuture = CompletableFuture.runAsync(() -> doWork())
    .thenRun(() -> System.out.println("Všechny úkoly byly hotovy."));
    // vraci CompletableFuture<Void>    
```

Rychlá přehledová tabulka:

<table data-header-hidden><thead><tr><th width="134"></th><th width="168"></th><th width="156"></th><th></th></tr></thead><tbody><tr><td><strong>Metoda</strong></td><td><strong>Vstup z předchozího kroku</strong></td><td><p><strong>Návratová</strong> </p><p><strong>hodnota</strong></p></td><td><strong>Typ výsledného Future</strong></td></tr><tr><td>thenApply</td><td>ANO (výsledek)</td><td><p>ANO </p><p>(nová hodnota)</p></td><td><code>CompletableFuture&#x3C;R></code></td></tr><tr><td>thenAccept</td><td>ANO (výsledek)</td><td>NE</td><td><code>CompletableFuture&#x3C;Void></code></td></tr><tr><td>thenRun</td><td>NE</td><td>NE</td><td><code>CompletableFuture&#x3C;Void></code></td></tr></tbody></table>

{% hint style="info" %}
Ke výše uvedeným (i některým níže uvedeným) funkcím existují ekvivalenty s postfixem `...Async()`. Tyto funkce říkají, že navazující operace se má vykonávat v novém vlákně (jinak se defaultně dělá ve stejném). to se může hodit, když a) chceme aktuální vlákno uvolnit pro jiné perace a odlehčit mu; b) potřebujeme novou práci spustit na jiném (konkrétním) vlákně/scheduleru.
{% endhint %}

#### Operace zřetězení další CompletableFuture

V rámci zřetězení se často dostaneme do stavu, kdy potřebujeme předchozí výsledek využít k dalšímu volání  `CompletableFuture` pro nové výsledky (například jsme obdrželi název státu a nyní se chceme doptat na jeho hlavní město; nebo jsme obdrželi seznam měst a pro každé potřebujeme dotáhnout aktuální počasí). Tedy v situacích, kdy asynchronní operace na sebe navazují takovým způsobem, že výstupem jedné je další asynchronní příslib. Zatímco metoda `thenApply` slouží k transformaci hodnoty (podobně jako u stream-kolekcí `map`), metoda `thenCompose` se používá k "vyrovnání" (flattening) vnořených struktur, což odpovídá operaci `flatMap` známé z funkcionálního programování. Výsledkem `thenCompose` jakoby není nový `CompletableFuture`, ale jeho výsledek.

Pokud by byla v takové situaci použita standardní metoda `thenApply`, výsledkem by byl vnořený objekt typu `CompletableFuture<CompletableFuture<R>>`, což je pro další manipulaci nepraktické. `thenCompose` zajistí, že se tyto dvě asynchronní operace spojí do jedné, čímž vznikne lineární řetězec.

Hlavní charakteristiky jsou popsány následovně:

* Závislost úloh: Druhá úloha začne až po úspěšném dokončení první, přičemž využívá její výsledek jako svůj vstup.
* Zamezení vnoření: Metoda automaticky "rozbalí" vnitřní `CompletableFuture`, takže výsledkem je opět jednoduchý `CompletableFuture<R>`.
* Asynchronní orchestrace: Je ideální pro skládání více asynchronních služeb, například když je nejprve načten uživatel z databáze a následně jsou pro tohoto uživatele asynchronně vyžádána jeho oprávnění z jiné služby.

**Příklad.**

Rozdíl v použití je nejlépe patrný na srovnání s metodou `thenApply`. Předpokládejme existenci metody `getInfo(String id)`, která vrací `CompletableFuture<String>`.

```java
// Použití thenApply by vytvořilo nežádoucí vnoření:
CompletableFuture<CompletableFuture<String>> nested = 
    CompletableFuture.supplyAsync(() -> "id123")
    .thenApply(id -> getInfo(id)); // Vrací Future uvnitř Future

// Použití thenCompose vytvoří plochou strukturu:
CompletableFuture<String> flat = 
    CompletableFuture.supplyAsync(() -> "id123")
    .thenCompose(id -> getInfo(id)); // Vrací přímo Future s výsledkem
```

Díky metodě thenCompose je zachována čitelnost kódu i při komplexních asynchronních scénářích, neboť umožňuje skládat jednotlivé kroky za sebe, místo aby docházelo k jejich nepřehlednému zanořování do sebe.

## Další operace

Kromě asynchronního řetězení a transformací nabízí třída CompletableFuture celou řadu metod pro přímou interakci s výsledkem, synchronizaci vláken a hromadné řízení úloh. Tyto funkce jsou nezbytné v momentě, kdy je zapotřebí asynchronní tok uzavřít, počkat na jeho výsledek v rámci hlavního vlákna, nebo koordinovat několik paralelně běžících procesů najednou.

#### Získání výsledku a čekání na dokončení

Pro situace, kdy je nutné „přepnout“ z asynchronního režimu zpět do synchronního a počkat na konkrétní hodnotu, jsou definovány především následující metody:

* `get()`: Jedná se o blokující metodu, která zastaví vykonávání aktuálního vlákna, dokud není výpočet dokončen. Vyžaduje ošetření kontrolovaných výjimek (`InterruptedException` a `ExecutionException`).
* `get(long timeout, TimeUnit unit)`: Tato varianta umožňuje specifikovat maximální dobu čekání. Pokud výsledek není v daném limitu k dispozici, je vyhozena výjimka `TimeoutException`.
* `getNow(T valueIfAbsent)`: Tato neblokující metoda se pokusí získat výsledek okamžitě. Pokud výpočet ještě nebyl dokončen, vrátí místo výsledku předem definovanou náhradní hodnotu (`valueIfAbsent`).
* `join()`: Funguje velmi podobně jako `get()`, ale na rozdíl od ní nevyhazuje kontrolované výjimky, nýbrž výjimky typu `Unchecked`. To ji činí vhodnější pro použití v rámci lambda výrazů a Stream API.

#### Koordinace více instancí

Často je vyžadována schopnost reagovat na stav několika asynchronních úloh současně. K tomuto účelu slouží statické metody, které umožňují seskupování:

* `allOf(CompletableFuture<?>... cfs)`: Vytvoří nový `CompletableFuture`, který je považován za dokončený až ve chvíli, kdy jsou úspěšně dokončeny všechny zadané úlohy. Tato metoda nevrací výsledky jednotlivých úloh, slouží pouze jako synchronizační bariéra pro čekání na celek.
* `anyOf(CompletableFuture<?>... cfs)`: Tato metoda je dokončena v momentě, kdy je hotova alespoň jedna (kterákoliv) z předaných úloh. Je ideální pro scénáře, kde je stejný dotaz posílán na více serverů a stačí výsledek od toho nejrychlejšího.

#### Přehled metod pro správu stavu a výsledku

V následující tabulce jsou shrnuty doplňkové funkce, které umožňují pokročilou kontrolu nad instancemi:

<table data-header-hidden><thead><tr><th></th><th width="110"></th><th></th></tr></thead><tbody><tr><td><strong>Metoda</strong></td><td><strong>Typ</strong></td><td><strong>Popis</strong></td></tr><tr><td>complete(T value)</td><td>Terminální</td><td>Manuálně dokončí objekt s danou hodnotou, pokud ještě nebyl dokončen.</td></tr><tr><td>completeExceptionally(Throwable ex)</td><td>Terminální</td><td>Manuálně ukončí úlohu s chybovým stavem a předanou výjimkou.</td></tr><tr><td>isDone()</td><td>Stavová</td><td>Vrací logickou hodnotu <code>true</code>, pokud je výpočet u konce (úspěch i chyba).</td></tr><tr><td>cancel(boolean interrupt)</td><td>Kontrolní</td><td>Pokusí se přerušit vykonávání úlohy a zrušit její platnost.</td></tr><tr><td>handle(BiFunction&#x3C;T, R Throwable,> fn)</td><td>Ošetřující</td><td>Umožňuje transformovat výsledek nebo chybu v jednom kroku po dokončení.</td></tr></tbody></table>

## Ošetření chyb a výjimek

Zpracování chyb v asynchronním prostředí třídy CompletableFuture se zásadně liší od standardního synchronního programování, kde jsou využívány bloky `try-catch`. Jelikož výpočty probíhají v jiných vláknech, výjimky nejsou vyhazovány přímo do hlavního vlákna, ale jsou zachyceny a uloženy uvnitř objektu `CompletableFuture`. Pokud v jakémkoli kroku asynchronního řetězce dojde k chybě, tento krok a všechny následující kroky (které nejsou určeny k ošetření chyb) jsou považovány za neúspěšné. Výjimka je poté „přepravována“ řetězcem dál, dokud nenarazí na metodu určenou k jejímu zpracování, nebo dokud není vyvolána při pokusu o získání výsledku pomocí metod `.get()` nebo `.join()`.

### Základní ošetření

Pro deklarativní a elegantní řešení těchto stavů jsou k dispozici tři hlavní mechanismy, které umožňují budovat robustní asynchronní procesy.  Každá z těchto metod slouží k jinému účelu a volí se podle toho, zda má být výsledek transformován, nebo zda je vyžadována pouze reakce na chybu.

* `exceptionally(Function<Throwable, T> fn)`: Tato metoda funguje jako zástupná hodnota v případě selhání. Přijímá výjimku jako argument a vrací náhradní výsledek stejného typu, čímž umožní řetězci pokračovat, jako by k chybě nedošlo. Je to obdoba bloku `catch`, který vrací výchozí hodnotu.
* `handle(BiFunction<T, R Throwable,> fn)`: Jedná se o univerzální metodu, která je zavolána vždy, bez ohledu na to, zda předchozí krok uspěl, či nikoliv. Přijímá jak výsledek (pokud existuje), tak výjimku (pokud nastala). Umožňuje transformovat úspěšný výsledek i ošetřit chybu v rámci jednoho bloku.
* `whenComplete(BiConsumer<T, Throwable> action)`: Tato metoda se používá pro akce, které nemají měnit výsledek, ale mají na něj reagovat (např. logování). Podobně jako `handle` přijímá výsledek i výjimku, ale jelikož vrací `void` (respektive původní výsledek), slouží spíše jako blok `finally` se schopností nahlížet na data.

### Propagace výjimek a terminální stavy

Pokud není výjimka v rámci řetězce ošetřena výše uvedenými metodami a je na objektu zavolána metoda `.get()`, dojde k vyhození kontrolované výjimky `ExecutionException`, která v sobě (jako příčinu – _cause_) balí původní chybu. Při použití metody `.join()` je vyhozena nekontrolovaná `CompletionException`. Tento mechanismus zajišťuje, že informace o chybě není nikdy ztracena, i když k ní došlo v jiném kontextu provádění.

Pro manuální ukončení s chybou je využívána metoda `completeExceptionally(Throwable ex)`. Ta je klíčová v situacích, kdy aplikace na základě vnější události (např. timeoutu nebo nevalidního vstupu) rozhodne, že asynchronní příslib již nemůže být splněn a je nutné informovat všechny navázané konzumenty o selhání. Tímto způsobem je zajištěno, že se asynchronní řetězec nezasekne v nekonečném čekání na hodnotu, která nikdy nedorazí.

### Příklad

V následujícím příkladu je demonstrován typický scénář asynchronního zpracování dat, který zahrnuje načtení hodnoty, její transformaci a následné ošetření potenciální chyby. Celý proces je navržen tak, aby i v případě selhání některého z kroků aplikace neskončila chybou, ale vrátila bezpečný náhradní výsledek.

V rámci řetězce jsou využity tyto operace:

1. supplyAsync: Simuluje asynchronní získání číselného vstupu.
2. thenApply: Provede matematickou operaci (v tomto případě dělení), která je náchylná k vyvolání výjimky (dělení nulou).
3. exceptionally: Slouží jako záchranná síť, která v případě chyby zachytí výjimku a poskytne neutrální výsledek (nulu).
4. thenAccept: Finální krok, který pouze vypíše konečný výsledek do konzole.

```java
import java.util.concurrent.CompletableFuture;

public class AsyncExample {
    public static void main(String[] args) {
        // Vytvoření a spuštění asynchronního řetězce
        CompletableFuture.supplyAsync(() -> {
            // Simulace získání čísla z externího zdroje
            return 10;
        })
        .thenApply(number -> {
            // Záměrné vyvolání chyby pro demonstraci (dělení nulou)
            // V reálném scénáři zde může nastat např. chyba sítě
            return number / 0; 
        })
        .exceptionally(ex -> {
            // Tato část se spustí pouze v případě, že v předchozích krocích nastala chyba
            System.err.println("Nastala chyba: " + ex.getMessage());
            // Je vrácena náhradní hodnota, aby řetězec mohl pokračovat
            return 0; 
        })
        .thenAccept(result -> {
            // Zpracování finálního výsledku (buď vypočteného, nebo náhradního)
            System.out.println("Konečný výsledek zpracování: " + result);
        })
        .join(); // čeká na dokončení operace

        // Jelikož jsou operace asynchronní, je nutné pomocí 'join()' počkat,
        // aby program neskončil dříve, než se vlákna na pozadí dokončí.
    }
}
```
