# Happens-Before kontrakt

V souvislosti s paměťovým modelem Javy (Java Memory Model – JMM) je happens-before klíčový koncept, který definuje pravidla pro viditelnost změn v paměti mezi různými vlákny. Bez tohoto kontraktu by moderní procesory a kompilátory, které agresivně optimalizují kód (mění pořadí instrukcí pro vyšší výkon), mohly způsobit, že jedno vlákno uvidí data v nekonzistentním nebo zastaralém stavu.

## Motivace

Hlavním problémem souběžnosti není jen to, že dvě vlákna mohou měnit data naráz, ale především to, že zápis provedený jedním vláknem nemusí být pro druhé vlákno okamžitě viditelný. Procesory mají vlastní vyrovnávací paměti (L1, L2 cache), a pokud není vynucena synchronizace, vlákno může pracovat s kopií dat, která neodpovídá hlavní paměti. Happens-before kontrakt slouží jako záruka, že pokud operace A "předchází" operaci B, pak výsledek operace A musí být pro operaci B viditelný.

## Implementace: Základní pravidla kontraktu

Kontrakt není jedna konkrétní metoda, ale sada pravidel, která JMM garantuje. Pokud váš kód tato pravidla dodržuje, Java vám zaručí konzistenci. Mezi nejdůležitější patří:

* Pravidlo programu: V rámci jednoho vlákna se operace dějí v tom pořadí, v jakém jsou zapsány v kódu.
* Monitor Lock (Synchronized): Uvolnění zámku (konec `synchronized` bloku) se děje před každým následným získáním téhož zámku. Vše, co vlákno udělalo před odemčením, uvidí vlákno, které zámek právě zamklo.
* Volatile proměnné: Zápis do `volatile` proměnné se děje před každým následným čtením této proměnné. Zápis do `volatile` navíc "protlačí" do hlavní paměti i všechny ostatní změny, které vlákno do té doby provedlo.
* Start/Join vlákna: Volání `thread.start()` se děje před jakoukoli akcí v daném vlákně. Ukončení vlákna se děje předtím, než jiná vlákna zjistí jeho konec pomocí `thread.join()`.
* Tranzitivita: Pokud operace A předchází operaci B a operace B předchází operaci C, pak platí, že A předchází C.

## Příklad v praxi

Představme si dvě vlákna sdílející obyčejnou proměnnou `int x = 0;` a `boolean ready = false;`.

1. Vlákno A nastaví `x = 42;` a poté nastaví `ready = true;`.
2. Vlákno B čeká v cyklu `while (!ready);` a poté vypíše `x`.

Bez happens-before kontraktu (pokud by `ready` nebyla `volatile`) by se mohlo stát, že kompilátor prohodí pořadí zápisů ve Vlákně A, nebo že Vlákno B uvidí `ready = true`, ale v jeho lokální cache bude stále stará hodnota `x = 0`.

Pokud však označíme `ready` jako volatile, vytvoříme happens-before hranu. Vše, co Vlákno A udělalo před zápisem do `ready` (včetně nastavení `x = 42`), je zaručeně viditelné pro Vlákno B poté, co úspěšně přečte `ready = true`. Kontrakt tedy funguje jako "bariéra", která zajistí synchronizaci paměti mezi těmito dvěma body.

## Shrnutí

* Definice: Happens-before je formální záruka Javy, že operace provedená jedním vláknem je viditelná pro jiné vlákno.
* Účel: Zamezení reordering (přeuspořádání instrukcí) a zajištění konzistence cache pamětí.
* Nástroje: Kontrakt vynucujeme pomocí `synchronized`, `volatile`, `final` polí nebo tříd z `java.util.concurrent` (např. `Lock`, `AtomicInteger`, `CountDownLatch`).
