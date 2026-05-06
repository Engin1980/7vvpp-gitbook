# TODO: Řízení běhu vláken

## Motivace

Při spuštění více vláken v rámci jednoho programu je občas nutné udržovat jejich synchronizaci. Cílem je zajistit, aby určité vlákno nezačalo vykonávat určitou operaci dříve, než jiné vlákno určitou operaci dokončí.

Příkladem může být inicializace databázového řešení – první vlákno spouští databázový stroj a čeká na jeho spuštění, ostatní vlákna, pokud chtějí s databází pracovat, však nejdříve musí počkat na dokončení části činnosti prvního vlákna.

Technik řešení je několik – počínaje tou nejjednodušší (a jednoznačně **nejhorší**), kdy druhé vlákno uvedeme do smyčky, čekající na hodnotu příznaku udávajícího, že vše je připraveno. Potřebujeme jednu proměnnou s příznakem:

{% code lineNumbers="true" %}
```java
private static jeVsePripraveno = false;
```
{% endcode %}

Dále vlákno realizující přípravu může vykonávat následující kód:

{% code lineNumbers="true" %}
```java
public static void pripravujiciOperace(){
  pripravuji();
  jeVsePripraveno = true;
} 
```
{% endcode %}

A nakonec vlákno, které čeká na příznak, aby mohlo začít pracovat:

{% code lineNumbers="true" %}
```java
public static void provadejiciOperace(){
  // nekonecna smycka
  while (jeVsePripraveno == false) {} // <- čeká na příznak
  pracujDal();
} 
```
{% endcode %}

Tento přístup - nazývaný _busy waiting_, je krajně nevýhodný. Důvodem je nekonečná smyčka, která se vykonává pořád dokolečka. Procesor totiž ve smyčce neustále provádí operace:

1. _while_ je všechno připraveno – vyhodnocení podmínky
2. provedení kódu pracuj dál.

Tedy neustále provádí nějaké instrukce, i když jejich „přidaná hodnota programu“ je nulová. Tato technika dokáže efektivně zaměstnat procesor, přičemž aplikace vlastně nic kloudného nedělá.

## Představení wait/notify

Mechanismus wait/notify představuje základní nízkoúrovňový nástroj pro koordinaci vláken v Javě, jehož hlavní motivací je efektivní správa systémových prostředků při synchronizaci, kdy jedno vlákno čeká na změnu stavu poskytnutou jiným vláknem - například konzument čeká, až producent vloží data do prázdné fronty.

Oproti _busy-waiting_  tento mechanismus umožňuje pomocí `wait()` vlákno zcela uspat, vyřadit jej z plánování procesoru a uvolnit zámek/zámky objektu pro jiná vlákna. Teprve ve chvíli, kdy jiné vlákno změní stav sdílených dat a vyvolá `notify()`, je čekající vlákno probuzeno a zařazeno zpět do fronty na získání zámku/ů.

Klíčovým aspektem a motivací pro integraci těchto metod přímo do kořenové třídy `Object` je jejich atomická vazba na vnitřní zámek (intrinsic lock). Metoda `wait()` je navržena tak, aby v jediném nedělitelném kroku vlákno uspala a zároveň uvolnila monitor objektu. Pokud by tyto operace neproběhly atomicky, hrozil by vznik uvíznutí (deadlock), kdy by vlákno usnulo, ale stále drželo zámek, čímž by znemožnilo ostatním vláknům provést změnu stavu potřebnou k jeho probuzení.

Celkově vzato je motivací pro wait/notify vytvoření úsporného komunikačního kanálu mezi vlákny, který zajišťuje, že vlákna pracují pouze tehdy, když mají k dispozici potřebná data nebo zdroje. Zatímco samotná synchronizace (`synchronized`) zajišťuje vzájemné vyloučení a integritu dat, wait/notify přidává vrstvu řízení toku, která umožňuje vláknům koordinovaně předávat řízení v závislosti na aktuálním stavu aplikace.

### Princip

Samotná procedura koordinace mezi vlákny pomocí metod `wait()`, `notify()` a `notifyAll()` se striktně váže na vlastnictví zámku (monitoru) daného objektu.&#x20;

{% hint style="info" %}
Monitorem, nad kterým se stavý zámek, se myslí úplně libovolná jakákoliv instance třídy v Javě, uložená v proměnné. Nejedná se tedy o instanci nějakého speciálního typu, ale prostě o libovolný objekt v proměnné, například:

`Object monitor = new Object();`
{% endhint %}

Celý proces probíhá v následujících logických krocích:

#### 1. Získání monitoru

Vlákno $W, které chce volat synchronizační metody, musí nejprve získat zámek objektu. Toho se typicky dosahuje vstupem do bloku `synchronized(monitor)`.&#x20;

{% hint style="warning" %}
Volání funkcí `wait()`, `notify()` a `notifyAll()` musí povinně probíhat v synchronizovaném bloku daného monitoru, tedy `synchronized(x) { x.wait(); }`.&#x20;

Pokud by libovolné vlákno volalo `wait()` nebo `notify()` bez vlastnictví zámku, vyvolá Java výjimku `IllegalMonitorStateException`.
{% endhint %}

#### 2. Volání wait() a uvolnění zámku

V momentě, kdy vlákno $W zjistí, že podmínka pro jeho práci není splněna (např. zásobník je prázdný), zavolá metodu `wait()`. V ten okamžik se dějí dvě věci:

* Vlákno $W je zařazeno do čekací množiny (wait set) daného objektu.
* Vlákno $W je suspendováno (uspáno) a atomicky uvolní zámek objektu monitoru, aby k němu mohla přistoupit jiná vlákna. Uvoňují se i všechny případné ostatní zámky, které pomocí `synchronized` vlákno vlastní.

#### 3. Změna stavu jiným vláknem

Jiné vlákno (např. producent) $N získá uvolněný zámek téhož objektu a provede operaci, která změní stav (např. přidá prvek do zásobníku). Poté, co je práce hotova, vlákno $N vyvolá metodu `notify()` nebo `notifyAll()`.

* `notify()`: Vybere náhodně jedno vlákno z čekací množiny a přesune jej do stavu čekání na zámek (blocked/runnable).
* `notifyAll()`: Probudí všechna čekající vlákna. Toto je bezpečnější varianta, pokud na objektu čeká více vláken s různými podmínkami.

#### 4. Opětovné získání zámku a probuzení

Důležitým detailem je, že probuzené vlákno $W nezačne běžet okamžitě. Nejprve musí znovu získat zámek objektu (případně všech, pokud bylo zanořeno hlouběji ve více blocích `synchronized`). Jakmile vlákno $N, které volalo `notify()`, opustí svůj synchronizovaný blok a uvolní monitor, probuzené vlákno $W o něj začne soupeřit s ostatními vlákny.

#### 5. Re-evaluace podmínky (While Loop)

Jakmile probuzené vlákno $W znovu získá zámek (nebo všechny, pokud jich potřebuje více), pokračuje v provádění hned za místem, kde volalo `wait()`. Z bezpečnostních důvodů musí být volání `wait()` vždy uzavřeno v cyklu `while`, nikoliv pouze v podmínce `if`.

* Důvodem jsou takzvaná falešná probuzení (spurious wakeups), kdy se vlákno $W může probudit bez explicitního volání `notify()`, nebo situace, kdy jiné vlákno stihne podmínku opět zneplatnit dříve, než se probuzené vlákno dostane k procesoru (například prvek z zásobníku našemu vláknu "vyfoukne pod nosem").
* Vlákno $W tedy znovu zkontroluje podmínku a pokud stále není splněna, opětovně usne. Pokud splněna je, pokračuje v práci.

## Příklad

Uvažujme příklad, kdy produkční vlákna provádějí nějaké výpisy do bufferu. Druhé vlákno čeká na nashromáždění určitého minimálního počtu výpisů a teprve ve skupince je zobrazí uživateli najednou.

TODO



### Schéma

Následující schémata ukazují sekvenci operací, jak se provádějí za sebou.



