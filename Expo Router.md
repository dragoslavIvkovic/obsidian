Super, Vukašine — evo kratkog “bootcampa” za **Expo Router** (expo-router) + checklista najčešćih grešaka i kako ih rešiti. Držim se TypeScript-a i najnovijeg (v3) obrasca.

# 1) Šta je Expo Router (ukratko)

- **File-based routing**: struktura foldera u `app/` = rute.
- **Bez ručne registracije**: nema `createNativeStackNavigator` konfiguracije — rute nastaju iz fajlova.
- **Stack, Tabs, Drawer**: koriste se kroz _layout fajlove.
- **Deep linking** i **URL parametri**: dobijaju se kroz `useLocalSearchParams()`.

# 2) Minimalni start

```bash
npx create-expo-app MyApp
cd MyApp
npm i expo-router
npx expo install react-native-safe-area-context react-native-screens

```

`package.json`:

```json
{
  "main": "expo-router/entry"
}

```

`app/_layout.tsx` (root layout = Stack kontejner):

```tsx
import { Stack } from "expo-router";

export default function RootLayout() {
  return (
    <Stack screenOptions={{ headerShown: false }} />
  );
}

```

`app/index.tsx` (home):

```tsx
import { Link } from "expo-router";
import { View, Text } from "react-native";

export default function Home() {
  return (
    <View>
      <Text>Home</Text>
      <Link href="/about">Idi na About</Link>
    </View>
  );
}

```

`app/about.tsx`:

```tsx
import { View, Text } from "react-native";
export default function About() {
  return <View><Text>About</Text></View>;
}

```

# 3) Imperativna navigacija

Možeš bilo gde u komponenti:

```tsx
import { router } from "expo-router";

// push (dodaj na stack)
router.push("/about");

// replace (zameni trenutni ekran)
router.replace("/login");

// back (kao Android back)
router.back();

```

Sa parametrima:

```tsx
router.push({ pathname: "/product/[id]", params: { id: "42" } });

```

# 4) Dinamičke rute i parametri

Struktura:

```
app/
  product/
    [id].tsx

```

Čitanje parametara:

```tsx
import { useLocalSearchParams } from "expo-router";

export default function ProductScreen() {
  const { id } = useLocalSearchParams<{ id: string }>();
  // id je "42"
  return null;
}

```

Catch-all:

```
app/docs/[...slug].tsx   // /docs/a/b/c -> slug=["a","b","c"]

```

# 5) Grupe, modali, autentiikacija

**Grupe** služe za organizaciju i zajednički layout (nema uticaja na URL):

```
app/
  (app)/
    _layout.tsx
    index.tsx
    profile.tsx
  (auth)/
    _layout.tsx
    login.tsx
    register.tsx

```

**Modalni ekrani** (preporučeno):

```
app/
  (app)/
    _layout.tsx
    index.tsx
    settings.tsx
  (modals)/
    _layout.tsx         // Stack sa presentation="modal"
    contact.tsx

```

Primer modal layouta:

```tsx
// app/(modals)/_layout.tsx
import { Stack } from "expo-router";
export default function ModalLayout() {
  return (
    <Stack screenOptions={{ presentation: "modal" }} />
  );
}

```

**Zaštita ruta (auth gate)**:

```tsx
// app/(app)/_layout.tsx
import { Slot, Redirect } from "expo-router";
import { useAuth } from "../providers/auth"; // tvoj hook

export default function AppLayout() {
  const { user, loading } = useAuth();
  if (loading) return null;
  if (!user) return <Redirect href="/(auth)/login" />;
  return <Slot />;
}

```

# 6) Tabs i Drawer

Tabs:

```
app/(tabs)/
  _layout.tsx
  index.tsx
  search.tsx
  favorites.tsx

```

```tsx
// app/(tabs)/_layout.tsx
import { Tabs } from "expo-router";
export default function TabsLayout() {
  return (
    <Tabs>
      <Tabs.Screen name="index" options={{ title: "Home" }} />
      <Tabs.Screen name="search" options={{ title: "Search" }} />
      <Tabs.Screen name="favorites" options={{ title: "Favs" }} />
    </Tabs>
  );
}

```

Drawer (slično, koristi `Drawer` umesto `Tabs`).

# 7) Tip-safe linkovi i rute (TS)

- `Href` tip pomaže da ne pogrešiš putanju:

```tsx
import { Href, Link } from "expo-router";
const path: Href = "/about";
<Link href={path}>Go</Link>

```

- Za parametre:

```tsx
router.push({ pathname: "/product/[id]" as Href, params: { id: "7" } });

```

# 8) Header opcije po ekranu

```tsx
import { Stack } from "expo-router";

export const unstable_settings = { initialRouteName: "index" };

export default function Layout() {
  return (
    <Stack>
      <Stack.Screen name="index" options={{ title: "Home" }} />
      <Stack.Screen name="about" options={{ title: "About", headerShown: true }} />
    </Stack>
  );
}

```

# 9) Deep linking (schema)

`app.json` / `app.config.ts`:

```tsx
export default {
  expo: {
    scheme: "myapp",
    ios: { bundleIdentifier: "com.me.myapp" },
    android: { package: "com.me.myapp" },
    extra: { router: { origin: "expo" } }
  }
};

```

Otvaranje linka:

```tsx
import * as Linking from "expo-linking";
Linking.openURL("myapp://product/42");

```

# 10) Data-fetching i state

- U ekranima radi kao u običnom RN: `useEffect`, React Query, Zustand, Redux…
- Za SSR nema na native, ali **Web** podržava URL-ove i možeš deliti rute (pazi na platform-specific code).

---

# Najčešće greške i kako ih rešiti

1. **“Cannot find module 'expo-router'” / “Unable to resolve module expo-router”**
    
    **Uzrok:** Nije instaliran ili `main` nije `expo-router/entry`.
    
    **Rešenje:** `npm i expo-router` i u `package.json` `"main": "expo-router/entry"`. Restart Metro: `expo start -c`.
    
2. **Crni ekran ili prazna app posle dodavanja routera**
    
    **Uzrok:** Nema `app/_layout.tsx` ili vraća `null` bez `Slot/Stack`.
    
    **Rešenje:** Dodaj root `_layout.tsx` sa `<Stack/>` ili `<Slot/>`.
    
3. **“No route found for path …”**
    
    **Uzrok:** Pogrešna putanja u `router.push`/`Link`, ili fajl nije na očekivanom mestu.
    
    **Rešenje:** Proveri ime fajla i hijerarhiju u `app/`. Putanja je **relativna strukturi**, npr. `app/product/[id].tsx` ↔ `/product/123`.
    
4. **Parametri undefined u `useLocalSearchParams()`**
    
    **Uzrok:** Nisi poslao `params` u `router.push`, ili ime parametra ne odgovara `[id].tsx`.
    
    **Rešenje:** `router.push({ pathname: "/product/[id]", params: { id: "123" } })`. Ime foldera **mora** da se poklopi.
    
5. **“useRouter must be used within a router context”**
    
    **Uzrok:** Pokušan import izvan ekrana/layouta u `app/` ili u fajlu koji se mount-uje pre Routera.
    
    **Rešenje:** Koristi `router` samo u komponentama koje su deca `_layout`/ruta.
    
6. **Duplirani headeri ili čudna navigacija**
    
    **Uzrok:** U gnezdu imaš više `Stack` layouta bez potrebe.
    
    **Rešenje:** Smanji broj `_layout.tsx` ili premesti screen opcije u jedan viši `Stack`.
    
7. **Android Back izlazi iz app umesto back u stack-u**
    
    **Uzrok:** Root stack nema gde da se vrati ili nepravilno rukovanje.
    
    **Rešenje:** Proveri da li koristiš `router.back()` u kritičnim tačkama; na početnom ekranu razmisli o potvrdi izlaza.
    
8. **Linkovi ne otvaraju app (deep link)**
    
    **Uzrok:** Schema nije podešena, ili se razlikuje između iOS/Android.
    
    **Rešenje:** Podesi `scheme` u `app.json`/`app.config.ts`. Na Androidu proveri `intentFilters`.
    
9. **“The action 'NAVIGATE' was not handled”** (ređe sa expo-router)
    
    **Uzrok:** Navigacija ka ruti koja ne postoji u tom Tab/Stack kontekstu.
    
    **Rešenje:** Proveri da li ekran postoji u tom layoutu; možda treba `router.push("/(tabs)/search")` umesto samo `/search`.
    
10. **Konflikt imena fajlova / keš Metro bundlera**
    
    **Uzrok:** Preimenovanje fajlova i stari keš.
    
    **Rešenje:** `expo start -c`, ugasi i upali bundler, obriši `node_modules/.cache` ako treba.
    
11. **TypeScript greške sa `Href` ili parametrima**
    
    **Uzrok:** Netipiziran `pathname` ili pogrešni nazivi parametara.
    
    **Rešenje:** Kastuj na `Href` i koristi iste nazive kao u `[id].tsx`. Dodaj generike u `useLocalSearchParams<{ id: string }>`.
    
12. **Modali se ne prikazuju kao modal**
    
    **Uzrok:** Nisi stavio modalne rute pod layout sa `presentation:"modal"`.
    
    **Rešenje:** Napravi grupu `(modals)/_layout.tsx` sa `Stack` koji ima `presentation:"modal"`.
    
13. **Auth redirect petlja**
    
    **Uzrok:** `Redirect` renderuje dok se auth još učitava.
    
    **Rešenje:** Dok `loading` true → prikaži “splash”/loader; tek posle odluči `Redirect`.
    
14. **Pomešani Tabs i Stack (ne otvara se očekivano)**
    
    **Uzrok:** Pokušaj da otvoriš screen izvan trenutne grupe.
    
    **Rešenje:** Navigiraj do “punog” puta (sa grupom), ili prestrukturiraj rute.
    
15. **Web path ne radi kao mobile ruta**
    
    **Uzrok:** Posebnosti web-a (history, basePath).
    
    **Rešenje:** Proveri `experiments.webRouterRoot`/konfiguraciju ako koristiš napredne web opcije (ili drži fokus na native).
    

# Debug i dijagnostika (brza lista)

- Očisti bundler: `expo start -c`
    
- Proveri instalacije: `npx expo-doctor`, `npx expo install expo-router`
    
- Štampaj trenutnu rutu:
    
    ```tsx
    import { usePathname } from "expo-router";
    console.log(usePathname());
    
    ```
    
- Proveri da li fajl postoji tačno gde misliš (`app/…` putanja).
    
- Uključi TS strožu proveru za rute/parametre.
    

# Mini-cheatsheet (za svaki projekat)

- `app/_layout.tsx` sa `<Stack/>`
- Stranice: `app/index.tsx`, `app/about.tsx`
- Dinamički: `app/product/[id].tsx` (+ `router.push({ pathname:"/product/[id]", params:{ id } })`)
- Tabs: `app/(tabs)/_layout.tsx` + `Tabs.Screen`
- Auth gate: `Redirect` u grupi `(app)` layouta
- Modal: `(modals)/_layout.tsx` sa `presentation:"modal"`

Ako hoćeš, mogu ti napraviti **starter strukturu** za tvoj slučaj (npr. Tabs + Auth + Modals + Products) koju samo copy-paste-uješ u `app/` i kreneš.

Odlično pitanje — evo razlike između **Stack**, **Tabs**, **Drawer** i **Slot** u Expo Router-u:

---

## 🔹 1. Stack

- **Šta je:** Navigacija bazirana na **stogu ekrana** (stack navigator).
- **Kako radi:** Svaki novi ekran se “stavlja” na vrh steka; `router.push()` dodaje, `router.back()` skida.
- **Gde se koristi:**
    - Kada imaš hijerarhiju (Home → Details → Subdetails).
    - Kada želiš back dugme koje vraća korak unazad.

**Primer:**

```tsx
// app/_layout.tsx
import { Stack } from "expo-router";

export default function RootLayout() {
  return <Stack />;
}

```

---

## 🔹 2. Tabs

- **Šta je:** Navigacija pomoću **donje trake sa tabovima** (bottom tabs).
- **Kako radi:** Svaki tab je ekran ili grupa ekrana; između tabova se prebacuje paralelno (ne briše state).
- **Gde se koristi:**
    - Za aplikacije tipa Instagram (Home / Search / Reels / Profile).
    - Kada korisnik često menja glavne sekcije.

**Primer:**

```tsx
// app/(tabs)/_layout.tsx
import { Tabs } from "expo-router";

export default function TabsLayout() {
  return (
    <Tabs>
      <Tabs.Screen name="index" options={{ title: "Home" }} />
      <Tabs.Screen name="profile" options={{ title: "Profile" }} />
    </Tabs>
  );
}

```

---

## 🔹 3. Drawer

- **Šta je:** Navigacija pomoću **bočnog menija** (hamburger menu).
- **Kako radi:** Korisnik prevuče sa leve strane ili klikne dugme i otvori se meni.
- **Gde se koristi:**
    - Kada imaš mnogo opcija/sekcija.
    - Za enterprise app-ove ili aplikacije sa složenim sadržajem.

**Primer:**

```tsx
// app/(drawer)/_layout.tsx
import { Drawer } from "expo-router";

export default function DrawerLayout() {
  return (
    <Drawer>
      <Drawer.Screen name="index" options={{ title: "Home" }} />
      <Drawer.Screen name="settings" options={{ title: "Settings" }} />
    </Drawer>
  );
}

```

---

## 🔹 4. Slot

- **Šta je:** **Placeholder** u layoutu gde će se učitati “deca” rute.
- **Kako radi:** Ako u `_layout.tsx` staviš `<Slot />`, Expo Router tu ubaci sve ekrane koji spadaju u taj layout.
- **Gde se koristi:**
    - Kada želiš da oko ekrana imaš zajednički UI (npr. header, footer, context provider).
    - Kada nemaš Stack/Tabs/Drawer, već samo “passthrough”.

**Primer:**

```tsx
// app/(app)/_layout.tsx
import { Slot } from "expo-router";
import { View, Text } from "react-native";

export default function AppLayout() {
  return (
    <View style={{ flex: 1 }}>
      <Text>Header</Text>
      <Slot />   {/* ovde se ubacuje npr. index.tsx ili about.tsx */}
      <Text>Footer</Text>
    </View>
  );
}

```

---

## 📝 Ukratko

- **Stack** = “korak po korak” navigacija (najčešći slučaj).
- **Tabs** = donja traka za brzo menjanje sekcija.
- **Drawer** = bočni meni za mnogo opcija.
- **Slot** = rupa u layoutu gde se prikazuje trenutna ruta.

---

Hoćeš da ti napravim **uporedni primer projekta** gde su u jednoj aplikaciji zajedno Stack + Tabs + Drawer + Slot (mala demo struktura za copy-paste)?

Evo ti jasna razlika, Vukašine 👇

---

## 🔹 `router.push("/about")`

- **Šta radi:** Dodaje **novi ekran na vrh steka**.
- **Primer:**
    - Trenutno si na `/home`.
    - Uradiš `router.push("/about")`.
    - Sad stack izgleda: `[home, about]`.
    - Ako klikneš back, vratiće te na `/home`.
- 👉 Koristiš kad hoćeš da korisnik može da se vrati nazad.

---

## 🔹 `router.replace("/login")`

- **Šta radi:** **Zamenjuje trenutni ekran** novim (nema povratka).
- **Primer:**
    - Trenutno si na `/home`.
    - Uradiš `router.replace("/login")`.
    - Stack sada izgleda: `[login]` (home je obrisan).
    - Ako klikneš back, app se zatvara (nema gde nazad).
- 👉 Koristiš za login/logout, onboarding, redirekcije gde nema smisla vraćati se nazad.

---

## 🔹 `router.back()`

- **Šta radi:** Vraća **jedan korak unazad** u stacku (kao dugme za back na Androidu/iOS).
- **Primer:**
    - Stack je `[home, about, details]`.
    - `router.back()` → sad si na `about`.
    - Još jedan `router.back()` → sad si na `home`.
- 👉 Koristiš kada želiš eksplicitno back akciju (npr. custom dugme).

---

## 📌 Ukratko

- `push` → dodaj novu stranicu (možeš nazad).
- `replace` → zameni trenutnu (nema nazad).
- `back` → idi unazad (ako ima gde).

---

Hoćeš da ti nacrtam **vizuelni dijagram stack-a** (kutije sa push/replace/back) da sve bude baš jasno?

Odlično pitanje — to je **srž razumevanja stack navigacije** u React Native / Expo Router-u 👇

---

## 🔹 Kada koristiš `router.push()`

- **Šta se dešava:**
    - Novi ekran se **dodaje na stack**, prethodni ekran **ostaje u memoriji**.
    - To znači da se **state, hookovi i context** prethodnog ekrana čuvaju (ako se ponovo vratiš `back`, biće u istom stanju).

**Primer:**

- `/home` → korisnik u formi kuca nešto.
- `router.push("/about")`.
- Ako se vratiš `router.back()`, sve što je uneo u formu i dalje je tu.

---

## 🔹 Kada koristiš `router.replace()`

- **Šta se dešava:**
    - Trenutni ekran se **uništava i uklanja iz memorije**.
    - Novi ekran zauzima njegovo mesto.
    - Ako pokušaš `back()`, nemaš gde — prethodni ekran više ne postoji.

**Primer:**

- `/login` → korisnik se ulogovao.
- `router.replace("/home")`.
- `/login` se briše iz memorije (više nema stanja, inputa, ničega).

---

## 🔹 Kada koristiš `router.back()`

- **Šta se dešava:**
    - Trenutni ekran se **uništava** (skida se sa steka).
    - Vraćaš se na prethodni koji je ostao u memoriji.

**Primer:**

- Stack: `[home, about, details]`.
- `router.back()` → `details` se briše iz memorije, `about` ostaje aktivan sa svim svojim stanjem.

---

## 🔹 Rezime (memorija vs uništavanje)

- **push** → novi ekran dodan, stari ostaje u memoriji.
- **replace** → stari ekran obrisan iz memorije, novi zauzima mesto.
- **back** → trenutni ekran obrisan iz memorije, vraćaš se na prethodni (koji je i dalje u memoriji).

---

👉 Bitna napomena:

Ako odeš na **potpuno drugu navigacionu grupu** (npr. iz `(tabs)` u `(auth)`), stari deo može biti unmount-ovan ceo (uništava se state, context, itd.), jer Expo Router tretira različite grupe kao odvojene navigacione “stacks”.

---

Hoćeš da ti napravim **tabelu sa push/replace/back + primer stack-a + šta se briše/šta ostaje** da imaš kao “cheat sheet”?

Evo ti **cheat sheet tabela** za memoriju i stack u **Expo Router-u** 👇

---

# 📊 Navigacija i memorija u Expo Router-u

|Akcija|Pre stack (primer)|Posle akcije|Šta ostaje u memoriji|Šta se uništava|
|---|---|---|---|---|
|**`router.push("/about")`**|`[home]`|`[home, about]`|`home` ostaje sa svim state-om, `about` se učitava|Ništa se ne briše|
|**`router.push("/details")`**|`[home, about]`|`[home, about, details]`|`home` i `about` ostaju u memoriji, `details` se učitava|Ništa se ne briše|
|**`router.back()`**|`[home, about, details]`|`[home, about]`|`home` i `about` ostaju netaknuti (state očuvan)|`details` se briše iz memorije|
|**`router.replace("/login")`**|`[home]`|`[login]`|Samo `login` postoji, `home` se briše|`home` ekran uništen|
|**`router.replace("/dashboard")`**|`[login]`|`[dashboard]`|Samo `dashboard` postoji|`login` se briše|
|**Kombinacija (push → back)**|`[home] → push("/about") → [home, about] → back()`|`[home]`|`home` je ostao netaknut (state isti)|`about` se briše|

---

## 🔹 Pravila za pamćenje

- `push` ➝ **dodaje** ekran (ništa se ne briše).
- `replace` ➝ **briše trenutni ekran** i ubacuje novi.
- `back` ➝ **skida trenutni ekran** i vraća prethodni.

---

👉 Ako želiš, mogu da ti nacrtam **vizuelni dijagram sa kutijama** (stack pre/posle za push, replace, back) da bukvalno vidiš kako se ekrani slažu i brišu. Hoćeš i to?

Da — **Expo Router** može da otkrije naziv trenutne stranice (ekrana), i to na više načina, u zavisnosti od toga **šta tačno želiš** (npr. ime rute, putanju, segment, parametre).  
Evo detaljno kako to funkcioniše 👇

---

### 🧭 1. Dohvatanje trenutne rute pomoću `useRouter`, `useSegments` ili `usePathname`

#### ✅ `usePathname()`

Najjednostavniji način da vidiš “naziv” stranice (tj. putanju).

```tsx
import { usePathname } from 'expo-router'

export default function MyScreen() {
  const pathname = usePathname()

  console.log(pathname) // npr. "/settings/profile"

  return null
}
```

- Ako si na stranici `app/settings/profile.tsx`, vratiće `/settings/profile`.
    
- Možeš da koristiš `pathname.split('/')` da izdvojiš samo poslednji deo, tj. “naziv” ekrana:
    

```tsx
const screenName = pathname.split('/').pop()
console.log(screenName) // "profile"
```

---

### 🧩 2. Ako koristiš **layouts** (`_layout.tsx`) i želiš ime trenutnog segmenta

Koristi **`useSegments()`**:

```tsx
import { useSegments } from 'expo-router'

export default function Layout() {
  const segments = useSegments()

  console.log(segments) // npr. ["settings", "profile"]

  const current = segments[segments.length - 1]
  console.log(current) // "profile"

  return null
}
```

To je korisno ako imaš ugnježdene rute (nested routes) i želiš precizno da znaš u kom “segmentu” si.

---

### 🧠 3. Ako ti treba ime rute za navigaciju ili logiku (npr. analytics, header title…)

Možeš da koristiš `useLocalSearchParams()` ako ti treba parametar rute (npr. `[id].tsx`):

```tsx
import { useLocalSearchParams } from 'expo-router'

export default function Details() {
  const { id } = useLocalSearchParams()

  console.log(id) // vrednost iz rute, npr. "42"
}
```

---

### 🧰 4. Kombinovano — logički “naziv ekrana”

Ako želiš univerzalnu funkciju koja vraća “naziv ekrana” (za npr. analytics logove), možeš ovako:

```tsx
import { usePathname } from 'expo-router'

export function useScreenName() {
  const pathname = usePathname()
  const segments = pathname.split('/').filter(Boolean)
  return segments.pop() || 'home'
}
```

Koristiš u bilo kom ekranu:

```tsx
const screenName = useScreenName()
console.log('Trenutni ekran:', screenName)
```

---

### 📊 5. Za integraciju sa Analytics (npr. PostHog, Firebase, Amplitude...)

Možeš da slušaš promene rute:

```tsx
import { usePathname } from 'expo-router'
import { useEffect } from 'react'

export function useScreenTracking() {
  const pathname = usePathname()

  useEffect(() => {
    // tvoj kod za analytics event
    console.log('Screen viewed:', pathname)
  }, [pathname])
}
```

---

Super — to je odlična ideja 👏  
Evo kako možemo to da uradimo: napraviću **vizuelni dijagram u tekstualnom formatu (kutije + strelice)** koji pokazuje:

- kako _Stack Navigator_ u **Expo Router v6** funkcioniše,
    
- šta se dešava kada koristiš `push`, `replace`, ili `back`,
    
- i kako `layout` + `index` učestvuju u tome.
    

---

## 🧱 OSNOVNI STACK (layout + index)

```
┌──────────────────────────────┐
│ app/_layout.tsx              │   ← Root layout sa <Stack>
│ ──────────────────────────── │
│   <Slot />                   │   ← Mesto gde se ubacuje aktivna ruta
└──────────────────────────────┘
           │
           ▼
┌──────────────────────────────┐
│ app/index.tsx                │   ← Home screen (ruta "/")
└──────────────────────────────┘
```

➡️ Kada pokreneš app, `layout` se učita **jednom**, a `index.tsx` se ubaci u `<Slot />`.

---

## 📚 Kada koristiš `router.push("/details")`

```
STACK PRE:
┌────────────┐
│ index.tsx  │  ← Home
└────────────┘

STACK POSLE:
┌────────────┐
│ index.tsx  │  ← Home
├────────────┤
│ details.tsx│  ← Nova stranica (dodato na vrh)
└────────────┘
```

➡️ `push` **dodaje novi ekran na vrh stacka**  
🔙 `router.back()` te vraća nazad (briše gornji ekran).

---

## 🔄 Kada koristiš `router.replace("/details")`

```
STACK PRE:
┌────────────┐
│ index.tsx  │
└────────────┘

STACK POSLE:
┌────────────┐
│ details.tsx│  ← Stari ekran zamenjen novim
└────────────┘
```

➡️ `replace` **briše trenutni ekran i ubacuje novi** (ne možeš se vratiti nazad).

---

## ⬅️ Kada koristiš `router.back()`

```
STACK PRE:
┌────────────┐
│ index.tsx  │
├────────────┤
│ details.tsx│
└────────────┘

STACK POSLE:
┌────────────┐
│ index.tsx  │  ← Vraćeno na prethodni ekran
└────────────┘
```

➡️ `back()` uklanja zadnji ekran iz stacka.

---

## 💡 Uloga `_layout.tsx` ovde

Zamisli `_layout.tsx` kao okvir (navigation container):

```
┌──────────────────────────────┐
│ Root _layout (Stack)         │
│ ──────────────────────────── │
│   ┌────────────┐             │
│   │ index.tsx  │             │
│   └────────────┘             │
│   ┌────────────┐             │
│   │ details.tsx│             │
│   └────────────┘             │
└──────────────────────────────┘
```

Layout ostaje u memoriji, dok se sadržaj `<Slot />` menja pri navigaciji (`push`, `replace`, `back`).

---

Ako želiš, mogu da ti napravim **grafički dijagram** (kutije sa strelicama kao slika, npr. za Notion prezentaciju), gde se vidi i `layout`, `index`, `router.push/replace/back`.  
Hoćeš da to generišem kao vizuelnu sliku (sa obojenim boxovima i strelicama)?