Evo kratkog, praktičnog pregleda za React Native (RN), na srpskom:

# Kada se komponenta mount-uje, update-uje, unmount-uje

- **Mount**: prvi put ulazi u React stablo → poziva se `function Component(props)` i render; `useEffect(..., [])` se pokreće **posle** prvog rendera.
    
- **Update**: menjaju se `props` ili `state` → render ponovo; `useEffect` sa zavisnostima se re-triguje ako su se promenile.
    
- **Unmount**: komponenta izlazi iz React stabla → React poziva sve **cleanup** funkcije iz `useEffect` hook-ova; referenca na komponentu se uklanja i GC može da oslobodi memoriju.
    

# Kada “kod ostaje u memoriji”, a kada se briše

## Ostaje u memoriji (često nehotice)

Komponentin JS objekti ostaju živi **dok postoji referenca**:

- **Nedovršene stvari**: aktivni `setInterval`, `setTimeout`, `requestAnimationFrame`.
    
- **Event listeneri** bez uklanjanja: `AppState`, `Dimensions`, `Keyboard`, `Linking`, `NetInfo`, custom emitteri, `addListener` u navigaciji.
    
- **Abortable operacije**: `fetch` bez `AbortController.abort()`, websockets bez `close()`.
    
- **Rešenja za state van komponente**: Redux/MobX/Zustand ako u njima čuvate velike objekte ili držite reference na funkcije/closures komponente.
    
- **Reanimated/Animated**: aktivne animacije/Shared Values koje se ne zaustave.
    
- **Keševi**: slike (Image pipeline), query keševi (React Query), memorisani rezultati (`memoize-one`, `useMemo`) – ovo je namerno “zadržavanje”.
    

> Dok god postoji **bilo koja** živa referenca (global, subscription, pending timer/promise), GC **ne može** da oslobodi taj objekat.

## Briše se iz memorije

- Kada se komponenta **unmount-uje** i **nema više referenci** na njene objekte/closures → JS GC je oslobađa.
    
- Native UI (view hijerarhija) za tu komponentu se oslobađa od strane RN/Fabric-a; teksture/slike se puštaju kada više nema korisnika.
    

# Navigacija: da li se ekran unmount-uje?

Zavisi od navigatora/parametara:

- **Stack (React Navigation)**: ekran A → ekran B
    
    - A je obično **i dalje mount-ovan** (iza u steku) → `useEffect` sa `focus/blur` događajima koristi `useFocusEffect` da pauzira poslove.
        
    - A se unmount-uje tek kad ga **izbacite iz stacka** (npr. `navigation.pop()` preko korena).
        
- **Tab navigator**: neaktivan tab je obično **mount-ovan** ali “blur-ovan”.
    
    - Možete kontrolisati: `unmountOnBlur: true` (tab se gasi kad napusti fokus), `freezeOnBlur: true` (čuva memoriju, ali ne re-renderuje).
        
- **Drawer**: slično kao Tab – najčešće ostaje mount-ovan.
    
- **Opcije performansi**: `detachInactiveScreens` (kod stacka) može da ukloni neaktivne ekrane sa native strane, ali JS instanca često ostaje dok je u navigatoru.
    

# FlatList i reciklaža

- Ćelije se **recilkuju** (native) → podkomponente se ne moraju svaki put unmount-ovati.
    
- Ako vam treba tvrdo upravljanje, koristite `keyExtractor`, `getItemLayout`, `recycle`, i pazite na `removeClippedSubviews`.
    

# Pravila za čist unmount (anti-leak checklista)

U **svakom** `useEffect` koji kreira spoljnu vezu napravite cleanup:

```tsx
useEffect(() => {
  const sub1 = AppState.addEventListener('change', onAppStateChange);
  const sub2 = Dimensions.addEventListener('change', onLayoutChange);

  const id = setInterval(tick, 1000);

  const controller = new AbortController();
  fetch(url, { signal: controller.signal }).catch(() => {});

  return () => {
    sub1.remove?.(); // novi API
    sub2.remove?.();
    clearInterval(id);
    controller.abort();
  };
}, []);
```

**Navigacija (focus/blur):**

```tsx
import { useFocusEffect } from '@react-navigation/native';
import { useCallback } from 'react';

useFocusEffect(
  useCallback(() => {
    startLiveUpdates();

    return () => {
      stopLiveUpdates();
    };
  }, [])
);
```

**WebSocket / EventSource:**

```tsx
useEffect(() => {
  const ws = new WebSocket(WS_URL);
  ws.onmessage = onMsg;

  return () => {
    ws.close();
  };
}, []);
```

**Animated / Reanimated:**

- `Animated.timing(...).start(({ finished }) => { ... })` → čuvajte referencu i zovite `stop()`.
    
- Reanimated 2/3: za **worklet-e** koristite `cancelAnimation(sharedValue)` u cleanupu.
    

# Gde se “pamti” i posle unmount-a (namerno)

- `useRef` (u toj komponenti) nestaje posle unmount-a, **osim ako** referencu na ref držite negde globalno.
    
- Globalni store (Redux/Zustand), React Query cache, i image cache **namerno** žive duže od komponente.
    
- `useMemo`/`useCallback` zadrže vrednosti dok je komponenta mount-ovana; ne “curi” samo po sebi.
    

# Kako da proveriš curenje memorije

- **React DevTools Profiler**: vidi mount/unmount cikluse.
    
- **Hermes inspector (JS)**: prati heap snapshots (Android/iOS dev build).
    
- **Xcode/Android Studio**: Instruments/Memory Profiler za native deo.
    
- Simptomi: rast heap-a posle navigacije “tamo-vamo”, neoslobodjeni `Image`s, neprekidni CPU kad je ekran iza fokusa.
    

# Brzi odgovori na tvoja pitanja

- **Kada ostaje u memoriji?**  
    Kada postoji **aktivna referenca**: timers, listeneri, websockets, global store, keševi ili kada ekran ostaje u navigatoru (stack/tab) i nije unmount-ovan.
    
- **Kada se briše?**  
    Kad se komponenta **unmount-uje** **i** nema više referenci na njene objekte → GC oslobađa; native view se uklanja.
    
- **Kada je unmount?**  
    Kad React ukloni čvor iz stabla (npr. ekran je zatvoren/izbačen iz stacka, conditional render `show && <Comp/>` postane `false`).
    
- **Kada je mount?**  
    Prvo pojavljivanje u stablu (prvi render); efekti sa `[]` se izvršavaju nakon rendera.
    

Ako želiš, mogu da pogledam konkretnu navigacionu konfiguraciju (Stack/Tab/Drawer) i predložim optimalne opcije (`unmountOnBlur`, `freezeOnBlur`, `detachInactiveScreens`) i hook-ove za cleanup za tvoj slučaj.