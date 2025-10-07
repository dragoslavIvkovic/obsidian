Super pitanje, Vukašine. Evo **kompletnog pregleda upravljanja memorijom u React Native-u** — od modela memorije (JS ↔ native), tipičnih uzroka curenja, do konkretnih kod-šablona, podešavanja listi/slika i alata za profilisanje na iOS/Android.

---

# 1) Kako RN koristi memoriju (mentalni model)

React Native ima **dva sveta**:

1. **JS runtime** (Hermes ili JSC)
    
    - **JS heap**: objekti, closure-i, referencirani podaci.
        
    - **GC (garbage collector)** oslobađa objekte kad **nema više referenci**.
        
2. **Native (iOS/Android)**
    
    - UI čvorovi (view-ovi), slike/bitmapi, fajlovi, mrežni baferi, nativne pretplate.
        
    - Oslobađanje radi nativni GC/ARC (Android/iOS).
        
    - Ako JS drži referencu na nešto nativno (npr. EventEmitter, slika u Image komponenti, WebView), to **sprečava** oslobađanje.
        

**Zaključak:** da bi se nešto oslobodilo, moraš:  
a) prestati da držiš JS reference (set na `null`, cleanup u efektima), i  
b) ukloniti nativne resurse (ukloniti listenere, zatvoriti streamove, obrisati privremene fajlove).

---

# 2) Najčešći izvori curenja memorije (i kako ih sprečiti)

## A) Efekti bez cleanupa

- **Timers**: `setInterval`, `setTimeout`  
    ✅ Uvek čistiti u `return () => clearInterval(id)` / `clearTimeout(id)`
    
- **Event listenere**: `AppState`, `Keyboard`, `Dimensions`, `BackHandler`, nativni `NativeEventEmitter`  
    ✅ `const sub = addListener(...); return () => sub.remove();`
    
- **Pretplate / soketi**: WebSocket, BLE, geolokacija  
    ✅ `ws.close()`, `watchId && Geolocation.clearWatch(watchId)`
    

```tsx
useEffect(() => {
  const sub = AppState.addEventListener("change", onAppState);
  const id = setInterval(tick, 1000);
  return () => {
    sub.remove();
    clearInterval(id);
  };
}, [onAppState]);
```

## B) Navigacija: ekran nije unmount-ovan

- **Tab/Drawer** ekrani obično **ostaju mount-ovani** (samo gube fokus).  
    Ako želiš hard reset:
    
    ```tsx
    <Tab.Screen
      name="Feed"
      component={Feed}
      options={{ unmountOnBlur: true }}
    />
    ```
    
- U suprotnom, koristi `useFocusEffect` da **start/stop** resurse **po fokusu** (kamera, lokacija).
    

```tsx
import { useFocusEffect } from "@react-navigation/native";
useFocusEffect(
  useCallback(() => {
    const sub = startExpensiveSubscription();
    return () => sub.stop();
  }, [])
);
```

## C) Liste i renderovanje velikih kolekcija

- Ne držati **hiljade elemenata** u memoriji renderovanih odjednom.
    
- Koristi **FlatList/SectionList** (virtualizacija) i optimizuj:
    
    - `keyExtractor` (stabilan), `getItemLayout`, `initialNumToRender`,  
        `windowSize`, `maxToRenderPerBatch`, `removeClippedSubviews`
        
- Za velike feed-ove razmisli o **FlashList** (Shopify).
    

```tsx
<FlatList
  data={items}
  keyExtractor={(x) => x.id}
  renderItem={Row}
  initialNumToRender={12}
  maxToRenderPerBatch={12}
  windowSize={7}
  removeClippedSubviews
  getItemLayout={(data, index) => ({
    length: ROW_H,
    offset: ROW_H * index,
    index,
  })}
/>
```

## D) Slike (najčešći OOM na Androidu)

- Bitmapa ≈ `width * height * 4` bajta u RAM-u (RGBA).  
    **1024×1024 ≈ 4 MB**; 3000×3000 ≈ 34 MB po slici!
    
- Izbegavaj **prevelike** slike i **base64** u JS (drže sve u heap-u).
    
- Prednost: `uri` sa server-side resize-om (parametri za širinu/visinu), ili biblioteka sa keširanjem (npr. FastImage).
    
- Ukloni nevidljive slike iz prikaza (list virtualizacija) — RN automatski oslobodi kada GC i view tree dozvoli.
    

```tsx
<Image
  source={{ uri: `${CDN}/img?w=600&h=400&fit=cover&id=${id}` }}
  resizeMode="cover"
  style={{ width: 300, height: 200 }}
/>
```

## E) Privremeni fajlovi / blobovi

- Ako koristiš `react-native-fs` / `react-native-blob-util`:  
    ✅ obriši temp fajl posle upotrebe (`unlink`)  
    ✅ za blob/stream zatvori handle (API biblioteke često imaju `close()`)
    

```ts
import RNFS from "react-native-fs";
const path = RNFS.CachesDirectoryPath + "/tmp.bin";
await RNFS.writeFile(path, data, "base64");
try {
  // ... radi nešto
} finally {
  await RNFS.unlink(path).catch(() => {});
}
```

## F) “Zaglavljene” reference u globalima / singletonima

- Velike strukture u **globalnim** varijablama, modulskim keševima, Redux/Context store-u → ostaju žive “zauvek”.  
    ✅ Čuvaj **ID-eve**, ne kompletne objekte/slike; čisti keš povremeno.
    

## G) WebView, Reanimated, Gesture Handler

- **WebView**: uklanjati kada nije potreban; izbegavati nepotrebno učitavanje ogromnih stranica.
    
- **Reanimated**: proveri da li worklet drži reference ka velikim objektima; ukloni listenere na unmount.
    
- **Gesture**: ukloni `onHandlerStateChange` side-effekte; nema curenja ako nemaš spoljne pretplate.
    

---

# 3) Obrasci za bezbedan async rad (otkazivanje + cleanup)

## Fetch sa timeout-om i abortom

```ts
async function fetchJson(url: string, ms = 8000, signal?: AbortSignal) {
  const ctrl = new AbortController();
  const t = setTimeout(() => ctrl.abort(), ms);
  const combined = signal
    ? new AbortController()
    : null;

  if (combined && signal) {
    signal.addEventListener("abort", () => combined.abort());
  }
  try {
    const res = await fetch(url, { signal: combined?.signal ?? ctrl.signal });
    if (!res.ok) throw new Error(`HTTP ${res.status}`);
    return await res.json();
  } finally {
    clearTimeout(t);
  }
}
```

## NativeEventEmitter ispravan cleanup

```ts
import { NativeEventEmitter, NativeModules } from "react-native";
useEffect(() => {
  const emitter = new NativeEventEmitter(NativeModules.MyModule);
  const sub = emitter.addListener("event", onEvent);
  return () => sub.remove();
}, [onEvent]);
```

---

# 4) Optimizacije koje indirektno štede memoriju

- **Memoizacija**: `useMemo` / `useCallback` za velike izračune i stabilne props-e → manje rendera = manje kratkotrajnog smeća u heap-u.
    
- **Split contexta**: odvoji “state koji se često menja” od “stabilnih vrednosti” → manje re-rendera potrošača (manje alokacija).
    
- **Selektori umesto ogromnih objekata**: u Redux/Zustand radije čuvaj minimalne delove; derivacije izračunaj “on the fly”.
    
- **Batched Updates**: moderni RN/React već grupiše setState pozive; izbegavaj nepotrebne setState u petljama.
    

---

# 5) Profilisanje memorije (kako da _merno_ uhvatiš problem)

## iOS (Xcode Instruments)

- Pokreni app sa **Instruments → Allocations / Leaks**.
    
- Reprodukuj scenario (scroll kroz listu, otvori/zatvori ekrane).
    
- Traži: rast alokacija bez vraćanja, ponavljajuće objekte, velike bitmape.
    

## Android (Android Studio Profiler)

- **Memory Profiler**: prati Java/Kotlin i native alokacije.
    
- Snimi heap dump; traži bitmap alokacije i reference lance.
    

## Hermes / JS heap

- Uključi **Hermes**.
    
- Koristi **Flipper** (Hermes Debugger & Profiler) da vidiš JS heap i CPU sample.
    
- Posmatraj JS objekate koje raste “bez plafona” (npr. keš, globalni niz, base64 stringovi).
    

> Savet: testiraj **duge sesije** (10–30 min) sa navigacijom, upload/download, beskonačni list — memory leak se često vidi tek vremenom.

---

# 6) Brzi “checklist” pred produkciju

-  Svi `useEffect` imaju **cleanup** (tajmeri, listenere, pretplate).
    
-  Za ekrane koji ne treba da žive u pozadini → `unmountOnBlur: true` (tabovi).
    
-  **FlatList** pravilno podešen (`getItemLayout`, `windowSize`, `removeClippedSubviews`).
    
-  Slike: **resize sa servera/CDN-a**, izbegni **base64**, koristi keširanje; proveri realne dimenzije.
    
-  Privremeni fajlovi se **brišu** posle upotrebe.
    
-  Nema globalnih nizova/objekata koji rastu bez granice; uvedi **limite keša**.
    
-  Hermes + Flipper profilisanje urađeno na “realnim rutama”.
    
-  Test dugog skrolovanja i multiple navigacija (mount/unmount) bez rasta memorije.
    

---

# 7) Mini-primer: “težak” ekran, ali bez curenja

```tsx
function HeavyScreen() {
  const isFocused = useIsFocused();

  useFocusEffect(
    useCallback(() => {
      const socket = new WebSocket("wss://...");
      const sub = AppState.addEventListener("change", onAppState);
      const id = setInterval(pulse, 500);
      return () => {
        socket.close();
        sub.remove();
        clearInterval(id);
      };
    }, [])
  );

  useEffect(() => {
    if (!isFocused) return;
    const ctrl = new AbortController();
    (async () => {
      try {
        const json = await fetch("https://api/huge", { signal: ctrl.signal }).then(r => r.json());
        // setState minimalno: čuvaj ID-eve ili prerađene podatke, ne ceo payload
      } catch (e) {
        if ((e as any).name !== "AbortError") console.warn(e);
      }
    })();
    return () => ctrl.abort();
  }, [isFocused]);

  return <ListOrWhatever />;
}
```

---

# 8) Kada je “normalno” da memorija raste?

- **Kratkoročno** tokom navigacije/rendera (alokacije, slike, GC dođe kasnije).
    
- Ako se posle iste akcije **mem ne vrati blizu početne**, i svaka iteracija diže “baznu liniju”, to je znak curenja.
    

---

Ako želiš, mogu da ti dam **check-script** zadatke (npr. test scroll + navigacija 10 minuta, logovanje memorije, heap dump analiza) i/ili **šablon Flipper sesije** sa koracima koje prolaziš da bi izolovao izvor curenja.