# False Sharing

### False Sharing

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

***

#### Jak se tomu v Javě bráníme?

Dříve se to řešilo šílenými hacky, kdy programátoři mezi dvě důležité proměnné vkládali 7 nevyužitých `long` proměnných (8 \* 7 = 56 bajtů), aby je od sebe "odsunuli".
