# Executor - ForkJoinPool

## 1. Executor vs ExecutorService vs ForkJoinPool

### 🔹 ExecutorService

To je „spravovaný thread pool“.

* umí spouštět tasky
* umí shutdown / awaitTermination
* typicky se používá pro vlastní pooly

```
ExecutorService es = Executors.newFixedThreadPool(10);
```

***

### 🔹 ForkJoinPool

To je defaultní „chytrý“ pool pro paralelní práci.

Používá ho:

* `CompletableFuture.supplyAsync()`
* `parallelStream()`

***

## ⚡ 2. Default pool: common ForkJoinPool

Když napíšeš:

```
CompletableFuture.supplyAsync(() -> ...)
```

👉 bez executoru

použije se:

```
ForkJoinPool.commonPool()
```

***

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
