Super! Krenuću praktično i detaljno, ali jasno — od nule do naprednih šablona. Sve je na srpskom, sa TypeScript primerima (možeš i JS, samo izbaci tipove).

# 1) Šta je React Context (i kada ga koristiti)

**Context** služi da deliš podatke kroz stablo komponenti bez “prop drilling-a” (prosleđivanje props kroz više nivoa).  
Tipični slučajevi:

- autentikacija (trenutni korisnik, token),
    
- tema (light/dark),
    
- jezik (i18n),
    
- globalna podešavanja/feature flags.
    

**Nemoj** ga koristiti kao generalnu zamenu za state menadžere u kompleksnim app-ovima (Redux/Zustand/Recoil). Context je super za **relativno statične** ili **nisko-frekventne** podatke.

---

# 2) Osnovni tok: `createContext` + Provider + `useContext`

## Minimalan primer (tema)

```tsx
// theme-context.tsx
import { createContext, useContext, useMemo, useState, ReactNode } from "react";

type Theme = "light" | "dark";

type ThemeContextValue = {
  theme: Theme;
  toggle: () => void;
};

const ThemeContext = createContext<ThemeContextValue | undefined>(undefined);

// Provider
export function ThemeProvider({ children }: { children: ReactNode }) {
  const [theme, setTheme] = useState<Theme>("light");
  const toggle = () => setTheme((t) => (t === "light" ? "dark" : "light"));

  // useMemo da ne pravimo novi objekat na svaki render bez potrebe
  const value = useMemo(() => ({ theme, toggle }), [theme]);

  return <ThemeContext.Provider value={value}>{children}</ThemeContext.Provider>;
}

// Custom hook (ergonomija + sigurnost)
export function useTheme() {
  const ctx = useContext(ThemeContext);
  if (!ctx) throw new Error("useTheme mora biti korišćen unutar <ThemeProvider>");
  return ctx;
}
```

Korišćenje:

```tsx
// App.tsx
import { ThemeProvider, useTheme } from "./theme-context";

function ToggleButton() {
  const { theme, toggle } = useTheme();
  return (
    <button onClick={toggle}>
      Tema: {theme} (klikni za promenu)
    </button>
  );
}

export default function App() {
  return (
    <ThemeProvider>
      <ToggleButton />
    </ThemeProvider>
  );
}
```

---

# 3) Uobičajene greške i kako da ih izbegneš

1. **Zaboravljen Provider** → `useContext` vrati `undefined`. Reši custom hook-om koji baca jasan error (kao gore).
    
2. **Re-render oluja** → Ako u `value` proslediš `{…}` direktno bez `useMemo`, svako re-renders izaziva promenu reference → svi potrošači se rerenderuju.
    
3. **Preširok kontekst** → Ako u jednom contextu držiš puno stanja koje se često menja, _sve_ komponente koje koriste taj context renderuju ponovo. Rešenje: podeli na više konteksta (npr. ThemeContext i LocaleContext odvojeno) ili koristi **selector pattern** (naprednije, niže).
    
4. **Čuvanje funkcija koje se stalno menjaju** → memoizuj handler-e pomoću `useCallback` (ili ih uključi u `useMemo` vrednost kao u primeru).
    

---

# 4) Context + useReducer (odlično za “mini-store”)

Kada imaš više povezanih akcija i stanja:

```tsx
// auth-context.tsx
import { createContext, useContext, useMemo, useReducer, ReactNode } from "react";

type User = { id: string; name: string } | null;

type State = { user: User; loading: boolean; error?: string };
type Action =
  | { type: "LOGIN_START" }
  | { type: "LOGIN_SUCCESS"; payload: User }
  | { type: "LOGIN_ERROR"; payload: string }
  | { type: "LOGOUT" };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case "LOGIN_START":
      return { ...state, loading: true, error: undefined };
    case "LOGIN_SUCCESS":
      return { user: action.payload, loading: false, error: undefined };
    case "LOGIN_ERROR":
      return { ...state, loading: false, error: action.payload };
    case "LOGOUT":
      return { user: null, loading: false, error: undefined };
    default:
      return state;
  }
}

type AuthContextValue = {
  state: State;
  login: (username: string, password: string) => Promise<void>;
  logout: () => void;
};

const AuthContext = createContext<AuthContextValue | undefined>(undefined);

export function AuthProvider({ children }: { children: ReactNode }) {
  const [state, dispatch] = useReducer(reducer, { user: null, loading: false });

  async function login(username: string, password: string) {
    dispatch({ type: "LOGIN_START" });
    try {
      // API poziv simulacija
      await new Promise((r) => setTimeout(r, 600));
      dispatch({ type: "LOGIN_SUCCESS", payload: { id: "1", name: username } });
    } catch (e) {
      dispatch({ type: "LOGIN_ERROR", payload: "Neuspela prijava" });
    }
  }
  function logout() {
    dispatch({ type: "LOGOUT" });
  }

  const value = useMemo(() => ({ state, login, logout }), [state]);

  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
}

export function useAuth() {
  const ctx = useContext(AuthContext);
  if (!ctx) throw new Error("useAuth mora biti korišćen unutar <AuthProvider>");
  return ctx;
}
```

Korišćenje:

```tsx
function Header() {
  const { state, logout } = useAuth();
  return (
    <div>
      {state.user ? (
        <>
          <span>Ćao, {state.user.name}</span>
          <button onClick={logout}>Odjava</button>
        </>
      ) : (
        <span>Nisi prijavljen</span>
      )}
    </div>
  );
}
```

---

# 5) Podela konteksta (smanjenje re-rendera)

Ako u jednom kontekstu držiš i **state koji se često menja** i **stabilne vrednosti**, podeli:

```tsx
// cart-context.tsx
import { createContext, useContext, useMemo, useReducer } from "react";

type Item = { id: string; title: string; qty: number };

type CartState = { items: Item[] };
type CartAction = { type: "ADD"; item: Item } | { type: "REMOVE"; id: string };

function cartReducer(state: CartState, action: CartAction): CartState {
  switch (action.type) {
    case "ADD": return { items: [...state.items, action.item] };
    case "REMOVE": return { items: state.items.filter(i => i.id !== action.id) };
    default: return state;
  }
}

const CartStateCtx = createContext<CartState | undefined>(undefined);
const CartActionsCtx = createContext<
  { add: (item: Item) => void; remove: (id: string) => void } | undefined
>(undefined);

export function CartProvider({ children }: { children: React.ReactNode }) {
  const [state, dispatch] = useReducer(cartReducer, { items: [] });

  const actions = useMemo(
    () => ({
      add: (item: Item) => dispatch({ type: "ADD", item }),
      remove: (id: string) => dispatch({ type: "REMOVE", id }),
    }),
    []
  );

  return (
    <CartActionsCtx.Provider value={actions}>
      <CartStateCtx.Provider value={state}>{children}</CartStateCtx.Provider>
    </CartActionsCtx.Provider>
  );
}

export const useCartState = () => {
  const ctx = useContext(CartStateCtx);
  if (!ctx) throw new Error("useCartState mora u okviru CartProvider");
  return ctx;
};
export const useCartActions = () => {
  const ctx = useContext(CartActionsCtx);
  if (!ctx) throw new Error("useCartActions mora u okviru CartProvider");
  return ctx;
};
```

Sada komponente koje samo čitaju stanje ne renderuju se kad se promene samo akcije, i obrnuto.

---

# 6) Selector pattern (napredno)

Problem: svaka promena konteksta tera **sve** potrošače da se renderuju.  
Rešenja:

- podela konteksta (kao gore),
    
- biblioteka **`use-context-selector`** (uzmi samo deo vrednosti iz contexta, pa renderuješ samo kada se taj deo promeni).
    

Primer sa bibliotekom (konceptualno):

```tsx
// npm i use-context-selector
import { createContext, useContextSelector } from "use-context-selector";
// ... kreiraš kontekst i umesto useContext koristiš useContextSelector((v) => v.deo)
```

Kada app postane veća i performance bitan, ovo ume da bude zlata vredno.

---

# 7) SSR/Next.js 14/15 napomene (pošto često radiš Next)

- Kontekste koji treba da budu **client-side** moraš deklarisati u datotekama sa `"use client"`.
    
- Ako koristiš i18n/tema/korisnik na nivou cele aplikacije, postavi Providere u **root layout** (npr. `app/layout.tsx` ili u `app/[locale]/layout.tsx` ako radiš lokalizaciju).
    
- Za inicijalne vrednosti koje stižu sa servera (npr. user iz kolačića), prosledi ih Provideru kroz props (SSR → Client).
    

Primer skeletona:

```tsx
// app/providers.tsx
"use client";
import { ThemeProvider } from "@/context/theme-context";
import { AuthProvider } from "@/context/auth-context";

export default function Providers({ children }: { children: React.ReactNode }) {
  return (
    <AuthProvider>
      <ThemeProvider>{children}</ThemeProvider>
    </AuthProvider>
  );
}

// app/layout.tsx (server)
import Providers from "./providers";
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="sr">
      <body>
        <Providers>{children}</Providers>
      </body>
    </html>
  );
}
```

---

# 8) Testiranje contexta (React Testing Library)

```tsx
// theme-context.test.tsx
import { render, screen, fireEvent } from "@testing-library/react";
import { ThemeProvider, useTheme } from "./theme-context";

function Btn() {
  const { theme, toggle } = useTheme();
  return <button onClick={toggle}>Tema: {theme}</button>;
}

test("menja temu", () => {
  render(
    <ThemeProvider>
      <Btn />
    </ThemeProvider>
  );
  expect(screen.getByText(/Tema: light/)).toBeInTheDocument();
  fireEvent.click(screen.getByText(/Tema:/));
  expect(screen.getByText(/Tema: dark/)).toBeInTheDocument();
});
```

---

# 9) Kada **ne** koristiti Context

- Visoko-frekventne, granularne promene (npr. kursor miša, animacije, editor teksta) → bolje lokalni state ili specijalizovan store.
    
- Složen “globalni” state sa derivacijama/selectorima → razmisli o Redux Toolkit/Zustand/Jotai.
    

---

# 10) Brzi “cheat-sheet”

- Napravi context:
    
    ```ts
    const XCtx = createContext<X | undefined>(undefined);
    ```
    
- Napravi Provider sa **stabilnim `value`**:
    
    ```tsx
    const value = useMemo(() => ({ a, b }), [a, b]);
    return <XCtx.Provider value={value}>{children}</XCtx.Provider>;
    ```
    
- Napravi **custom hook**:
    
    ```ts
    export function useX() {
      const ctx = useContext(XCtx);
      if (!ctx) throw new Error("useX u okviru XProvider");
      return ctx;
    }
    ```
    
- Podeli state i akcije u odvojene kontekste kad treba.
    
- Za performance: podela, memo, ili `use-context-selector`.
    

---

# 11) Mini-vežbe (za učenje)

1. **LocaleContext**: `locale` = "sr" | "en", akcija `setLocale`. Dodaj `<LanguageSwitcher/>` komponentu.
    
2. **SnackbarContext**: globalni `show(message, type)` koji renderuje snackbar; koristi podelu (akcije odvojeno od stanja prikaza).
    
3. **FeatureFlagsContext**: niz booleana; napravi hook `useFeature("betaSearch")` koji vraća samo jedan flag (selector pattern ili podela).
    
4. **Cart**: implementiraj iznad dati reducer + prikaz totalne količine bez re-rendera celog kataloga (razdvajanje contexta).
    

---

Ako želiš, mogu da ti pripremim **starter repo strukturu** (Next.js 15, `app/`, sa `Providers`, temom, auth mini-store, i i18n skeletonom) ili da prilagodim primere baš tvom projektu (npr. `next-intl`, `/[locale]` rute, SSR user). Samo reci šta da uključim.