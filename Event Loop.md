Odlično pitanje, Vukašine 👌 – ovo je srž **JavaScript event loop-a** i pomaže da razumeš kako se kod zaista izvršava u memoriji.

Idemo redom:

---

# 1. Memorija i Event Loop kontekst

JavaScript (u browseru ili Node.js) radi **single-threaded** – postoji **call stack** (stog funkcija) i **heap** (memorija za objekte).  
Ali asinhroni kod (npr. `setTimeout`, `fetch`, `Promise`) ne radi odmah na stacku. On se zakazuje u tzv. **task queues** (redove zadataka), i event loop odlučuje _kada_ ih vraća na stack.

---

# 2. Mikro-zadaci (Microtasks)

- Tipično: **`Promise.then`**, **`async/await` nastavak**, **MutationObserver**.
    
- Izvršavaju se **odmah posle trenutnog stacka**, pre nego što browser pređe na sledeći makro-zadatak.
    
- Brži prioritet → uvek se isprazni ceo microtask red pre sledećeg makro-task-a.
    

Primer:

```js
console.log("A");

Promise.resolve().then(() => console.log("Microtask 1"));

console.log("B");

Promise.resolve().then(() => console.log("Microtask 2"));

console.log("C");
```

**Redosled:**

```
A
B
C
Microtask 1
Microtask 2
```

---

# 3. Makro-zadaci (Macrotasks)

- Tipično: **`setTimeout`**, **`setInterval`**, **I/O događaji**, **requestAnimationFrame**.
    
- Izvršavaju se jedan po jedan – svaki ide u red čekanja, i event loop bira sledeći kad se završe stack i microtasks.
    

Primer:

```js
console.log("A");

setTimeout(() => console.log("Macrotask"), 0);

console.log("B");
```

**Redosled:**

```
A
B
Macrotask
```

---

# 4. Kombinacija mikro + makro

```js
console.log("1");

setTimeout(() => console.log("2 (macrotask)"), 0);

Promise.resolve().then(() => console.log("3 (microtask)"));

console.log("4");
```

**Redosled:**

```
1
4
3 (microtask)
2 (macrotask)
```

👉 Zašto?

1. `console.log("1")` → odmah
    
2. `setTimeout` → zakazan kao **macrotask**
    
3. `Promise.then` → zakazan kao **microtask**
    
4. `console.log("4")` → odmah
    
5. Event loop: stack prazan → isprazni **microtasks** → onda tek uzmi **macrotask**.
    

---

# 5. Async/Await u kontekstu micro/macro

```js
async function demo() {
  console.log("X");
  await null;
  console.log("Y");
}

demo();
console.log("Z");
```

**Redosled:**

```
X
Z
Y
```

- `await` razbije funkciju: sve posle njega ide u **microtask**.
    
- Zato `Y` ide posle sinhronog `Z`.
    

---

# 6. Greške i zamke

1. **Misliti da je `setTimeout(..., 0)` odmah**  
    ❌ Nije – čeka da se završe svi microtasks i tek onda dođe na red.  
    ✅ Koristi `queueMicrotask(fn)` ako baš hoćeš da ubaciš u microtask red.
    
2. **Zaboraviti da se microtasks isprazne do kraja**  
    – Ako unutar microtaska zakazuješ nove microtaske, oni se dodaju odmah i izvršavaju pre makrotaskova.  
    To može da napravi “beskonačnu petlju microtaska”.
    
    ```js
    function loop() {
      console.log("ping");
      queueMicrotask(loop);
    }
    loop(); // zamrzne event loop – makrotaskovi nikad ne dođu na red
    ```
    
3. **Pretpostaviti da async/await blokira ceo thread**  
    ❌ Ne blokira. Samo kaže: “pauziraj ovu funkciju i nastavi je u microtasku kad promise reši”.
    

---

# 7. Vizuelna slika (pojednostavljena)

```
CALL STACK
   ↓
──────────────
|   Macrotask Queue    |  ← (setTimeout, setInterval, I/O)
──────────────
|   Microtask Queue    |  ← (Promise.then, await continuation)
──────────────
EVENT LOOP bira:
- ako je stack prazan → prvo microtask queue (dok se ne isprazni)
- tek onda macrotask queue
```

---

# 8. Gde ulazi memorija?

- **Stack** → kratkoročno, za aktivne funkcije. Prazni se kad funkcije završe.
    
- **Heap** → dugi objekti (npr. closure, DOM reference). Ostaju dok GC ne obriše.
    
- **Queues** (micro/macro) → reference na callback funkcije dok ne dođu na red. Ako ih ne obrišeš (npr. `setInterval` bez `clearInterval`), ostaju zauzeti → memory leak.
    

---

✅ Ukratko:

- **Microtasks** = prioritetni, odmah posle stacka (Promise, await).
    
- **Macrotasks** = “sporiji”, redovni događaji (timeout, I/O).
    
- Event loop uvek isprazni **sve microtasks pre nego što krene sledeći makrotask**.
    
- Čišćenje memorije (GC) se dešava kad više nema referenci na objekte i callbacke u queue-u.
    

---

Hoćeš da ti napravim **živu simulaciju** (Node.js kod koji loguje redosled događaja micro/macro taskova sa više kombinacija)?