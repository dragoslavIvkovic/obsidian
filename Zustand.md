

### Šta je Zustand?

Zustand je malo, brzo i skalabilno rešenje za upravljanje stanjem (state management). Njegovo ime na nemačkom znači "stanje". Ključna ideja Zustanda je jednostavnost i minimalizam. Ne zahteva mnogo "boilerplate" koda (kao što je to često slučaj sa Reduxom) i koristi moderan pristup zasnovan na hook-ovima, što ga čini veoma lakim za integraciju u React aplikacije.

Osnovne karakteristike:

  * **Minimalan API:** Lako se uči i koristi.
  * **Manje koda:** Značajno smanjuje količinu koda potrebnu za upravljanje stanjem.
  * **Nije vezan samo za React:** Može se koristiti i van React komponenti.
  * **Koristi hook-ove:** Prirodno se integriše u React funkcionalne komponente.

-----

### 1\. Osnove: Kreiranje i Korišćenje Store-a

Sve u Zustandu počinje sa kreiranjem "store-a". Store je kontejner za tvoje stanje i akcije koje to stanje menjaju. Za ovo se koristi `create` funkcija.

#### Kreiranje Store-a

Store se definiše pozivanjem `create` funkcije. Ova funkcija prihvata callback funkciju koja vraća objekat. Taj objekat je tvoj inicijalni state.

```javascript
import { create } from 'zustand'

const useBearStore = create((set) => ({
  bears: 0,
  increasePopulation: () => set((state) => ({ bears: state.bears + 1 })),
  removeAllBears: () => set({ bears: 0 }),
}))
```

Hajde da razložimo ovaj primer:

  * `create((set) => ({ ... }))`: Glavna funkcija za kreiranje store-a.
  * `set`: Ovo je najvažnija funkcija koju dobijaš. Koristiš je da bi menjao stanje. Ona "zna" kako da bezbedno ažurira state i obavesti sve komponente koje ga koriste.
  * `bears: 0`: Ovo je deo našeg stanja (state).
  * `increasePopulation`: Ovo je "akcija". Funkcija koja, kada se pozove, koristi `set` da ažurira stanje. Ovde koristimo pristup sa funkcijom `(state) => ({ ... })` da bismo dobili trenutno stanje pre ažuriranja. Ovo je preporučeni način kada novo stanje zavisi od prethodnog.
  * `removeAllBears`: Još jedna akcija. Ovde direktno prosleđujemo objekat `set({ bears: 0 })` jer novo stanje ne zavisi od starog.

#### Korišćenje Store-a u Komponenti

Kada je store kreiran, možeš ga koristiti u bilo kojoj React komponenti. Kreirani `useBearStore` je sada hook\!

```jsx
import React from 'react';
import { useBearStore } from './path-to-your-store';

function BearCounter() {
  const bears = useBearStore((state) => state.bears);
  return <h1>{bears} around here ...</h1>;
}

function Controls() {
  const increasePopulation = useBearStore((state) => state.increasePopulation);
  return <button onClick={increasePopulation}>one up</button>;
}
```

**Ključna stvar za razumevanje (Selectors):**
Obrati pažnju na `useBearStore((state) => state.bears)`. Ovo se zove "selektor". Umesto da uzmemo ceo store objekat, mi biramo samo onaj deo stanja koji nam je potreban.

**Zašto je ovo važno?** React komponenta će se ponovo renderovati **samo ako se vrednost koju selektor vraća promeni**.

  * Ako koristiš `const { bears, increasePopulation } = useBearStore()`, tvoja komponenta će se renderovati svaki put kada se bilo šta u store-u promeni.
  * Ako koristiš `const bears = useBearStore(state => state.bears)`, komponenta će se renderovati samo kada se `bears` promeni. Ovo je ključ za dobre performanse.

-----

### 2\. Rad sa Akcijama i Asinhronim Operacijama

Akcije ne moraju biti jednostavne. One mogu imati logiku, pozivati druge akcije ili raditi sa asinhronim operacijama kao što je pozivanje API-ja.

#### Pristupanje Stanju i Akcijama unutar Akcije

Ponekad jedna akcija treba da pristupi drugim delovima stanja ili čak da pozove drugu akciju. Za to se koristi `get()` funkcija.

```javascript
import { create } from 'zustand'

const useExampleStore = create((set, get) => ({
  value: 0,
  text: "hello",
  incrementValue: () => set((state) => ({ value: state.value + 1 })),
  logCurrentState: () => {
    // Koristimo get() da pročitamo trenutno stanje bez re-renderovanja
    const currentValue = get().value;
    const currentText = get().text;
    console.log(`Current value is ${currentValue} and text is ${currentText}`);
    
    // Možemo pozvati i drugu akciju
    get().incrementValue();
  }
}))
```

  * `set`: Koristi se za **pisanje** (ažuriranje) stanja.
  * `get`: Koristi se za **čitanje** trenutnog stanja unutar akcija.

#### Asinhrone Akcije (npr. API poziv)

Zustand ovo rešava veoma elegantno. Pošto su akcije obične funkcije, možeš koristiti `async/await` kao što bi i inače radio.

Primer: Store koji dohvata listu repozitorijuma sa GitHub-a.

```javascript
import { create } from 'zustand'

const useRepoStore = create((set) => ({
  repos: [],
  loading: false,
  error: null,
  
  fetchRepos: async (username) => {
    set({ loading: true, error: null }); // Postavi stanje na "učitava se"
    try {
      const response = await fetch(`https://api.github.com/users/${username}/repos`);
      if (!response.ok) {
        throw new Error('Network response was not ok');
      }
      const data = await response.json();
      set({ repos: data, loading: false }); // Uspešno, sačuvaj podatke
    } catch (error) {
      set({ error: error.message, loading: false }); // Greška, sačuvaj poruku o grešci
    }
  },
}));
```

Kako bi se ovo koristilo u komponenti:

```jsx
function RepoList() {
  const { repos, loading, error, fetchRepos } = useRepoStore();

  React.useEffect(() => {
    // Pozivamo akciju kada se komponenta prvi put učita
    fetchRepos('pmndrs'); 
  }, [fetchRepos]);

  if (loading) return <p>Loading...</p>;
  if (error) return <p>Error: {error}</p>;

  return (
    <ul>
      {repos.map(repo => (
        <li key={repo.id}>{repo.name}</li>
      ))}
    </ul>
  );
}
```

Ovde smo uzeli ceo store objekat jer nam trebaju svi njegovi delovi u ovoj komponenti.

-----

### 3\. Middleware

Middleware su funkcije koje "obmotavaju" tvoj store i dodaju mu nove mogućnosti. Dva najpopularnija su `devtools` i `persist`.

#### `devtools` Middleware

Ovaj middleware povezuje tvoj Zustand store sa Redux DevTools ekstenzijom u browseru. Iako si rekao da se ne koriste, važno je da razumeš kako funkcionišu po dokumentaciji, jer je to jedna od ključnih mogućnosti Zustanda za debagovanje.

**Kako radi?** Svaka akcija koja se desi biće zabeležena u DevTools. Moći ćeš da vidiš kako se stanje menjalo korak po korak, što je neverovatno korisno za praćenje tokova podataka i pronalaženje grešaka.

**Implementacija:**
Jednostavno obmotaš svoj kreator store-a sa `devtools` funkcijom.

```javascript
import { create } from 'zustand'
import { devtools } from 'zustand/middleware'

// Originalni kreator store-a
const bearStoreCreator = (set) => ({
  bears: 0,
  increasePopulation: () => set((state) => ({ bears: state.bears + 1 })),
  removeAllBears: () => set({ bears: 0 }),
})

// Kreiranje store-a sa devtools middleware-om
const useBearStore = create(devtools(bearStoreCreator));

// Možeš mu dati i ime koje će se prikazivati u DevTools
// const useBearStore = create(devtools(bearStoreCreator, { name: "Bear Store" }));
```

Sada, kada pozoveš `increasePopulation`, u Redux DevTools tab-u u browseru ćeš videti akciju pod nazivom "increasePopulation", prethodno stanje i novo stanje.

#### Drugi korisni Middleware: `persist`

Ovaj middleware omogućava da automatski sačuvaš stanje svog store-a u `localStorage` (ili `sessionStorage`, `AsyncStorage` u React Native-u). Kada korisnik osveži stranicu, stanje će biti vraćeno.

```javascript
import { create } from 'zustand'
import { persist } from 'zustand/middleware'

const useSettingsStore = create(
  persist(
    (set) => ({
      theme: 'light',
      toggleTheme: () => set((state) => ({ theme: state.theme === 'light' ? 'dark' : 'light' })),
    }),
    {
      name: 'app-settings', // Ime pod kojim će se sačuvati u local storage
      // getStorage: () => sessionStorage, // Opciono, podrazumevano je localStorage
    }
  )
)
```

Sada, svaki put kada se `theme` promeni, automatski će se sačuvati. Kada se aplikacija ponovo učita, `persist` middleware će automatski povući sačuvanu vrednost.

-----

### 4\. Saveti i Dobre Prakse

  * **Razdvajanje Store-ova (Slices):** Za veće aplikacije, nemoj stavljati sve u jedan ogroman store. Kreiraj više manjih, fokusiranih store-ova. Na primer: `useAuthStore`, `useCartStore`, `useProductStore`. Ovo čini kod organizovanijim i lakšim za održavanje.
  * **Koristi selektore pametno:** Kao što je pomenuto, uvek biraj samo one delove stanja koji su ti potrebni u komponenti da bi izbegao nepotrebne re-rendere.
    `const user = useAuthStore(state => state.user);` je bolje od `const { user } = useAuthStore();` ako ti treba samo `user`.
  * **Izvedeno stanje (Derived State):** Ako imaš stanje koje se može izračunati iz postojećeg stanja, nemoj ga čuvati u store-u. Izračunaj ga direktno u komponenti.
    ```jsx
    function Cart() {
      // Umesto da čuvamo 'totalPrice' u store-u...
      const items = useCartStore(state => state.items);
      
      // ...izračunamo ga kada nam zatreba.
      const totalPrice = items.reduce((acc, item) => acc + item.price, 0);

      return <div>Total: ${totalPrice}</div>
    }
    ```

Nadam se da ti ovaj detaljan pregled pomaže da razumeš snagu i jednostavnost Zustanda. Počni sa jednostavnim store-om i postepeno dodaj složenije stvari kao što su asihrone akcije i middleware. Srećno\!
Odlično pitanje. Hajde da detaljno objasnimo kako `persist` i hidratacija rade zajedno u Zustandu, sa konkretnim primerom za React Native.

Ukratko:

1.  **`persist`** je *middleware* (dodatak) koji **čuva** stanje tvog store-a na fizičkoj memoriji telefona.
2.  **Hidratacija (Hydration)** je proces **učitavanja** tog sačuvanog stanja nazad u store kada se aplikacija ponovo pokrene.

`persist` middleware automatski radi obe stvari. Ti mu kažeš *šta* da čuva i *gde* da čuva, a on se brine o čuvanju i kasnijoj hidrataciji.

-----

### Kako radi `persist` Storage u React Native-u

U web pregledačima, podrazumevano mesto za čuvanje je `localStorage`. Međutim, u React Native-u **ne postoji `localStorage`**. Umesto toga, standard za jednostavno čuvanje podataka je **`AsyncStorage`**.

Moramo eksplicitno reći Zustand `persist` middleware-u da koristi `AsyncStorage`.

#### Korak 1: Instalacija potrebnih paketa

Ako već nisi, treba ti `AsyncStorage`.

```bash
npm install @react-native-async-storage/async-storage
# ili
yarn add @react-native-async-storage/async-storage
```

#### Korak 2: Kreiranje Store-a sa `persist`

Sada ćemo napraviti store koji čuva neka podešavanja aplikacije, na primer, da li je korisnik odabrao tamnu temu i da li su notifikacije uključene.

```javascript
// src/stores/useSettingsStore.js

import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';
import AsyncStorage from '@react-native-async-storage/async-storage';

const useSettingsStore = create(
  persist(
    (set, get) => ({
      // 1. Naše stanje (state)
      theme: 'light',
      notificationsEnabled: true,
      _hasHydrated: false, // Tehničko stanje za praćenje hidratacije

      // 2. Naše akcije (actions)
      toggleTheme: () => set((state) => ({ 
        theme: state.theme === 'light' ? 'dark' : 'light' 
      })),
      
      setNotifications: (enabled) => set({ notificationsEnabled: enabled }),
      
      setHasHydrated: (state) => {
        set({
          _hasHydrated: state
        });
      }
    }),
    {
      // 3. Konfiguracija za persist middleware
      name: 'app-settings-storage', // Ime pod kojim se čuvaju podaci
      storage: createJSONStorage(() => AsyncStorage), // EKSPLICITNO kažemo da koristi AsyncStorage
      onRehydrateStorage: () => (state) => {
        // Ovo se izvršava kada se stanje uspešno učita (hidrira)
        state.setHasHydrated(true);
      }
    }
  )
);

export default useSettingsStore;
```

**Analiza koda:**

1.  **Stanje:** Definisali smo `theme` i `notificationsEnabled` kao podatke koje želimo da sačuvamo. Dodali smo i `_hasHydrated` - ovo je ključno za sledeći korak.
2.  **Akcije:** `toggleTheme` i `setNotifications` su funkcije koje menjaju stanje. `setHasHydrated` je pomoćna akcija.
3.  **Konfiguracija (`persist` opcije):**
      * `name`: Ovo je ključ (key) pod kojim će se tvoji podaci sačuvati u AsyncStorage. Možeš ga videti ako debuguješ memoriju telefona.
      * `storage: createJSONStorage(() => AsyncStorage)`: **Ovo je najvažniji deo za React Native.** `AsyncStorage` je asinhron, a `createJSONStorage` je omotač (wrapper) koji omogućava `persist` middleware-u da radi sa asinhronim "storage-om".
      * `onRehydrateStorage`: Ovo je opciona, ali veoma korisna funkcija. Ona se poziva **tačno u trenutku kada su podaci uspešno pročitani iz AsyncStorage-a i "ubrizgani" nazad u store**. Mi je koristimo da postavimo naš `_hasHydrated` flag na `true`.

-----

### Kako radi Hidratacija i zašto je `_hasHydrated` bitan

Kada se tvoja aplikacija pokrene, dešava se sledeće:

1.  JavaScript kod se učitava.
2.  Tvoj Zustand store se inicijalizuje sa početnim vrednostima (`theme: 'light'`, `_hasHydrated: false`).
3.  **U pozadini**, `persist` middleware asinhrono čita podatke iz `AsyncStorage` pod ključem `app-settings-storage`.
4.  Kada se čitanje završi, on **ažurira** store sačuvanim vrednostima (npr. `theme: 'dark'`). Ovo je **hidratacija**.
5.  Nakon toga, poziva se `onRehydrateStorage`, i naš `_hasHydrated` postaje `true`.

**Problem:** Postoji kratak vremenski period (delić sekunde) između koraka 2 i 4, gde tvoja aplikacija može prikazati početno stanje (`light` tema) pre nego što se učita sačuvano stanje (`dark` tema). Ovo može izazvati treperenje (flickering) na ekranu.

**Rešenje:** Koristimo `_hasHydrated` da prikažemo Splash Screen ili neki indikator učitavanja dok se hidratacija ne završi.

### Kompletan Primer u React Native Komponenti

Hajde da sve ovo povežemo u `App.js`.

```jsx
// App.js

import React, { useEffect } from 'react';
import { View, Text, Button, StyleSheet, ActivityIndicator } from 'react-native';
import useSettingsStore from './src/stores/useSettingsStore';

const App = () => {
  // Selektujemo _hasHydrated da znamo kada je store spreman
  const hasHydrated = useSettingsStore((state) => state._hasHydrated);

  if (!hasHydrated) {
    // Prikazujemo loading spinner dok se podaci ne učitaju iz AsyncStorage
    return (
      <View style={styles.container}>
        <ActivityIndicator size="large" />
        <Text>Loading settings...</Text>
      </View>
    );
  }

  // Kada je hidratacija gotova, renderujemo glavnu aplikaciju
  return <MainAppScreen />;
};

const MainAppScreen = () => {
  // Sada kada znamo da je store hidriran, možemo bezbedno koristiti njegove vrednosti
  const { theme, notificationsEnabled, toggleTheme, setNotifications } = useSettingsStore();

  const backgroundColor = theme === 'light' ? '#FFF' : '#333';
  const textColor = theme === 'light' ? '#000' : '#FFF';

  return (
    <View style={[styles.container, { backgroundColor }]}>
      <Text style={[styles.text, { color: textColor }]}>
        Current Theme: {theme}
      </Text>
      <Text style={[styles.text, { color: textColor }]}>
        Notifications: {notificationsEnabled ? 'Enabled' : 'Disabled'}
      </Text>
      
      <View style={styles.buttonContainer}>
        <Button title="Toggle Theme" onPress={toggleTheme} />
      </View>
      <View style={styles.buttonContainer}>
        <Button 
          title="Disable Notifications" 
          onPress={() => setNotifications(false)} 
          disabled={!notificationsEnabled}
        />
      </View>
    </View>
  );
};

const styles = StyleSheet.create({
  container: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
  },
  text: {
    fontSize: 20,
    marginBottom: 20,
  },
  buttonContainer: {
    marginVertical: 10,
  }
});

export default App;
```

**Kako ovaj primer radi:**

1.  `App` komponenta prvo proverava `hasHydrated`. U početku je `false`.
2.  Prikazuje se `ActivityIndicator` (loading spinner).
3.  U međuvremenu, Zustand u pozadini čita podatke iz `AsyncStorage`.
4.  Kada završi, `_hasHydrated` u store-u postaje `true`.
5.  `App` komponenta to detektuje (jer je "pretplaćena" na tu promenu) i ponovo se renderuje.
6.  Sada je uslov `!hasHydrated` netačan, i renderuje se `<MainAppScreen />`.
7.  `MainAppScreen` sada slobodno pristupa `theme` i `notificationsEnabled` vrednostima, siguran da su to poslednje sačuvane vrednosti, a ne početne. Svaki put kad pritisneš "Toggle Theme", novo stanje se automatski čuva u AsyncStorage-u. Ako zatvoriš i ponovo otvoriš aplikaciju, poslednja tema će biti učitana.


`shallow` u Zustandu je **funkcija za plitko poređenje** (shallow-equal). Koristi se kao `equalityFn` u `useStore` da bi sprečila nepotrebne re-render-e kada selektor vraća **objekat ili niz**.

Zašto je bitno?
Ako tvoj selektor na svako renderovanje kreira novi objekat/niz, React misli da je “novo” i re-renderuje, iako su unutrašnje vrednosti iste. `shallow` proverava **samo prvi nivo ključeva** i njihove reference (===). Ako su sve iste, komponenta se **ne** re-renderuje.

### Kako se koristi

**Varijanta 1 (klasično):**

```ts
import { shallow } from 'zustand/shallow';
import { useStore } from './store';

const MyComp = () => {
  const { bears, fish } = useStore(
    (state) => ({ bears: state.bears, fish: state.fish }),
    shallow
  );
  // re-render samo kad se promeni bears ili fish (po referenci/primitivu)
  ...
};
```

**Varijanta 2 (hook helper):**

```ts
import { useShallow } from 'zustand/react/shallow';
import { useStore } from './store';

const MyComp = () => {
  const { bears, fish } = useStore(
    useShallow((state) => ({ bears: state.bears, fish: state.fish }))
  );
  ...
};
```

### Kada je korisno

* Kada selektuješ **više delova stanja odjednom** i vraćaš objekat:

  ```ts
  useStore((s) => ({ a: s.a, b: s.b }), shallow);
  ```
* Kada selektor vraća **niz** (npr. `items.map(...)`) i želiš da izbegneš re-render dok se reference elemenata nisu promenile.
* Kada praviš **memoizovan view** nad stanjem koji vraća objekat istih referenci dok se ništa suštinski ne menja.

### Kada ti **ne treba**

* Ako selektuješ **primitiv** (`number`, `string`, `boolean`) ili već vraćaš **ista referenca** (npr. deo stanja koji se ne menja), dovoljno je podrazumevano `===`.
* Ako koristiš **više `useStore` poziva sa pojedinačnim selektorima**:

  ```ts
  const bears = useStore((s) => s.bears);
  const fish  = useStore((s) => s.fish);
  ```

  Ovo je često sasvim ok (svaki selektor je jeftin i precizan).

### Ograničenja / zamke

* `shallow` gleda **samo prvi nivo**. Ako vratiš `{ user: { name } }` i promeni se `user.name` bez promene reference `user` objekta, `shallow` to **neće primetiti**. Rešenje: ili menjaj referencu u store-u (immutability), ili selektuj dublje vrednosti:

  ```ts
  useStore((s) => ({ userName: s.user.name }), shallow);
  ```
* Ako tvoj selektor kreira **novi objekt sa novim referencama** pri svakoj promeni nebitnog dela, `shallow` pomaže samo ako se vrednosti stvarno nisu promenile. I dalje pazi da u store-u menjaš stanje **immutably** (nove reference kad se nešto promeni).

### Kratak zaključak

* `shallow` = plitko poređenje rezultata selektora → manje nepotrebnih re-render-a kad vraćaš objekte/nizove.
* Koristi ga kad spajaš više polja u jedan objekat ili vraćaš niz; izbegavaj ga za trivijalne primitive.

Ako hoćeš, mogu da pogledam tvoj konkretan selektor/komponentu i predložim gde tačno da ubaciš `shallow`.

Kratko: **Ne.** Zustand **ne traži nikakav Provider/omotač** da bi radio. Store je obična funkcija/hook i možeš ga koristiti bilo gde u React/React Native aplikaciji bez globalnog context-a.

### Standardna upotreba (bez omotača)

```ts
// store.ts
import { create } from 'zustand';

type AppState = {
  isNotificationLink: boolean;
  onboardingFinished: boolean;
  setOnboardingFinished: (v: boolean) => void;
};

export const useAppStore = create<AppState>((set) => ({
  isNotificationLink: false,
  onboardingFinished: false,
  setOnboardingFinished: (v) => set({ onboardingFinished: v }),
}));

// bilo koja komponenta
const Comp = () => {
  const onboardingFinished = useAppStore(s => s.onboardingFinished);
  ...
};
```

### Kada **ipak** koristiti omotač (Provider)?

Nije obavezno, ali je korisno u specifičnim slučajevima:

1. **Više instanci istog store-a** (npr. po tabu, po widgetu, multi-tenant).
    
2. **SSR/Next.js per-request store** (svakom requestu daješ “svež” state da izbegneš deljenje).
    
3. **Testovi/Storybook** gde želiš izolovan state po priči/testu.
    

Primer sa kontekstom (scoped store):

```ts
// store-context.ts
import { create } from 'zustand';
import { createContext } from 'zustand/context';

type AppState = { count: number; inc: () => void };

const { Provider, useStore } = createContext<AppState>();

const createStore = () =>
  create<AppState>((set) => ({
    count: 0,
    inc: () => set((s) => ({ count: s.count + 1 })),
  }));

export const AppStoreProvider = ({ children }: { children: React.ReactNode }) => (
  <Provider createStore={createStore}>{children}</Provider>
);

export const useScopedAppStore = useStore;

// _app.tsx ili layout
// <AppStoreProvider>{children}</AppStoreProvider>
```

### Šta još da znaš

- `persist`, `devtools`, `subscribeWithSelector` **ne zahtevaju** nikakav Provider.
    
- U RN i u običnom React-u radi isto — bez omotača.
    
- Ako ne znaš da li ti treba: u 95% slučajeva **ne treba**. Dodaj Provider tek ako imaš konkretan razlog iz liste gore.
    

Ako želiš, mogu da ti pripremim Next.js (App Router) primer sa per-request store-om i hidracijom.

Zato što **Zustand store nije React komponenta** niti React-ov “custom hook”. Pravila hook-ova kažu: _React hookove (useEffect, useState, useMemo, useRef, kao i bilo koji tvoj custom hook koji ih koristi) smeš da pozivaš samo iz funkcionalnih komponenti ili drugih hook-ova_—ne i iz “obične” JS funkcije poput `create()` inicijalizatora Zustand-a. Zato:

- ❌ Ne može: pozivanje **bilo kog** React hooka (ili custom hooka koji interno koristi React hookove) **u telu** `create((set, get) => { ... })` ili unutar slice-a.
    
- ✅ Može: koristiti **Zustand hook** (`useStore(...)`) **u komponentama ili u tvojim React custom hookovima** (koji se onda koriste u komponentama).
    
- ✅ Može: raditi side-effects bez React hookova kroz `store.subscribe(...)` van Reacta ili kroz thunk akcije, ali to nisu React hookovi.
    

## Tipični antiprimer

```ts
// ❌ Ovo će “pasti” jer je useRouter React hook:
const useBoundStore = create((set, get) => {
  const router = useRouter(); // <-- React hook u store-u (zabranjeno)
  return { /* ... */ }
});
```

## Ispravan obrazac #1: koristi hookove u React sloju, ne u store-u

```ts
// store.ts
export const useBoundStore = create<State>()((set, get) => ({
  hasPremium: false,
  init: async (deps?: { posthog?: PostHog; router?: any }) => {
    // koristi deps, NE React hookove
    const { posthog } = deps || {};
    // ... uradi async, set(...)
  },
}));

// useInitBilling.ts (custom React hook)
export function useInitBilling() {
  const init = useBoundStore(s => s.init);
  const posthog = usePostHog();      // ✅ React hook sme ovde
  const router = useRouter();        // ✅ React hook sme ovde

  useEffect(() => {
    init({ posthog, router });       // prosledi zavisnosti store-u
  }, [init, posthog, router]);
}
```

U komponenti:

```tsx
function App() {
  useInitBilling();           // ✅ React custom hook koristi Zustand akcije
  const hasPremium = useBoundStore(s => s.hasPremium);
  return /* ... */
}
```

## Ispravan obrazac #2: derived state sa React hookovima – ali u custom hooku, ne u store-u

```ts
// ✅ Napravi React custom hook koji koristi Zustand selektore
export function usePaywallVariant() {
  const variant = useBoundStore(s => s.featureFlags.paywallVariant);
  // Po želji deriviraj
  return useMemo(() => variant ?? 'control', [variant]);
}
```

## Ispravan obrazac #3: side-effects bez React-a (van komponenti)

Ako želiš da reaguješ na promene stanja _izvan_ React-a:

```ts
import { useBoundStore } from './store';

const unsub = useBoundStore.subscribe(
  state => state.hasPremium,             // selector
  (now, prev) => {
    if (now && !prev) {
      // pokreni neki “service” kod (bez React hookova)
    }
  }
);

// kasnije: unsub()
```

## Gde ubaciti integracije (RevenueCat, PostHog, AppsFlyer…)?

- Drži ih u **services/** modulima (obične funkcije).
    
- Thunk akcije u Zustand-u samo pozivaju te servise i rade `set(...)`.
    
- Ako servisu trebaju React stvari (npr. router), **proslijedi ih kao argumente** iz custom React hooka (primer #1).
    

## Sažetak

- React hookovi (i tvoji custom hookovi koji ih koriste) **ne smeju** u `create(...)`/slice-ove → store se ne izvršava u React hook kontekstu.
    
- Hookove koristi u **React sloju** (komponente ili tvoji custom hookovi), a store neka bude čista JS logika + thunks.
    
- “Spajanje” sa spoljnim SDK-ovima radi preko **services** funkcija i/ili `store.subscribe`, ne preko React hookova unutar store-a.
    

Ako hoćeš, mogu ti prepakovati tvoja tri providera u jedan Zustand sa slice-ovima i pokazati gde tačno idu `useEffect`/`usePostHog`/`Purchases.*` delovi.

Super kratko:

- `subscribe` u Zustand-u služi da **slušaš promene stanja bez renderovanja React komponenti**. To je “van-React” mehanizam za side-effects, logging, sinkronizaciju sa SDK-ovima, itd.
    
- **Ne** možeš koristiti **PostHog React hookove** (`usePostHog`) unutar `subscribe`, jer `subscribe` nije u React hook kontekstu. Ali možeš koristiti **PostHog klijent** (objekat), ako ga učiniš dostupnim van React-a.
    

---

## Kako `subscribe` radi

- Bez selektora (sluša celu state promenu):
    

```ts
const unsub = useStore.subscribe((state, prevState) => {
  // svaka promena bilo čega u store-u
})
```

- Sa selektorom (sluša samo deo):
    

```ts
const unsub = useStore.subscribe(
  s => s.hasPremium,                        // selektor
  (now, prev) => {
    if (now && !prev) {
      // reaguj samo kada hasPremium pređe sa false -> true
    }
  },
  { equalityFn: Object.is, fireImmediately: false } // opcionalno
)
```

- Selektor može vratiti tuple za višestruke vrednosti + `shallow`:
    

```ts
import { shallow } from 'zustand/shallow'

useStore.subscribe(
  s => [s.hasPremium, s.paywallVariant] as const,
  ([nowPremium, nowVariant], [prevPremium, prevVariant]) => {
    // reaguj kada se bilo šta od ta dva promeni
  },
  { equalityFn: shallow }
)
```

> `subscribe` **ne renderuje UI**. Za UI koristiš `useStore(selector)` u komponentama.

---

## Kako (ne) koristiti PostHog u `subscribe`

- ❌ **Ne**: pozivati `usePostHog()` unutar `subscribe` callbacka (to je React hook).
    
- ✅ **Da**: koristi **PostHog klijent** koji je dostupan van React-a.
    

### Pattern A — servis + setter iz React-a

**services/analytics.ts**

```ts
import type { PostHog } from 'posthog-react-native'

let ph: PostHog | null = null

export const setPostHogClient = (client: PostHog | null) => {
  ph = client
}

export const capture = (event: string, props?: Record<string, any>) => {
  ph?.capture(event, props)
}
```

**u nekoj root komponenti (React sloj)**:

```tsx
import { usePostHog } from 'posthog-react-native'
import { useEffect } from 'react'
import { setPostHogClient } from '@/services/analytics'

function AppBootstrap() {
  const posthog = usePostHog()

  useEffect(() => {
    setPostHogClient(posthog)         // injektuj klijent u servis (van React-a)
    return () => setPostHogClient(null)
  }, [posthog])

  return null
}
```

**subscribe negde pri boot-u (van React-a ili u još jednom useEffect-u):**

```ts
import { useStore } from '@/store'
import { capture } from '@/services/analytics'

const unsub = useStore.subscribe(
  s => s.hasPremium,
  (now, prev) => {
    if (now && !prev) capture('user_upgraded', { source: 'billingSlice' })
  }
)

// kasnije: unsub()
```

### Pattern B — proslediš klijent kao argument akciji

Ako ideš akcijom (thunk) iz React-a:

```ts
// slice
purchaseProduct: async (product, deps?: { posthog?: PostHog }) => {
  const ok = await Billing.purchase(product)
  if (ok && deps?.posthog) deps.posthog.capture('purchase', { id: product.identifier })
  set({ hasPremium: ok })
}

// komponenta
const purchase = useStore(s => s.purchaseProduct)
const ph = usePostHog()

await purchase(product, { posthog: ph })
```

---

## Kada koristiti `subscribe`

- Slanje događaja ka PostHog/Firebase/AppsFlyer pri promeni stanja.
    
- Sinkronizacija ograničenja/flagova (npr. kad `hasPremium` postane `true`, resetuj dnevni limit).
    
- Debouncing/persist custom logike (mada za storage koristi `persist` middleware).
    

## Kada **ne** koristiti `subscribe`

- Za UI reakcije (render) — koristi `useStore(selector)`.
    
- Za logiku koja zavisi od React ciklusa (router, fokus ekrana, lifecycle) — uradi to u **custom React hookovima** i/ili prosledi zavisnosti u thunks.
    

---

### TL;DR

`subscribe` je “van-React” slušalac promena u Zustand-u. Ne možeš da koristiš PostHog **hookove** unutra, ali možeš PostHog **klijent** (preko servisa/singletona ili prosleđivanjem iz React-a). To je idealno mesto za event logging i druge side-effects koji ne treba da diraju render.

U Zustand-u **“set”** (ili često `setState`) se ne _čita_ — on je **funkcija kojom menjaš stanje u store-u**, a ne vrednost koju bi čitao.  
Ali ako želiš da joj pristupiš (recimo, da je koristiš u nekom spoljašnjem modulu), evo kako:

---

### ✅ Primer 1: Čitanje `set` unutar samog store fajla

```ts
import { create } from 'zustand';

interface CounterState {
  count: number;
  increment: () => void;
}

const useCounterStore = create<CounterState>((set) => ({
  count: 0,
  increment: () => set((state) => ({ count: state.count + 1 })),
}));

export default useCounterStore;
```

Ovde `set` postoji **samo unutar** callback-a koji prosleđuješ `create()`.  
Znači – čitaš ga samo tu, jer je to deo interne logike store-a.

---

### ✅ Primer 2: Ako hoćeš da koristiš `set` **izvan hook-a** (npr. iz util fajla)

Zustand ti dozvoljava da pozoveš `useStore.setState(...)` direktno, jer `useStore` nije samo hook — to je i sam store objekat:

```ts
import useCounterStore from '@/store/useCounterStore';

// direktno menja stanje:
useCounterStore.setState({ count: 10 });

// ili koristi callback:
useCounterStore.setState((state) => ({ count: state.count + 1 }));
```

👉 Ovo je isto kao `set` unutra, ali dostupan spolja.

---

### ✅ Primer 3: Ako želiš i da “čitaš” trenutno stanje (bez React hooka)

Možeš da koristiš:

```ts
const current = useCounterStore.getState();
console.log(current.count);
```

Znači:

- `useCounterStore.getState()` → čita trenutno stanje (bez re-rendera)
    
- `useCounterStore.setState()` → menja stanje
    
- `useCounterStore.subscribe()` → sluša promene
    

---

### 📘 Ukratko

|Akcija|Funkcija|
|---|---|
|Čitanje vrednosti u React komponenti|`const count = useCounterStore(state => state.count)`|
|Čitanje vrednosti van React-a|`useCounterStore.getState().count`|
|Promena stanja izvan React-a|`useCounterStore.setState({ count: 5 })`|
|Subscribovanje na promene|`useCounterStore.subscribe(console.log)`|

---

Hoćeš da ti pokažem konkretno kako da izvan komponente (npr. u API call-u ili util funkciji) koristiš `set` iz nekog tvog konkretnog store-a (npr. `useExamStore`)?


Super 👍  
Evo konkretan primer na osnovu tvog **`useExamStore`** (kao što si ga koristio ranije):

---

### 🔹 Primer store-a (`useExamStore.ts`)

```ts
import { create } from 'zustand';

interface ExamState {
  examId: string | null;
  setExamId: (id: string) => void;
}

const useExamStore = create<ExamState>((set) => ({
  examId: null,
  setExamId: (id) => set({ examId: id }),
}));

export default useExamStore;
```

---

### 🔹 Kako da **promeniš** `examId` izvan React komponente

Recimo da imaš neku funkciju u `utils/examUtils.ts` i želiš da postaviš novi ID:

```ts
import useExamStore from '@/store/useExamStore';

export const selectExam = (id: string) => {
  useExamStore.setState({ examId: id });
};
```

👉 Ovo **radi isto kao da si pozvao `setExamId(id)`** iz komponente.

---

### 🔹 Kako da **pročitaš** trenutno stanje izvan komponente

Ako ti treba da vidiš koji je trenutni `examId` bez hook-a:

```ts
import useExamStore from '@/store/useExamStore';

export const getCurrentExamId = () => {
  const id = useExamStore.getState().examId;
  return id;
};
```

Ovo ne izaziva re-render — čisto čita vrednost direktno iz memorije.

---

### 🔹 Kratak rezime razlika

|Akcija|Gde se koristi|Sintaksa|
|---|---|---|
|Čitanje u React komponenti|unutar komponente|`const examId = useExamStore(state => state.examId)`|
|Čitanje izvan React-a|npr. u utils fajlu|`useExamStore.getState().examId`|
|Menjanje izvan React-a|npr. u service-u|`useExamStore.setState({ examId: '1234' })`|
|Subscribovanje na promene|npr. logovanje ili sync|`useExamStore.subscribe(console.log)`|

---

Hoćeš da ti pokažem i kako da pomoću `subscribe` automatski reaguješ kad se `examId` promeni (npr. da se loguje u PostHog ili pošalje API poziv)?