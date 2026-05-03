# Future

## 🧠 Základní pojmy

### 1. Future

* Rozhraní **Future**
* Reprezentuje **výsledek, který ještě není hotový**
* Umí:
  * zjistit, jestli už je hotovo (`isDone()`)
  * blokovat vlákno (`get()`)

👉 problém: neumí dobře řetězit operace (je „hloupý“)

***

### 2. CompletableFuture (to je to hlavní)

* Třída **CompletableFuture**
* Vylepšený Future → podobné jako:
  * C#: `Task`
  * JS: `Promise`

✔ umí:

* spustit async operaci
* řetězit operace (`thenApply`, `thenAccept`, ...)
* kombinovat více async úloh
* řešit chyby

***

## ⚙️ Základní použití

### 🔹 Spuštění async úlohy

```
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {    return "Hello";});
```

👉 `supplyAsync` = spustí na background threadu a vrátí výsledek

***

### 🔹 Získání výsledku (blokující)

```
String result = future.get(); // blokuje (jako .Result v C#)
```

***

### 🔹 Řetězení (neblokující)

```
future    .thenApply(s -> s + " World")    .thenAccept(System.out::println);
```

👉 ekvivalent:

```
await Task.Run(...).ContinueWith(...)
```

***

## 🔗 Typy operací

### thenApply (map)

```
.thenApply(x -> x + "!")
```

➡ transformuje výsledek

***

### thenAccept (consume)

```
.thenAccept(x -> System.out.println(x));
```

➡ nic nevrací

***

### thenRun

```
.thenRun(() -> System.out.println("Done"));
```

➡ nezajímá ho výsledek

***

## ⚡ Paralelní operace

### Více úloh najednou

```
CompletableFuture<String> f1 = CompletableFuture.supplyAsync(() -> "A");CompletableFuture<String> f2 = CompletableFuture.supplyAsync(() -> "B");CompletableFuture<Void> all = CompletableFuture.allOf(f1, f2);all.thenRun(() -> {    try {        System.out.println(f1.get() + f2.get());    } catch (Exception e) {}});
```

***

## ❗ Výjimky

```
future    .thenApply(x -> x.toUpperCase())    .exceptionally(ex -> {        return "ERROR";    });
```
