Јасно, ево детаљног водича за `useCallback` у React/React Native — шта ради, када га користити, које су предности и мане, како утиче на меморију и где може да доведе до „memory leak“-а (индиректно).

# Шта је `useCallback`

- `useCallback(fn, deps)` враћа исту **референцу функције** између рендера све док се зависности (`deps`) не промене.
    
- То **не убрзава аутоматски** код; користи се да:
    
    - стабилизујеш пропсе ка `React.memo` деци (да не ререндерују без потребе),
        
    - стабилизујеш dependency листе унутар `useEffect`/`useMemo` (да се не тригерирају сваки пут).
        

```tsx
const onPress = useCallback(() => {
  // ... handler логика
}, [someState, propA]);
```

---

# Када је КОРИСТАН

1. **Пропсујеш handler у `React.memo` компоненту**  
    Ако дете пореди пропсе по референци, стабилна функција спречава непотребан ререндер.
    
    ```tsx
    const onItemPress = useCallback((id: string) => { /* ... */ }, [navigate]);
    return <ItemRow onPress={onItemPress} />; // ItemRow = React.memo
    ```
    
2. **`FlatList`/велике листе**  
    Стабилан `renderItem`, `keyExtractor`, `getItemLayout`, `onEndReached` смањују посао приликом скроловања.
    
    ```tsx
    const renderItem = useCallback(({item}) => (
      <Row item={item} onPress={onItemPress} />
    ), [onItemPress]);
    ```
    
3. **Dependency стабилност у ефектима/мемоизацијама**  
    Кад је handler у `useEffect` зависностима, `useCallback` спречава перма-ретригере.
    
    ```tsx
    const onResize = useCallback(() => setDims(Dimensions.get('window')), []);
    useEffect(() => {
      const sub = Dimensions.addEventListener('change', onResize);
      return () => sub.remove();
    }, [onResize]);
    ```
    
4. **Пренос handler-а у дубља стабла или контексте**  
    Кад више слојева прослеђује исту функцију — мање garbage-а и мање rebind-овања.
    

---

# Када НИЈЕ потребан (или је штетан)

- **Дете није `React.memo`** → стабилност референце не доноси корист (дете се свакако ререндерује).
    
- **Handler је банално јефтин** и не користи се у зависностима/кешевима.
    
- **Прерана оптимизација**: `useCallback` има сопствени трошак (чување deps, поређење). Ако немаш мерљиве добитке (листe, мемоизована деца, скупе гране), често је сувишан.
    
- **Погрешне зависности**: ако не наведеш реалне deps, добићеш „stale closure“ (ради са застарелим вредностима).
    

---

# Понашање у меморији

- `useCallback` **чува једну референцу функције** док се deps не промене. То је мало.
    
- Индиректно може **смањити** алокације (мање нових функција при сваком ререндеру, мање ре-рендера деце → мање привремених објеката).
    
- Али може и **повећати** меморијски отисак ако га користиш:
    
    - масовно (нпр. креираш стотине различитих мемоизованих callback-а по item-у),
        
    - без стварне користи (дете није мемоизовано, нема зависности које би се иначе палиле).
        

### Практичан савет за листе

Уместо да правиш по један `useCallback` по реду са затварањем ID-а, боље направи **један стабилан handler** који прима ID као аргумент:

```tsx
const onItemPress = useCallback((id: string) => { /* ... */ }, [navigate]);

// у renderItem:
<Row onPress={() => onItemPress(item.id)} /> // овај inline јефтин је и не разбија React.memo Row ако Row мемоизује onPress преко useEvent (види ниже)
```

Или користи шаблон „stable handler + useEvent“ у детету.

---

# Типични шаблони у React Native

1. **Стабилан `renderItem` за `FlatList`**
    

```tsx
const renderItem = useCallback(({ item }) => (
  <ChatRow item={item} onPress={onPress} />
), [onPress]);

<FlatList
  data={messages}
  renderItem={renderItem}
  keyExtractor={useCallback((x) => x.id, [])}
/>
```

2. **Gesture/Keyboard/Dimensions слушаоци**
    

```tsx
const onKeyboardShow = useCallback(() => setShown(true), []);
useEffect(() => {
  const sub = Keyboard.addListener('keyboardDidShow', onKeyboardShow);
  return () => sub.remove();
}, [onKeyboardShow]);
```

3. **Навигација**
    

```tsx
const goDetails = useCallback(() => navigation.navigate('Details', { id }), [navigation, id]);
```

---

# Честе грешке

1. **Stale closure**  
    Заборавиш зависности → handler користи старо стање/props.
    
    ```tsx
    const onSave = useCallback(() => {
      // користи стар value ако [value] није у deps
      submit(value);
    }, []); // ❌
    ```
    
2. **Прекомпликоване deps**  
    У deps ставиш објекат који се мења сваког рендера (нпр. `{a:1}`), па се callback увек рекреира.
    
    - Реши тиме што стабилизујеш тај објекат (`useMemo`) или разложиш deps на примитиве.
        
3. **Мемоизација без добити**  
    Нема `React.memo` деце, нема ефеката који зависе од handler-а → чист терет.
    
4. **Ланчано мемоизовање**  
    Мемоизујеш _све_ handler-е на екрану „за сваки случај“. Обично су 1–3 довољна (они у листама, они у ефектима).
    

---

# Memory leaks — како се стварно дешавају (и превенција)

`useCallback` САМ по себи не прави leak; али **како користиш стабилан handler** може да гурне код у leak:

1. **Слушаоци/интервали без cleanup-а**  
    Ако стабилну функцију региструјеш на дуговечни извор (Keyboard, Dimensions, EventEmitter, WebSocket, setInterval) и **не уклониш** на unmount, извор држи референцу на callback → компонента и све што callback хвата остају у меморији.
    
    - ✅ Увек одради `remove()` / `clearInterval()` у cleanup-у.
        
2. **Callback хвата велике објекте**  
    Нпр. `useCallback(() => useHugeMap.current.doStuff(), [useHugeMap])` и тај handler остане регистрован негде дуго → држиш огроман граф у RAM-у.
    
    - ✅ Сузи захват: унутар handler-а извучеш/копираш само што треба, или чуваш лакшу референцу.
        
3. **Глобалне мапе/кешеви унутар handler-а**  
    Ако у callback-у додаш у глобални Map/Set и никад не уклањаш (нпр. по `id`) → расте без краја.
    
    - ✅ Уведи LRU/лимит/explicit delete у cleanup-у.
        

### Сигурни шаблони

- **Ефекти увек чисти:**
    
    ```tsx
    useEffect(() => {
      const sub = emitter.addListener('evt', handler); // handler је useCallback
      return () => sub.remove();
    }, [handler]);
    ```
    
- **Избегни stale closure без leak-а:** користи `useRef`/„useEvent“ образац за „увек најновљи“ handler.
    
    ```tsx
    // прост useEvent:
    function useEvent<T extends (...a:any)=>any>(fn: T): T {
      const ref = React.useRef(fn);
      React.useEffect(() => { ref.current = fn; });
      return React.useCallback(((...args:any[]) => ref.current(...args)) as T, []);
    }
    
    const onPress = useEvent((id: string) => submit(id, state)); // нема deps, нема stale
    ```
    

---

# Предности и мане (сажето)

**Предности**

- Стабилизује референце → мање непотребних ререндерa у `React.memo` деци.
    
- Стабилност deps у `useEffect`/`useMemo`.
    
- Мање привремених алокација функција у честим ререндерима.
    
- Корисно у листама (`FlatList`) и за слушаоце система.
    

**Мане**

- Има трошак (deps поређење, bookkeeping) — ако нема мерљиве користи, губитак.
    
- Лако направиш „stale closure“ ако заборавиш зависности.
    
- Масовна употреба (много instance-ова) може повећати сложеност и RAM без добитка.
    
- Не решава проблеме анимација/радника/IO — није замена за `useRef`, `useMemo`, `useEffect`, `useSharedValue`.
    

---

# Брзи избор: `useCallback` vs `useMemo` vs `useRef`

- **Стабилан handler функције?** → `useCallback`.
    
- **Стабилан објекат/низ/дериват?** → `useMemo`.
    
- **Мутабилна кутија која не тригерира ререндер (ID, флаг, ref на нативну компоненту)?** → `useRef`.
    

---

# TL;DR

- Користи `useCallback` када стабилност функције **стварно доноси корист** (мемоизована деца, листе, ефекти).
    
- Не мемоизуј све редом; мери добит.
    
- Избегни `stale closure`: тачне зависности или користи „`useEvent`“ шаблон.
    
- Leaks настају кад стабилан handler остане регистрован без cleanup-а или хвата огромне структуре — увек чисти слушаоце/интервале и сужавај захват.
    

Ако хоћеш, пошаљи део твог RN кода (листe, gesture, keyboard, навигација), па ћу ти конкретно показати где `useCallback` помаже и где је вишак.

Супер, ево практичних мини-задатака (RN) за `useRef / useMemo / useCallback` са решењима и објашњењем — баш фокус на меморију, перформансе и избегавање leak-ова.

# Zadaci

## 1) FlatList re-render i skok memorije

**Problem:** Lista štuca pri skrolu. Svaki render pravi nove propse i funkcije.

**Start kod (loše):**

```tsx
const Screen = ({ data }) => {
  const renderItem = ({ item }) => <Row item={item} onPress={() => console.log(item.id)} />;
  return (
    <FlatList
      data={data}
      renderItem={renderItem}
      keyExtractor={(x) => x.id}
    />
  );
};
```

**Rešenje (stabilizuj reference):**

```tsx
const Screen = ({ data, navigation }) => {
  const onItemPress = useCallback((id: string) => {
    navigation.navigate('Details', { id });
  }, [navigation]);

  const renderItem = useCallback(({ item }) => (
    <Row item={item} onPress={() => onItemPress(item.id)} />
  ), [onItemPress]);

  const keyExtractor = useCallback((x: {id: string}) => x.id, []);

  return <FlatList data={data} renderItem={renderItem} keyExtractor={keyExtractor} />;
};
```

**Zašto:** `useCallback` čuva stabilne funkcije → manje nepotrebnih re-rendera i alokacija.

---

## 2) „Stale closure“ i interval koji curi

**Problem:** Timer koristi staru vrednost `count`; zaboravljen cleanup → leak.

**Start kod (loše):**

```tsx
const C = () => {
  const [count, setCount] = useState(0);
  useEffect(() => {
    const id = setInterval(() => setCount(count + 1), 1000); // stale!
  }, []); // bez cleanup-a
  return <Text>{count}</Text>;
};
```

**Rešenje A (funkcionalni setState + cleanup):**

```tsx
useEffect(() => {
  const id = setInterval(() => setCount(c => c + 1), 1000);
  return () => clearInterval(id);
}, []);
```

**Rešenje B (anti-stale sa ref):**

```tsx
const latest = useRef(0);
useEffect(() => { latest.current = count; }, [count]);

useEffect(() => {
  const id = setInterval(() => setCount(latest.current + 1), 1000);
  return () => clearInterval(id);
}, []);
```

**Zašto:** Cleanup sprečava leak; ref rešava zastarelu vrednost bez re-rendera.

---

## 3) Teško filtriranje + pogrešan `useMemo`

**Problem:** Skupo filtriranje radi _svaki_ render; ili je `useMemo` sa praznim deps (drži stare i potencijalno velike strukture).

**Start (loše):**

```tsx
const filtered = useMemo(() => heavyFilter(items, query), []); // ❌ zavisi od items/query!
```

**Rešenje:**

```tsx
const filtered = useMemo(() => heavyFilter(items, query), [items, query]);
```

**Saveti za memoriju:**

- Ako `heavyFilter` vraća ogroman objekat, vrati **light view-model** (id, title), ne ceo objekat.
    

---

## 4) Keyboard listener bez cleanup-a

**Problem:** Stabilan handler registruješ, ali ne skidaš listener → RN zadrži referencu (leak).

**Start (loše):**

```tsx
const onShow = useCallback(() => setOpen(true), []);
useEffect(() => {
  Keyboard.addListener('keyboardDidShow', onShow);
}, [onShow]); // nema remove
```

**Rešenje:**

```tsx
useEffect(() => {
  const sub = Keyboard.addListener('keyboardDidShow', onShow);
  return () => sub.remove();
}, [onShow]);
```

**Zašto:** Uvek cleanup za event emitere (Keyboard, Dimensions, AppState, NetInfo, custom emitters).

---

## 5) Čuvanje velikog bafera u ref-u

**Problem:** U `ref.current` držiš veliki `Uint8Array` koji preživi ekran.

**Start (loše):**

```tsx
const bufRef = useRef<Uint8Array | null>(null);
useEffect(() => {
  bufRef.current = new Uint8Array(1024 * 1024 * 5); // 5MB
}, []);
```

**Rešenje (lifecycle kontrola):**

```tsx
const bufRef = useRef<Uint8Array | null>(null);
useEffect(() => {
  bufRef.current = new Uint8Array(1024 * 1024 * 5);
  return () => { bufRef.current = null; }; // eksplicitno „pusti“ referencu
}, []);
```

**Zašto:** Bez postavljanja na `null`, GC može kasniti jer referenca živi u komponenti do unmount-a.

---

# Mini cheatsheet (memorija & leaksi)

- **useRef**
    
    - ✅ Drži mutable vrednost bez re-rendera; idealno za ID-eve, flagove, native ref-ove.
        
    - ⚠️ Ne čuvaj ogromne objekte bez cleanup-a (`ref.current = null` na unmount).
        
- **useMemo**
    
    - ✅ Keširaj _skupa_ izračunavanja i stabilizuj objekte za props/deps.
        
    - ⚠️ Tačni deps! Prazan `[]` kad zavisiš od nečeg → zastareli podaci i zadržane reference.
        
    - ⚠️ Ne memoizuj trivijalnosti; pazi da rezultat ne bude prevelik.
        
- **useCallback**
    
    - ✅ Stabilne funkcije za `React.memo` decu, `FlatList`, i deps u efektima.
        
    - ⚠️ Ne registruj long-lived listenere bez cleanup-a (leak).
        
    - ⚠️ Ne koristi ako nema merljive koristi (trošak bez dobitka).
        

---

# Debug & merenje (RN)

- **Flipper → React DevTools & Performance**: vidi re-rendere (highlight updates).
    
- **Why Did You Render** (dev-only): otkrij nepotrebne re-rendere.
    
- **Xcode/Android Studio Memory profiler**: posmatraj heap dok mount/unmount ekrana.
    
- **Logovi cleanup-a**: u `useEffect` cleanup dodaj `console.log('cleanup <Name>')` da potvrdiš skidanje resursa.
    

Ako želiš, pošalji mi deo tvog ekrana (naročito `FlatList` sa item komponentom i handler-ima) i optimizujem ga korak-po-korak uz objašnjenje šta tačno dobijaš na memoriji i renderima.