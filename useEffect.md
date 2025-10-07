Odlično pitanje! Ajde da razbijemo temu **`useEffect` hook-a** i kako React Native (i React uopšte) rukuje **uništavanjem (unmounting)** i **ostankom komponenti u memoriji**.

---

## 1. Šta radi `useEffect`

`useEffect` je hook koji služi da:

- reaguješ na promene (npr. kad se neki **state** ili **props** promeni),
    
- pokrećeš **sporedne efekte** (fetch podaci, animacije, event listeneri, tajmeri…),
    
- očistiš resurse kada komponenta **nestane** ili se efekat ponovo izvrši.
    

Osnovna forma:

```tsx
useEffect(() => {
  // efekat
  console.log("Komponenta montirana ili zavisnosti promenjene");

  return () => {
    // cleanup
    console.log("Komponenta uništena ili efekat osvežen");
  };
}, [dependencies]);
```

---

## 2. Kada se efekat izvršava

- **Bez deps array-a** → svaki render:
    
    ```tsx
    useEffect(() => { ... });
    ```
    
- **Prazan array `[]`** → samo jednom, kad se komponenta montira (i cleanup kad se uništi):
    
    ```tsx
    useEffect(() => { ... }, []);
    ```
    
- **Sa deps `[x, y]`** → izvršava se kad `x` ili `y` promene vrednost, i cleanup pre ponovnog pokretanja:
    
    ```tsx
    useEffect(() => { ... }, [x, y]);
    ```
    

---

## 3. Uništavanje (unmount) u React Native

Kada komponenta **nestane sa ekrana**, React:

1. **Pokrene cleanup** za svaki `useEffect` koji ga ima (`return () => {}` deo).
    
2. **Ukloni DOM/native view čvor** iz memorije (ako više nije nigde u hijerarhiji).
    
3. Ako je komponenta i dalje negde mount-ovana (npr. u stack navigatoru, ali van fokusa), ostaje u memoriji dok je React Navigation ne unmount-uje.
    

Primer:

```tsx
useEffect(() => {
  const id = setInterval(() => console.log("tick"), 1000);

  return () => {
    // očisti interval kada komponenta nestane
    clearInterval(id);
    console.log("Komponenta uništena, interval obrisan");
  };
}, []);
```

Ako zaboraviš **cleanup**, interval/pretplata ostaje živ → memory leak (posebno bitno u React Native aplikacijama gde prelaziš kroz ekrane).

---

## 4. Kada komponenta ostaje u memoriji

U React Native, **navigacija** najčešće diktira životni ciklus:

- **React Navigation `Stack.Navigator`**  
    Kada pređeš na novi screen, stari se **uništava** (defaultno). Dakle cleanup radi i memorija se oslobađa.
    
- **React Navigation `Tab.Navigator`**  
    Tabovi se obično **ne uništavaju**, već ostaju mount-ovani (samo gube fokus).  
    → Komponenta ostaje u memoriji, `useEffect` cleanup se **ne poziva** sve dok tab zaista ne nestane.
    
- **Conditional rendering (`{show && <Comp/>}`)**  
    Kada `show` postane `false`, `Comp` se unmountuje i cleanup se pokreće.
    

---

## 5. Tipični slučajevi sa `useEffect` u React Native

1. **Event listeneri (Keyboard, Dimensions, AppState)**
    
    ```tsx
    useEffect(() => {
      const sub = Keyboard.addListener("keyboardDidShow", () => console.log("Up"));
    
      return () => sub.remove(); // cleanup
    }, []);
    ```
    
2. **WebSocket/subscription**
    
    ```tsx
    useEffect(() => {
      const ws = new WebSocket("wss://...");
      ws.onmessage = (msg) => console.log(msg);
    
      return () => ws.close(); // cleanup kad screen nestane
    }, []);
    ```
    
3. **Location/permissions**  
    Moraš da očistiš watcher, jer će u suprotnom GPS raditi i posle unmount-a.
    

---

## 6. Kako testirati da li komponenta ostaje u memoriji

- Stavi `console.log("mount")` i `console.log("unmount")` u `useEffect`.
    
- Menjaj ekrane/tabove u React Native app.
    
- Ako vidiš **“unmount”** → komponenta je uništena.
    
- Ako ga nema → komponenta i dalje postoji u memoriji.
    

---

✅ **Za pamćenje:**

- `useEffect` cleanup uvek radi **pre nego što se komponenta uništi** ili **pre nego što se efekat ponovo pokrene**.
    
- U **Stack navigatoru** → ekrani se uništavaju kad odeš dalje.
    
- U **Tab navigatoru** → ekrani obično ostaju mount-ovani dok se aplikacija ne zatvori ili ne izrenderuje drugačije.
    
- Ako ne očistiš tajmere/listenere/subscription-e → curenje memorije i bugovi.
    

---
Super tema! U RN svetu “focus” uglavnom znači **fokus ekrana u React Navigation-u**. Najčešće koristiš:

- `useFocusEffect` i `useIsFocused` iz **@react-navigation/native**
    
- “običan” `useEffect` iz **react**
    

Ispod je detaljno: šta je šta, kada se šta poziva, razlike, tipične greške, i tačno **kada se ekran unmount-uje**.

---

# Šta je fokus ekrana

U React Navigation-u, ekran je:

- **focused** (vidljiv, aktivan, na vrhu navigatora)
    
- **blurred** (nije na ekranu, ali je i dalje **mount-ovan**)
    
- **unmounted** (skroz uklonjen iz stabla → cleanup svih efekata)
    

> Bitno: U **Stack** navigatoru prethodni ekran obično **ostaje mount-ovan** (samo je blurred) kad otvoriš novi preko njega. Unmount se dešava tek kad ga **popuješ** sa stacka (Back) ili ga **zameniš** (`replace`) ili ga ukloniš uslovnim renderovanjem. U **Tab**/Drawer navigatoru ekrani se tipično **ne unmount-uju** pri prebacivanju tabova, osim ako eksplicitno uključiš `unmountOnBlur: true`.

---

# `useFocusEffect` (React Navigation)

Radi kao “specijalizovani `useEffect`” koji se:

- pokreće **kad ekran dobije fokus**
    
- radi **cleanup kad ekran izgubi fokus** _ili_ kad se unmount-uje
    

```tsx
import { useFocusEffect } from "@react-navigation/native";
import { useCallback } from "react";

useFocusEffect(
  useCallback(() => {
    // start: npr. pretplata, kamera, tajmer, AppState listener
    const id = setInterval(() => console.log("tick while focused"), 1000);

    return () => {
      // stop: isto kao cleanup u useEffect
      clearInterval(id);
      console.log("cleanup on blur or unmount");
    };
  }, [])
);
```

Zašto **`useCallback`**?  
Callback mora da bude **stabilan po referenci**. Ako se na svakom renderu menja, `useFocusEffect` će tretirati to kao “novi efekat” i opet raditi cleanup/start → nepotreban rad i bagovi.

---

# `useIsFocused` (React Navigation)

Vraća `boolean` — da li je ekran trenutno u fokusu. Koristiš ga da usloviš ponašanje u kombinaciji sa `useEffect` ili direktno u JSX.

```tsx
import { useIsFocused } from "@react-navigation/native";
import { useEffect } from "react";

const isFocused = useIsFocused();

useEffect(() => {
  if (!isFocused) return; // radi samo kad je fokusiran

  let active = true;
  (async () => {
    const data = await fetchData();
    if (active) setState(data); // izbegni setState nakon blur/unmount
  })();

  return () => { active = false; }; // cleanup ako izgubi fokus/unmount
}, [isFocused]);
```

Kada birati `useIsFocused`?

- Kad želiš finiju kontrolu (npr. kombinovati više uslova, više efekata), ili želiš da **ne** pokrećeš ništa na blur nego samo kad je fokusiran.
    

---

# `useEffect` (React)

`useEffect` se vezuje za **mount/unmount** i **promene zavisnosti**, ali _ne zna_ za fokus ekrana.

- `[]` → jednom na **mount**, cleanup na **unmount**
    
- `[deps]` → kad se `deps` promene, cleanup starog pa izvršenje novog
    
- bez `[]` → svaki render
    

Ako ti je zaista bitno “pokreni kad je ekran vidljiv, pauziraj kad nije” → `useFocusEffect` je tačniji alat od čistog `useEffect`.

---

# Razlike ukratko

|Tema|`useEffect`|`useFocusEffect`|`useIsFocused` + `useEffect`|
|---|---|---|---|
|Svest o fokusu|❌|✅ automatski|✅ ručno proveravaš|
|Start/Stop na focus/blur|❌|✅|✅ (uslovno)|
|Cleanup na unmount|✅|✅|✅|
|Potreba za `useCallback`|—|**Da** (stabilnost)|Ne (ali pazi deps)|
|Tipične upotrebe|inicijalni fetch, globalno stanje|kamera, geolokacija, pretplate koje troše resurse|fetch samo kad je ekran vidljiv, revalidacija na povratku|

---

# Kada se ekran **unmount-uje**

- **Stack**:
    
    - `navigation.pop()` ili back gest → **unmount** top ekrana
        
    - `navigation.replace()` → unmountuje aktuelni, mountuje novi
        
    - Ako uslovno renderuješ ekran (`{show && <Screen/>}`) i `show` postane `false` → unmount
        
- **Tab/Drawer**:
    
    - Prebacivanje taba = **blur**, ne unmount (osim ako `options={{ unmountOnBlur: true }}`)
        
    - Zatvaranje parent navigatora ili uklanjanje rute → unmount
        
- **React Strict Mode (dev)**:
    
    - Efekti se mogu **pokrenuti dva puta na mount** (samo u developmentu) da uhvate impure efekte. Cleanup mora biti ispravan.
        

---

# Najčešće greške (i kako ih izbeći)

1. **Zaboravljen cleanup** (tajmeri, event listeneri, WebSocket, kamera)  
    → curenje memorije / neželjeni pozivi.
    
    - **Rešenje:** Uvek vratiti funkciju iz `useEffect`/`useFocusEffect` koja prekida resurse.
        
2. **Nedostatak `useCallback` oko `useFocusEffect`**  
    → callback menja referencu na svaki render → konstantno “blur → focus” ciklusi.
    
    - **Rešenje:** Uvij u `useCallback` sa odgovarajućim deps.
        
3. **Async direktno kao efekt funkcija**
    
    - `useFocusEffect(async () => { ... })` → efekat očekuje sync funkciju (ili cleanup), ne Promise.
        
    - **Rešenje:** Definiši async unutra:
        
        ```tsx
        useFocusEffect(useCallback(() => {
          let active = true;
          (async () => { /* ... */ })();
          return () => { active = false; };
        }, []));
        ```
        
4. **Stale closure** (koristi zastarelu vrednost state-a)
    
    - **Rešenje:** uključi zavisnosti u `useCallback`/`useEffect` ili koristi funkcionalni `setState`.
        
5. **Prečesto pokretanje zbog pogrešnih deps**
    
    - **Rešenje:** minimizuj deps; memoizuj handler-e (`useCallback`), vrednosti (`useMemo`).
        
6. **Oslanjanje na `useEffect` za fokus logiku**
    
    - Efekat se ne pauzira na blur → trošenje resursa u pozadini.
        
    - **Rešenje:** `useFocusEffect` ili `useIsFocused` guard.
        
7. **Strict Mode panika**
    
    - Vidite dupli log u devu i mislite da se sve “udvostručuje”.
        
    - **Rešenje:** To je očekivano u devu; bitno je da cleanup radi ispravno. U produkciji se ne duplira.
        

---

# Praktični šabloni

## 1) Start/stop resursa po fokusu (idealno za kameru, lokaciju)

```tsx
useFocusEffect(
  useCallback(() => {
    const subscription = startExpensiveResource(); // npr. Camera.start()

    return () => {
      subscription.stop?.();
    };
  }, [])
);
```

## 2) Refetch kad se vratiš na ekran

```tsx
const isFocused = useIsFocused();

useEffect(() => {
  if (!isFocused) return;
  let active = true;
  (async () => {
    const fresh = await api.get("/items");
    if (active) setItems(fresh);
  })();
  return () => { active = false; };
}, [isFocused]);
```

## 3) Event listener samo dok je ekran vidljiv

```tsx
useFocusEffect(
  useCallback(() => {
    const sub = AppState.addEventListener("change", handleAppState);
    return () => sub.remove();
  }, [handleAppState])
);
```

## 4) Abort fetch na blur/unmount (izbegni setState after unmount)

```tsx
useFocusEffect(
  useCallback(() => {
    const c = new AbortController();
    (async () => {
      try {
        const res = await fetch(url, { signal: c.signal });
        const data = await res.json();
        setData(data);
      } catch (e) {
        if (e.name !== "AbortError") console.warn(e);
      }
    })();
    return () => c.abort();
  }, [url])
);
```

## 5) `unmountOnBlur` kada baš želiš unmount

```tsx
<Tab.Screen
  name="Feed"
  component={FeedScreen}
  options={{ unmountOnBlur: true }} // svako napuštanje taba unmountuje ekran
/>
```

---

# Brzi “cheat-sheet”

- **Želim da nešto radi samo dok je ekran vidljiv:** `useFocusEffect` (+ `useCallback`)
    
- **Želim da refetch-ujem kad se vratim na ekran:** `useIsFocused` + `useEffect`
    
- **Želim hard reset ekrana pri svakoj promeni taba:** `options.unmountOnBlur = true`
    
- **Kada se unmount dešava u Stack-u:** na `pop`/`replace`/uklanjanje iz stabla (ne pri običnom `navigate` na viši ekran — tada je samo blur)
    
- **Uvek napiši cleanup** za tajmere, subscribe-ove, socket-e, kameru, lokaciju
    

Ako želiš, mogu da ti pripremim mini RN projekat sa Stack + Tab navigacijom i tri ekrana gde se u konzoli jasno vidi: **mount → focus → blur → unmount**, plus primere za fetch, tajmer i kameru.