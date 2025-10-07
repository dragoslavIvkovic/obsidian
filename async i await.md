Sjajno — idemo **duboko i praktično** u `async/await`. Sve u JavaScript/TypeScript stilu, bez obećanja “kasnije” — odmah dobijaš **osnove, obrasce, greške, čišćenje resursa (uništavanje), otkazivanje, konkurentnost** i gotove “recept” funkcije.

---

# 1) Brzi uvod (šta je `async` i `await`)

- `async function` **uvek vraća Promise**.
    
- `await` “pauzira” tu **jednu** async funkciju dok se Promise ne reši (ispod haube: raspakovanje u microtask fazi, ne blokira thread).
    

```ts
async function hello() {
  return "Ćao"; // implicitno: Promise.resolve("Ćao")
}

async function run() {
  const x = await hello();   // x = "Ćao"
  console.log(x);
}
run();
```

**Važno:** `await` (klasično) radi **samo unutar** `async` funkcije. Van nje — koristi **top-level await** u ESM modulima (`type: module` u Node-u ili `<script type="module">` u browseru).

---

# 2) Red izvršavanja (što često zbunjuje)

```ts
console.log("A");
(async () => {
  console.log("B");
  await null;
  console.log("C");
})();
console.log("D");
// Redosled: A, B, D, C  (C ide u microtask posle sinhronog dela)
```

---

# 3) Sekvencijalno vs paralelno

## (a) Sekvencijalno (sporije, ali jednostavno)

```ts
async function sequential() {
  const a = await fetch("/api/a");
  const b = await fetch("/api/b");
  return [a, b];
}
```

## (b) Paralelno (brže, preporučeno kad pozivi NISU međuzavisni)

```ts
async function parallel() {
  const [a, b] = await Promise.all([
    fetch("/api/a"),
    fetch("/api/b"),
  ]);
  return [a, b];
}
```

## (c) “Sve, ali ne pucaj na prvu grešku”

```ts
const results = await Promise.allSettled([
  fetch("/a"),
  fetch("/b"),
]);
// results[i] je { status: "fulfilled" | "rejected", value|reason }
```

---

# 4) Obrada grešaka (pravilno)

## Osnovno `try/catch/finally`

```ts
async function getJson(url: string) {
  try {
    const res = await fetch(url);
    if (!res.ok) {
      // fetch ne baca na 4xx/5xx — PROVERI ok
      throw new Error(`HTTP ${res.status}`);
    }
    return await res.json();
  } catch (err) {
    // log + rethrow sa cause (više konteksta)
    throw new Error(`Greška na ${url}`, { cause: err });
  } finally {
    // cleanup koji ZAUVEK radi (čak i kad se baci greška)
    // npr. zatvaranje file handle-a, brisanje temp fajla, otkazivanje tajmera…
  }
}
```

## Tanka razlika: `throw` vs `return Promise.reject(..)`

Koristi `throw` u `async` funkciji — čitljivije je.

---

# 5) “Uništavanje podataka” i **čišćenje resursa** (ključna praksa)

## Primer: privremeni fajl + `finally`

```ts
import { writeFile, unlink } from "node:fs/promises";

async function processTemp(content: string) {
  const path = "/tmp/data.txt";
  await writeFile(path, content);
  try {
    // ...radi nešto nad fajlom...
  } finally {
    // briši uvek — i na success i na grešku
    await unlink(path).catch(() => {}); // ignoriši ako već ne postoji
  }
}
```

## Primer: WebSocket/subscription

```ts
async function useSocket(url: string) {
  const ws = new WebSocket(url);
  try {
    // await dok ne dobiješ nešto, ili radi sa događajima…
  } finally {
    ws.close(); // garantovano zatvaranje
  }
}
```

---

# 6) Otkazivanje (cancellation) sa `AbortController`

## `fetch` + timeout + abort (browser/Node 18+)

```ts
async function fetchWithTimeout(url: string, ms = 5000) {
  const c = new AbortController();
  const t = setTimeout(() => c.abort(), ms);
  try {
    const res = await fetch(url, { signal: c.signal });
    if (!res.ok) throw new Error(`HTTP ${res.status}`);
    return await res.json();
  } finally {
    clearTimeout(t); // cleanup tajmera
  }
}
```

## Propagacija signala (komponovanje)

```ts
async function getUser(id: string, signal?: AbortSignal) {
  const res = await fetch(`/users/${id}`, { signal });
  return res.json();
}

async function getUserWithPosts(id: string, signal?: AbortSignal) {
  const controller = new AbortController();
  signal?.addEventListener("abort", () => controller.abort()); // prosledi abort

  const [user, posts] = await Promise.all([
    getUser(id, controller.signal),
    fetch(`/posts?user=${id}`, { signal: controller.signal }).then(r => r.json()),
  ]);
  return { user, posts };
}
```

---

# 7) `await` u petljama i kontrola konkurentnosti

## Anti-pattern: `await` u `for` kada želiš paralelno

```ts
// SPORO: ide jedan po jedan
for (const url of urls) {
  const data = await fetch(url).then(r => r.json());
}
```

## Ispravno paralelno

```ts
const results = await Promise.all(
  urls.map(u => fetch(u).then(r => r.json()))
);
```

## Ali pazi na “stampedo” — uvedi limit konkurentnosti

Minimalni “pool” bez biblioteka:

```ts
async function mapWithConcurrency<T, R>(
  items: T[],
  limit: number,
  worker: (item: T, i: number) => Promise<R>
): Promise<R[]> {
  const res: R[] = new Array(items.length);
  let i = 0;
  let active = 0;
  return new Promise((resolve, reject) => {
    const next = () => {
      if (i === items.length && active === 0) return resolve(res);
      while (active < limit && i < items.length) {
        const idx = i++;
        active++;
        worker(items[idx], idx)
          .then(v => { res[idx] = v; })
          .catch(reject)
          .finally(() => { active--; next(); });
      }
    };
    next();
  });
}

// korišćenje
const out = await mapWithConcurrency(urls, 5, async (u) => {
  const r = await fetch(u);
  return r.json();
});
```

---

# 8) `for await...of` i async generatori (sjajno za strim/paginaciju)

## Async generator: povlači stranice dok ih ima

```ts
async function* paginate(url: string) {
  let next = url;
  while (next) {
    const r = await fetch(next).then(x => x.json());
    yield r.items;
    next = r.next; // null kada nema dalje
  }
}

// potrošnja:
for await (const items of paginate("/api/products?page=1")) {
  // radi sa svakim “page” paketom
}
```

---

# 9) Top-level await (ESM)

- Radi u **ES modulima** (Node sa `type: "module"` ili `.mjs`; browser `<script type="module">`).
    
- Blokira inicijalizaciju modula dok se await ne završi (pazi na vreme učitavanja).
    

```ts
// main.mjs
const cfg = await fetch("/config.json").then(r => r.json());
console.log(cfg);
```

---

# 10) Rad sa callback API-jima (promisify)

## Ručno

```ts
function sleep(ms: number) {
  return new Promise<void>((res) => setTimeout(res, ms));
}

function readLegacy(cb: (err: Error|null, data?: string) => void) {
  setTimeout(() => cb(null, "ok"), 100);
}

function readLegacyAsync() {
  return new Promise<string>((res, rej) => {
    readLegacy((err, data) => err ? rej(err) : res(data!));
  });
}

const val = await readLegacyAsync();
```

## Node: `util.promisify`

```ts
import { promisify } from "node:util";
const readFileAsync = promisify(require("fs").readFile);
const data = await readFileAsync("file.txt", "utf8");
```

---

# 11) Retry sa eksponencijalnim backoff-om + jitter

```ts
type TryFn<T> = () => Promise<T>;

async function retry<T>(
  fn: TryFn<T>,
  { retries = 5, baseMs = 200, maxMs = 4000 } = {}
): Promise<T> {
  let attempt = 0;
  // koristimo jednu instancu AbortController-a spolja ako treba
  while (true) {
    try {
      return await fn();
    } catch (err) {
      attempt++;
      if (attempt > retries) throw err;
      const expo = Math.min(maxMs, baseMs * 2 ** (attempt - 1));
      const jitter = Math.random() * (expo * 0.2);
      const wait = expo + jitter;
      await new Promise(res => setTimeout(res, wait));
    }
  }
}
```

---

# 12) Testiranje async koda (primer sa Jest-om)

```ts
test("radi async bez done", async () => {
  const result = await Promise.resolve(42);
  expect(result).toBe(42);
});
```

> Nemoj koristiti `done` kada koristiš `async/await`. Ako koristiš `try/catch` u testu, **vrati/await-uj** promisu ili test “prođe” prerano.

---

# 13) Najčešće greške (i kako da ih izbegneš)

1. **Zaboravljen `await`**  
    – Dobiješ Promise umesto vrednosti; kasnije puca.  
    ✅ _Rešenje:_ ESLint rule `no-floating-promises` (TypeScript `@typescript-eslint/no-floating-promises`).
    
2. **`await` u petlji kada treba paralelno**  
    – Bespotrebno usporava.  
    ✅ _Rešenje:_ `Promise.all` ili kontrolisana konkurentnost (pool).
    
3. **Ne proveravaš `res.ok` kod `fetch`**  
    – 404/500 ne baca grešku.  
    ✅ _Rešenje:_ ručno baci grešku kad `!res.ok`.
    
4. **Zaboravljen cleanup u `finally`**  
    – Otvoreni handle-ovi, tajmeri → curenje resursa.  
    ✅ _Rešenje:_ uvek `finally`.
    
5. **Nepromenljiva referenca kod otkazivanja**  
    – Ne propagiraš `AbortSignal`.  
    ✅ _Rešenje:_ prosledi signal kroz slojeve.
    
6. **Mešanje `.then()` i `await` bez potrebe**  
    – Gubiš čitljivost i hvatanje grešaka.  
    ✅ _Rešenje:_ drži se `try/catch` obrazca.
    
7. **Top-level await zloupotreba**  
    – Usporava bootstrap modula.  
    ✅ _Rešenje:_ učitaj “teško” lenjo iza događaja (npr. posle user akcije) ili paralelizuj.
    
8. **`unhandledrejection`**  
    – Odbijen Promise bez catch-a.  
    ✅ _Rešenje:_ uvek `await` ili dodaj globalni handler (za log).
    

---

# 14) Gotovi “recepti” za kopiranje

## a) `timeout` wrapper

```ts
async function withTimeout<T>(p: Promise<T>, ms: number, msg = "Timeout"): Promise<T> {
  let t: NodeJS.Timeout;
  try {
    return await Promise.race([
      p,
      new Promise<T>((_, rej) => { t = setTimeout(() => rej(new Error(msg)), ms); })
    ]);
  } finally {
    clearTimeout(t!);
  }
}
```

## b) “Safe JSON” (osetljivi API-ji)

```ts
async function safeJson(res: Response) {
  const text = await res.text();
  try { return JSON.parse(text); }
  catch { throw new Error("Nevažeći JSON", { cause: text.slice(0, 300) }); }
}
```

## c) “Resource scope” (garantovano zatvaranje)

```ts
async function using<T>(open: () => Promise<{ close: () => Promise<void> }>,
                        work: (h: any) => Promise<T>) {
  const h = await open();
  try { return await work(h); }
  finally { await h.close(); }
}
```

---

# 15) Mini-zadaci (ako želiš da uvežbaš)

1. Napiši `fetchJson(url, { signal })` sa `ok` proverom + `AbortController` timeoutom.
    
2. Napravi `mapWithConcurrency` za 10 URL-ova (limit 3).
    
3. Implementiraj `retry` sa backoff-om i `AbortSignal` propagacijom.
    
4. Napiši async generator koji stranično čita `/items?page=n` i vrati sve ID-eve.
    

---

Ako hoćeš, mogu odmah da ti pripremim **mali repo** sa ovim utilima (TypeScript, ESM, Node 20) + par testova, ili da ih uklopim u tvoj RN/Next projekat. Samo reci šta ti je prioritet (timeout/abort, konkurentnost, retry, async generatori…).