Јасно — хоћеш „научи ме useRef у React Native-у: како се користи, како реагује на рендере и које грешке прави“. Ево ти практичан водич (све на RN примерима).

# Шта је `useRef`

- `useRef` прави **mutable „кутију”** облика `{ current: T }`.
    
- **Не покреће ре-рендер** кад мењаш `ref.current`.
    
- Вредност **преживљава између рендера** (слично као инстанс поља у класама).
    

```tsx
const ref = useRef<number>(0);
ref.current += 1; // не рендерује поново
```

# Кад га користиш

1. **Императиван приступ нативним компонентама**
    

- Фокусирање `TextInput`, скроловање `ScrollView`/`FlatList`, мерење, `setNativeProps`, итд.
    

```tsx
import { useRef } from "react";
import { TextInput, Button, View } from "react-native";

export default function Form() {
  const inputRef = useRef<TextInput>(null);

  return (
    <View>
      <TextInput ref={inputRef} placeholder="Email" />
      <Button
        title="Fokus"
        onPress={() => inputRef.current?.focus()}
      />
    </View>
  );
}
```

2. **Чување ID-ева и хендлера** (тајмери, интервали, отварачи модала…)
    

```tsx
const intervalRef = useRef<NodeJS.Timer | null>(null);

useEffect(() => {
  intervalRef.current = setInterval(() => {
    // ... посао
  }, 1000);
  return () => { if (intervalRef.current) clearInterval(intervalRef.current); };
}, []);
```

3. **Анти-статe променљиве** — кад ти треба стање које **не треба** да изазове ре-рендер (нпр. бројање кликова, кеш последње вредности, „isMounted” заставица).
    

```tsx
const isMounted = useRef(false);
useEffect(() => {
  isMounted.current = true;
  return () => { isMounted.current = false; };
}, []);
```

4. **Стабилан референтни идентитет** између рендера (алтернативно `useMemo`/`useCallback` у неким триковима).
    

---

# Како `useRef` реагује на рендере

- На **сваком ререндеру** `ref` објекат је **исти** (референца не мења идентитет).
    
- Промена `ref.current` **неће изазвати** нови рендер.
    
- Код **object ref** (`const r = useRef(null)`) React попуњава `r.current = node` кад се DOM/Native чвор повеже.
    
- Код **callback ref** (`node => { ... }`) функција се позива при mount/unmount/промени.
    

---

# `useRef` vs `useState`

|Карактеристика|useRef|useState|
|---|---|---|
|Тригери ре-рендер|❌|✅|
|Перзистира вредност|✅|✅|
|Користи се за UI мутације преко ref-а|✅|❌|
|Добар за derived/ephemeral податке|✅|❌ (боље мемоизација)|

Правило: ако UI треба да се **ажурира на екрану**, то је вероватно `useState`. Ако је интерно/императивно, `useRef`.

---

# Типични RN примери

## 1) Скрол до грешке у форми

```tsx
const listRef = useRef<FlatList<any>>(null);

const scrollToIndex = (i: number) => {
  listRef.current?.scrollToIndex({ index: i, animated: true });
};

<FlatList
  ref={listRef}
  data={fields}
  renderItem={...}
/>
```

## 2) Управљање `Animated`/`Reanimated`

- `react-native-reanimated`: користи `useSharedValue` за анимационе вредности; `useRef` је за класичне референце (нпр. `Animated.Value` у старом API-ју или реф на `ScrollView`).
    
- Не мешај `useRef` уместо `useSharedValue` — неће покренути анимациони „worklet”.
    

## 3) Мерење layout-а

- Преферирај `onLayout` уместо старих API-ја; резултат можеш чувати у `useRef` ако ти не треба ре-рендер.
    

```tsx
const sizeRef = useRef({ w: 0, h: 0 });

<View
  onLayout={(e) => {
    const { width, height } = e.nativeEvent.layout;
    sizeRef.current = { w: width, h: height }; // без ре-рендера
  }}
/>
```

---

# `forwardRef` и императивни хендл

Када правиш **сопствену компоненту** која излаже императивне методе (нпр. `.open()` модал):

```tsx
import React, { forwardRef, useImperativeHandle, useRef } from "react";
import { View, Modal, Button } from "react-native";

export type ModalHandle = { open: () => void; close: () => void };

const MyModal = forwardRef<ModalHandle>((props, ref) => {
  const modalVisible = useRef(false);

  useImperativeHandle(ref, () => ({
    open() { modalVisible.current = true; /* setState… */ },
    close() { modalVisible.current = false; /* setState… */ },
  }));

  return (
    <Modal visible={/* state from useState, not ref! */ false}>
      <View>{/* ... */}</View>
    </Modal>
  );
});

export default function Screen() {
  const modalRef = useRef<ModalHandle>(null);

  return (
    <View>
      <Button title="Open" onPress={() => modalRef.current?.open()} />
      <MyModal ref={modalRef} />
    </View>
  );
}
```

> Напомена: за **видљивост модала** користи `useState` (рендер мења UI). `useRef` само чува методе/флагове ако не мораш да рендерујеш.

---

# Најчешће грешке (и како да их избегнеш)

1. **Ослоњен си на `ref` за UI-стање**
    

- Грешка: мењаш `ref.current = true` и очекујеш да се дугме/модал појави.
    
- Решение: стање које утиче на UI → `useState`.
    

2. **Читаш `ref.current` пре mount-а**
    

- `ref.current` је `null` док се чвор не повеже.
    
- Увијек користи `?.` и провери `if (ref.current)`.
    

3. **Збуна око „стале” вредности у замци (closure)**
    

- Ако у `setInterval` користиш променљиве из старог рендера, добићеш „stale state”.
    
- Решение: или користи функционални `setState`, или чувај променљиву у `ref` коју ажурираш у `useEffect`.
    

```tsx
const countRef = useRef(0);
useEffect(() => {
  const id = setInterval(() => {
    countRef.current += 1; // увек текућа вредност
  }, 1000);
  return () => clearInterval(id);
}, []);
```

4. **Реасигнујеш сам `ref`**
    

- Грешка: `inputRef = useRef(null)` уместо `const inputRef = useRef(null)`.
    
- Увек `const` — мењаш само `inputRef.current`.
    

5. **Мешање `createRef` и `useRef`**
    

- `createRef` прави НОВИ објекат при сваком рендеру (добро за класе).
    
- У функционалним компонентама користи `useRef` (стабилан између рендера).
    

6. **Callback ref без стабилности**
    

- Ако дефинишеш `ref={(n) => {...}}` унутар JSX без мемоизације, може се звати чешће него што очекујеш.
    
- Обично је object-ref довољан (`useRef`). Ако баш треба callback ref, мемоизуј `useCallback`.
    

7. **Reanimated: погрешан алат**
    

- За анимационе вредности користи `useSharedValue`, не `useRef`. `useRef` не синхронизује UI тред.
    

---

# Мини „рецепти”

**Фокус на прво празно поље:**

```tsx
const emailRef = useRef<TextInput>(null);
const passRef = useRef<TextInput>(null);

<TextInput
  ref={emailRef}
  returnKeyType="next"
  onSubmitEditing={() => passRef.current?.focus()}
/>
<TextInput ref={passRef} secureTextEntry />
```

**Спречи двоклик:**

```tsx
const busyRef = useRef(false);
const onPress = () => {
  if (busyRef.current) return;
  busyRef.current = true;
  doAsync().finally(() => { busyRef.current = false; });
};
```

**Чување последње пропс функције (стабилан handler):**

```tsx
function useEvent<T extends (...args:any)=>any>(fn: T): T {
  const ref = useRef(fn);
  useEffect(() => { ref.current = fn; });
  return useRef(((...args:any[]) => ref.current(...args)) as T).current;
}
```

---

# Блиц провера (меморија & рендер)

- `useRef` = кутија која преживљава → **да**.
    
- Промена `ref.current` рендерује → **не**.
    
- Добро за фокус/скрол/тајмер/флаг → **да**.
    
- Добро за UI стање → **не** (узми `useState`).
    

Јасно — фокус на „меморију“ са `useRef` у React Native-у: како се понаша, када је користан, а када штетан.

# Како `useRef` утиче на меморију

- `useRef` прави стабилан објекат који живи колико и компонента. Сам по себи је **врло мали** (само „кутија“ са пољем `current`).
    
- Све што ставиш у `ref.current` остаје **у меморији** док:
    
    1. не промениш/обришеш `ref.current` (нпр. на `null`), или
        
    2. компонента не буде unmount → сам `ref` постаје недоступан и GC може да почисти.
        
- Ако нешто друго држи референцу на исту вредност (нпр. глобална променљива, затварање у дуготрајном тајмеру/листенеру), то може **спречити сакупљање смећа** и после unmount-а.
    

> Кључ: `ref.current` **не изазива ре-рендер**, али **задржава** објекат у меморији.

---

# Када је `useRef` користан за меморију

1. **Избегавање непотребних ре-рендера за велике/пролазне податке**  
    Краткотрајни бафери, последња вредност из мерења/layout-а, ID-еви тајмера, „busy” флаг.
    
    ```tsx
    const workBufRef = useRef<Uint8Array | null>(null);
    // користиш бафер у ефекту / хендлеру; не тераш UI да се рендерује
    useEffect(() => {
      workBufRef.current = new Uint8Array(1024 * 64);
      return () => { workBufRef.current = null; }; // ослободи
    }, []);
    ```
    
2. **Заштита од „stale closure” без копирања стања**  
    Чуваш најновију вредност без мемоизација које праве нове објекте.
    
    ```tsx
    const latestProps = useRef(props);
    useEffect(() => { latestProps.current = props; });
    ```
    
3. **Императивна контрола нативних ресурса** (фокус/скрол/мерач)  
    Реф ка `TextInput`, `FlatList`, `ScrollView` не прави додатне алокације због стања.
    
4. **Локални микро-кеш у животном веку компоненте**  
    Нпр. резултат скупе функције користан само у тој компоненти. Мање GC притиска од сталног рекреирања.
    

---

# Када је `useRef` штетан (ризик за цурење меморије)

1. **Чување великих структура „за стално” без чишћења**  
    Велике листе, бафери, слике у `ref.current` и никад их не враћаш на `null` у cleanup-у.
    
    ```tsx
    useEffect(() => {
      bigRef.current = massiveArray; 
      return () => { bigRef.current = null; }; // обавезно
    }, []);
    ```
    
2. **Реф + дуготрајни тајмер/листенер без cleanup-а**  
    `setInterval`, `Keyboard.addListener`, `Dimensions.addEventListener`, `AppState.addEventListener`, gesture/scroll слушаоци.  
    Ако не уклониш листенер, **нативна страна** може да настави да држи референце.
    
    ```tsx
    const intervalRef = useRef<ReturnType<typeof setInterval> | null>(null);
    useEffect(() => {
      intervalRef.current = setInterval(tick, 1000);
      return () => { if (intervalRef.current) clearInterval(intervalRef.current); };
    }, []);
    ```
    
3. **Чување fetch/захтева без отказивања**  
    Ако чуваш `AbortController` у `ref` и не позовеш `abort()` на unmount, позадински посао и буфер могу остати живи дуже.
    
    ```tsx
    const abortRef = useRef<AbortController | null>(null);
    useEffect(() => {
      abortRef.current = new AbortController();
      fetch(url, { signal: abortRef.current.signal });
      return () => abortRef.current?.abort();
    }, [url]);
    ```
    
4. **Кршење граница животног века**  
    Премешташ `ref.current` у глобалну променљиву/сервис → објекат живи дуже од компоненте.
    
5. **Складиштиш UI-стање у `ref`**  
    Очекујеш да се екран освежи кад промениш `ref.current` (неће) → додајеш додатне ефекте и логике → накупљање грешака/ресурса. За UI користи `useState`/стор.
    
6. **Reanimated/Animations погрешан алат**  
    Анимационе вредности држиш у `useRef` → не ради „worklet”/UI тред синхронизација → ручне поправке и потенцијални вишак меморије. За ово је `useSharedValue`.
    

---

# Патерни за здравље меморије

**Увек имај cleanup за све што „живи”:**

- тајмери/интервали → `clearTimeout/clearInterval`
    
- листенери (Keyboard, Dimensions, AppState, NetInfo, EventEmitter) → `remove()`
    
- мрежа → `AbortController.abort()`
    
- стримови/сокети → `close()`
    
- реф бафери → `ref.current = null`
    

**Минимизуј величину у `ref.current`:**

- Чувај само оно што мора да преживи ререндер; велике низове/резултате стави у state/стор/кеш слој (React Query/Redux), где боље контролишеш lifecycle.
    

**Ограничи домет:**

- Немој износити `ref.current` ван компоненте (глобал/сервис) ако не мора. Ако мораш, документуј и управљај lifecycle-ом експлицитно.
    

**Callback рефови:**

- Ако користиш `ref={node => {...}}`, мемоизуј handler (`useCallback`) да избегнеш вишак attach/detach циклуса.
    

---

# Брзе смернице (TL;DR)

- `useRef` је безбедан и лаган; проблем постаје **оно што у њега ставиш**.
    
- Добар: флагови, ID-еви, последње вредности, хендлови ка нативним компонентама, микро-кеш у оквиру екрана.
    
- Лош: огромни објекти без чишћења, дуготрајни listeners/intervals без cleanup-а, „замрзнути” fetch-еви, глобализовани `ref`.
    
- На unmount: почисти све што си сместио у `ref` и што држи ресурсе (listeners, контролере, бафере).
    

 