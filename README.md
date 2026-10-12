# 🍽️ Restaurant App MVP (React Native + Node.js + Express + MongoDB)

Customers browse the menu, search, build a cart, apply promo codes, reserve a table and place Dine-in / Takeaway orders and watch them progress. Managers handle incoming orders, reservations and the menu. A1 = frontend only (mock data). **A2 (this version)** adds the Express server and MongoDB.

> Demo video (max 4 min): **ADD LINK HERE**  
> Screenshots: put them in `docs/screenshots/` and list them here.

## Folder structure

```
restaurant-app-mvp/
├── A1/            SRS.pdf, UML/ (use case, class, sequence, state, component)
├── A2/            SRS_v2.pdf, diagrams/, restaurant-api.postman_collection.json
├── server/        config, models, routes, controllers, middleware, server.js, seed.js
├── src/           api, components, screens, context, reducers, hooks, navigation, theme, utils, constants
├── App.js
└── README.md
```

## 1. Run the server

Requirements: **Node.js 20 LTS or newer**, and MongoDB (free Atlas cluster, or MongoDB Community + Compass).

```bash
cd server
npm install
cp .env.example .env        # then fill in the values (Windows: copy .env.example .env)
```

`.env` contents:

```
PORT=5000
MONGO_URI=mongodb+srv://<user>:<password>@<cluster>/restaurant_app   # or mongodb://127.0.0.1:27017/restaurant_app
JWT_SECRET=any-long-random-string
```

On Atlas: *Network Access → add your current IP*. Then:

```bash
npm run seed     # empties the collections and inserts users, 17 menu items, 6 tables
npm run dev      # nodemon, http://localhost:5000/api/health
```

> ⚠️ Never commit `.env`. Only `.env.example` is committed.

## 2. Run the app

```bash
# in the project root
npm install
npx expo install --fix      # aligns package versions with your Expo Go / SDK version
npx expo start
```

Scan the QR code with **Expo Go** (phone and laptop on the **same Wi-Fi**), or press `a` for the Android emulator / `i` for the iOS simulator.

### Where to change BASE_URL
`src/api/client.js` (the only place). By default the app uses the IP of the laptop running Expo, so it normally works without changes. If you see **"Network request failed"**:

1. Is the server running (`npm run dev`)? Open `http://<laptop-IP>:5000/api/health` in the phone browser.
2. Set `MANUAL_HOST = '192.168.x.x'` in `src/api/client.js` (Android emulator: `'10.0.2.2'`).
3. Allow port 5000 in the laptop firewall. Your laptop IP may change when you change Wi-Fi.

## 3. Login details (created by `npm run seed`)

| Role | Email | Password |
|------|-------|----------|
| Customer | customer@test.com | customer123 |
| Manager | manager@test.com | manager123 |

You can also sign up from the app (choose Customer or Manager).

## 4. Hooks used on each screen

| Screen / file | Hooks |
|---------------|-------|
| LoginScreen | `useState` (mode, showPassword, isSubmitting), `useForm` (custom), `useCallback` (validate), `useAuth` → `useContext` |
| MenuScreen | `useState`, `useEffect` (load with abort cleanup, header title, focus refresh, recent searches), `useRef` (TextInput, FlatList, previous query, render counter), `useMemo` (filter + sort), `useCallback` (handlers), `useDebounce` (custom), `useCart` → `useContext` |
| MenuItemCard | `React.memo`, `useState` (image fallback), `useTheme` |
| CartScreen | `useReducer` via `useCart`, `useMemo` (totals), `useCallback`, `useState` (promo input) |
| OrderSummaryScreen | `useMemo` (totals, pickup options), `useCallback` (place order), `useState`, `useApi` (custom: tables) |
| ReservationScreen | `useReservation` (custom: all logic), `useMemo` (date chips), `useState` (modal) |
| useReservation | `useState`, `useMemo`, `useCallback`, `useApi` |
| MyOrdersScreen | `useApi` (polling every 10 s) |
| OrderTrackingScreen | `useEffect` + `setInterval` (poll server every 10 s, elapsed timer every 1 s, both cleared), `useState`, `useCallback` |
| ProfileScreen | `useAuth`, `useTheme`, `useCart` (`useContext`) |
| ManagerDashboardScreen + tabs | `useState` (tab), `useApi` (polling), role-based tab through `useAuth` in the navigator |
| AuthContext | `useState`, `useEffect` (restore session from AsyncStorage), `useCallback`, `useMemo` |
| ThemeContext | `useState`, `useEffect` (load saved theme), `useCallback`, `useMemo` |
| CartContext | `useReducer`, `useMemo` |
| AppNavigator | `useContext` hooks (`useAuth`, `useTheme`, `useCart`), `useSafeAreaInsets` |

## 5. API endpoints (base `http://<laptop-IP>:5000`)

| Method | Endpoint | Access | Request body | Response |
|--------|----------|--------|--------------|----------|
| GET | /api/health | Public | – | `{status, time}` |
| POST | /api/auth/register | Public | name, email, password, role | 201 token + user (409 duplicate email) |
| POST | /api/auth/login | Public | email, password | token + user (401 wrong) |
| GET | /api/auth/me | Auth | – | user |
| GET | /api/menu | Public | – (`?category=&search=`) | menu items |
| GET | /api/menu/:id | Public | – | item (404 / 400 bad id) |
| POST | /api/menu | Manager | name, price, category, … | 201 item |
| PUT | /api/menu/:id | Manager | fields to change | item |
| DELETE | /api/menu/:id | Manager | – | message |
| GET | /api/tables | Public | – | tables |
| POST | /api/orders | Auth | items[{menuItem, quantity, note}], promoCode, type, tableNumber/pickupTime | 201 order (server-calculated total) |
| GET | /api/orders/my | Auth | – | own orders |
| GET | /api/orders | Manager | – | all orders |
| PATCH | /api/orders/:id/status | Manager (owner may Cancel) | status | order (400 invalid transition) |
| GET | /api/reservations/availability | Auth | – (`?date=&partySize=`) | slots with free tables |
| POST | /api/reservations | Auth | tableId, date, time, partySize, contactName, phone | 201 (409 table taken) |
| GET | /api/reservations/my | Auth | – | own reservations |
| GET | /api/reservations | Manager | – | all reservations |
| PATCH | /api/reservations/:id | Manager | status Accepted/Declined | reservation |
| DELETE | /api/reservations/:id | Owner | – | reservation (status Cancelled) |

Import `A2/restaurant-api.postman_collection.json` into Postman. Run **01 Auth** first: it saves the customer and manager tokens, then every other folder works.

## 6. Notes required by the assignment

### A1 Q4 — empty dependency array in the filtering effect
If the effect that updates `filteredItems` had an empty array `[]`, it would run only once, right after the first render, when `menuItems` is still empty. The list would therefore stay empty: later changes to `selectedCategory` or to the loaded `menuItems` would never trigger it, so category chips would appear to do nothing. The dependency array tells React which values the effect "reads"; leaving one out creates a stale closure that keeps using the first render's values. (In the final code this effect was replaced by `useMemo`, see Q8.)

### A1 Q6 — why Context instead of prop drilling
The logged-in user and the theme are needed by almost every screen, including deeply nested ones. Passing them as props through every navigator and component would clutter code that does not use them and break easily when a screen is added. Context lets any component read `useAuth()` or `useTheme()` directly, so there is one source of truth and login/logout or the dark-mode switch updates the whole app instantly. The custom hooks also throw a clear error if a provider is missing. **Drawback:** every component that consumes a context re-renders whenever the context value changes, even if it uses only a part of it.

### A1 Q7 — `useReducer` vs `useState` for the cart
The cart has several related values (`items`, `promoCode`, `discountPercent`) and many transitions (add, remove, increment, decrement, note, promo, clear). With `useReducer` all rules live in one pure function that is easy to read and unit-test, and screens only dispatch intent (`INCREMENT`), so no two screens can update the cart in different ways. With `useState` each screen would need its own update logic, and related fields could get out of sync. `useState` would have been enough if the cart were just a single list with only add/remove and no promo or notes.

### A1 Q7 — reducer test cases (`src/reducers/__tests__/cartReducer.test.js`, run with `npm test`)

| # | Action | Initial state | Expected state |
|---|--------|---------------|----------------|
| 1 | `ADD_ITEM` (Biryani) | empty cart | one item, `quantity: 1`, `note: ''` |
| 2 | `ADD_ITEM` (Biryani) | cart has Biryani ×1 | still one line, `quantity: 2` |
| 3 | `INCREMENT` m1 | Biryani ×1 | `quantity: 2` |
| 4 | `DECREMENT` m1 | Biryani ×1 | `items: []` (removed at zero) |
| 5 | `UPDATE_NOTE` "no onions" | Biryani ×1 | `note: 'no onions'` |
| 6 | `APPLY_PROMO` "welcome10" | any | `promoCode: 'WELCOME10'`, `discountPercent: 10` |
| 7 | `APPLY_PROMO` "FREEFOOD" | any | `discountPercent: 0`, `promoError: 'Invalid promo code'` |
| 8 | `REMOVE_PROMO` | promo applied | `discountPercent: 0`, `promoCode: null` |
| 9 | `CLEAR_CART` | items + promo | initial state |
| 10 | any action on a frozen state | frozen state | no exception, old state unchanged (purity) |

### A1 Q8 — when NOT to use `useMemo` / `useCallback`
Do not wrap everything "just in case". Both hooks cost memory and comparison work on every render, and they add code that is harder to read. Skip them when the calculation is cheap (adding a few numbers, mapping a short list), when the value is passed only to plain DOM/native elements that re-render anyway, or when the component is not wrapped in `React.memo` and the function is not a dependency of another hook — a stable reference then gives no benefit. Measure first (React DevTools profiler or a `console.log`), optimise only the real hot spots: large lists, expensive derived data, and callbacks passed to memoised children, as done for `MenuItemCard`.

### A1 Q8 — proof that only one card re-renders
`MenuItemCard` contains `console.log('Render MenuItemCard:', item.name)`. Because the card is wrapped in `React.memo` and receives stable `useCallback` handlers, tapping a heart logs only the toggled card's name (screenshots: `docs/screenshots/before-memo.png`, `after-memo.png`). To see the "before" case, temporarily remove `React.memo` from the export in `MenuItemCard.js`.

### Why the server recalculates the total (A2 Q7)
The app only sends item ids and quantities. A modified app or Postman could send `total: 1`; the server therefore reads prices from MongoDB and calculates the total itself.

## 7. Two-device test script (for the video)

1. Phone A: log in as **customer**. Phone B (or emulator): log in as **manager**.
2. Manager → Dashboard → Menu Management: change a price. Customer: pull to refresh on Menu — new price appears.
3. Customer: search "biryani", add items, apply `WELCOME10`, open Order Summary, choose a table, **Place order**.
4. Manager: Incoming Orders → *Mark Preparing* → *Mark Ready*. Customer's Order Tracking updates within 10 s.
5. Customer: Reserve a table for tomorrow 19:00. Manager: Reservations → Accept. Customer sees *Accepted* after refresh.
6. Try to book the same table and time again → server answers 409.

## 8. Common errors

| Error | What to check |
|-------|---------------|
| Network request failed | Same Wi-Fi, server running, correct IP in `src/api/client.js`, firewall port 5000 |
| MongoServerSelectionError | Atlas Network Access IP, username/password in `MONGO_URI` |
| 401 on every request | Header must be `Authorization: Bearer <token>` |
| Worked yesterday, not today | Laptop IP changed |
