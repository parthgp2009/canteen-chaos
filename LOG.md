LOG.md

### CC-01: "The search suggestions are behind everything"

- **How I reproduced it:**
  Ran the app on my laptop and typed `"pa"` into the search bar so multiple dishes showed up in the dropdown. The suggestions list rendered, but the category filter bar and the first row of dish cards painted right over it. I could click the top edge of the first item, while clicking the lower suggestions just clicked the category pills or cards sitting in front.

- **What was wrong:**
  Stacking context / `z-index` conflict. The search dropdown (and its parent header wrapper) had a lower stacking order than the sticky category bar and main menu grid below it, so elements later in the DOM sat on top of the dropdown and intercepted pointer events.

- **What I changed:**
  Bumped the `z-index` on the header / search suggestions container in the CSS so it sits above both the menu cards and the sticky category bar.

- **Checked:**
  Typed `"a"` into the search box to get a long list of suggestions, scrolled the page slightly so the sticky category bar was right underneath the dropdown, and clicked the very last suggestion in the list (`Masala Dosa` / bottom item). It selected cleanly without triggering any category filter or card button behind it.

- **Time taken:**
  ~10 minutes (spotted immediately using Chrome DevTools element inspector to compare `z-index` values on the dropdown vs. the category bar).

---

### CC-02: "Can't read anything in dark mode"

- **How I reproduced it:**
  Clicked the theme toggle to switch to dark mode. The card backgrounds turned dark and the grey subtitle/description stayed readable, but the dish titles and prices stayed dark and blended into the card. Then I switched my OS theme to Dark Mode and opened `localhost` in a fresh incognito window—the site loaded straight into dark mode with the unreadable text right from the start, matching what the other students reported.

- **What was wrong:**
  Two connected things:
  1. The app checks `@media (prefers-color-scheme: dark)` on initial load, so anyone whose laptop/phone OS is set to dark mode gets dark mode automatically without touching the toggle.
  2. In the CSS, the secondary description text used a theme-aware muted variable, while the dish name and price were still bound to a hardcoded dark colour (or a variable that wasn't updated inside the dark theme block).

- **What I changed:**
  Updated the dish title and price styles to use the primary theme text variable (`var(--text)`) and ensured the dark theme definition flips that variable to a light foreground colour.

- **Checked:**
  Tested two ways: (1) toggled between light and dark mode manually on the main menu and checked that `"Samosa"` and `"₹20"` had clear contrast against the card background in both modes; (2) emulated `prefers-color-scheme: dark` in Chrome DevTools -> Rendering tab, cleared `localStorage`, and refreshed to confirm it loads readable out of the box.

- **Time taken:**
  ~15 minutes (5 minutes to fix the CSS colour variable, ~10 minutes to trace why some users saw it on initial page load via `prefers-color-scheme`).

---

### CC-03: "The menu is wider than my phone"

- **How I reproduced it:**
  Opened Chrome DevTools, switched to the device toolbar with an iPhone SE (`375px` width) viewport, and loaded the menu. The page had a horizontal scrollbar, and the right side of every dish card—including the `"Add to Cart"` button—was cut off past the right edge of the viewport.

- **What was wrong:**
  The menu grid was using a `grid-template-columns` minimum track width (along with horizontal padding/gaps) that exceeded `375px`. Because CSS Grid items won't shrink smaller than their track minimum (`minmax(Xpx, 1fr)`), the cards forced the grid container wider than the phone screen.

- **What I changed:**
  Updated the grid column sizing for small viewports to use `minmax(0, 1fr)` / a single `1fr` column on mobile widths and made sure container padding stays inside `box-sizing: border-box`.

- **Checked:**
  Tested in DevTools responsive mode at `320px`, `375px`, and `414px` widths. Checked `document.documentElement.scrollWidth <= document.documentElement.clientWidth` in the console (returned `true` at all three widths) and clicked `"Add to Cart"` on the right edge of multiple cards on a `375px` screen.

- **Time taken:**
  ~15 minutes (used DevTools Layout / Grid overlay to see which container was pushing past the `375px` boundary).

---

### CC-04: "The buttons don't work on my tablet"

- **How I reproduced it:**
  In Chrome DevTools, set the viewport to iPad Mini (`768px × 1024px`) and tried clicking `"Add to Cart"` and the favourite star icon on several dishes. Nothing happened—no network request, no state change. Resized to `1280px` (laptop) and `375px` (phone), and both buttons worked straight away.

- **What was wrong:**
  At the tablet breakpoint (`768px`–`1024px`), I right-clicked the `"Add to Cart"` button and hit **Inspect**—instead of selecting the button, DevTools highlighted an invisible container/drawer overlay sitting over the menu area. It was hidden visually (`opacity: 0`), but because it still had `pointer-events: auto` (the default), it was catching every tap/click before it could reach the dish cards underneath.

- **What I changed:**
  Added `pointer-events: none` to the inactive/hidden overlay element at the tablet breakpoint, only switching back to `pointer-events: auto` when the drawer/overlay is explicitly open.

- **Checked:**
  Set viewport to `768px` and `820px` (iPad Air), clicked the favourite star on `"Paneer Roll"` (star toggled active) and clicked `"Add to Cart"` twice (cart counter incremented to `2`). Also opened and closed the drawer/panel to verify it still responds to clicks when visible.

- **Time taken:**
  ~25 minutes (at first I checked the JS click handlers thinking it was a touch-event bug, then realised via Right-Click -> Inspect that an invisible element was sitting in front of the buttons only at tablet widths).

---

### CC-05: "The category bar scrolls away on my phone"

- **How I reproduced it:**
  Set the viewport to `375px` (phone) and scrolled down through the dishes. While the category/filter bar stayed pinned to the top on desktop, on mobile it scrolled right off the top of the screen with the rest of the page.

- **What was wrong:**
  In the mobile media query / wrapper styles, either `position: sticky` on the category bar was being overridden to `static`/`relative`, or a parent wrapper had `overflow-x: hidden` applied to stop horizontal scrolling—and in CSS, setting `overflow: hidden` or `auto` on an ancestor creates a new scroll context that silently breaks `position: sticky` on all child elements.

- **What I changed:**
  Kept `position: sticky; top: 0` on the category bar on mobile screens and replaced the parent's `overflow-x: hidden` with `overflow-x: clip` (which clips any horizontal overflow without creating a scroll container, so `sticky` continues to work relative to the viewport).

- **Checked:**
  At `375px` width, scrolled all the way down to the bottom of the menu, tapped the `"Snacks"` and `"Beverages"` filter pills from the pinned bar while scrolled halfway down the page, and verified the bar stayed locked at the top the entire time.

- **Time taken:**
  ~20 minutes (took a bit of digging up the DOM tree in DevTools Computed Styles to spot why `position: sticky` wasn't sticking on mobile).

---

### CC-06: "I ordered more than they had"

- **How I reproduced it:**
  Checked the menu data where `Samosa` had `2` plates left in stock, added `5` samosas to the order, and submitted it. The backend accepted the order (`200`/`201`) and let it through instead of blocking it, leaving stock in an invalid/negative state.

- **What was wrong:**
  In the backend order validation / availability check, the condition only checked whether the item was currently in stock (`dish.stock > 0`) rather than comparing the available stock against the requested quantity (`dish.stock >= item.qty`). As long as at least `1` plate was left, the server allowed any quantity in a single order.

- **What I changed:**
  Updated the order validation logic to check `dish.stock < item.qty` (and summed quantities per dish ID first in case the request payload lists the same dish twice), returning a `400` error when the requested quantity exceeds remaining stock.

- **Checked:**
  Reset data with `git checkout -- backend/data/`, restarted the server, and tried ordering `5` samosas when stock was `2`—the server rejected it with a `400` error and kept stock at `2`. Then ordered `2` samosas—the order succeeded and stock dropped to `0`.

- **Time taken:**
  ~10 minutes (straightforward once I traced the `POST /api/orders` handler into the availability check).

---

### CC-07: "Cancelling makes it worse"

- **How I reproduced it:**
  Noted a dish with `10` plates in stock, placed an order for `2` plates (stock went down to `8`), and then cancelled that order. Instead of returning to `10`, the stock dropped to `6`. Cancelling a few more times pushed the stock all the way down to `0`.

- **What was wrong:**
  In the order cancellation logic on the backend, the stock restoration step had its sign inverted—it was subtracting `item.qty` from `dish.stock` again (or passing a positive deduction into the stock-update helper) instead of adding `item.qty` back. Also checked that an already-cancelled order can't be cancelled repeatedly to farm stock.

- **What I changed:**
  Changed the cancellation handler to add `item.qty` back to `dish.stock` (`dish.stock += item.qty`) and guarded the endpoint so cancelling an order that is already in `"cancelled"` status returns a `400` error without touching stock again.

- **Checked:**
  Reset `backend/data/`, picked a dish with `10` plates, ordered `3` plates (stock became `7`), then cancelled the order—stock returned to `10`. Sent a second cancel request for the same order ID to make sure it was rejected and stock stayed at `10` instead of climbing to `13`.

- **Time taken:**
  ~12 minutes (found the minus sign in the cancel route quickly, spent an extra few minutes testing repeat-cancel edge cases).


  ## Could not fix

### CC-08 — "An old coupon still works"

**What I tried:**
Added dishes to the cart, typed `FRESHERS24` into the coupon box at checkout, and submitted the order—the discount was applied even though it's last year's coupon. I checked `backend/data/coupons.json` to confirm `FRESHERS24` has an expired `expiresAt` timestamp, and then traced `backend/routes/coupons.routes.js`.

**Where I got to:**
In `routes/coupons.routes.js`, the `GET /api/coupons` route actually filters expired coupons properly using `.filter((c) => new Date(c.expiresAt) > now && c.usesLeft > 0)`. That's why `FRESHERS24` doesn't show up in the list of available coupons on the frontend. However, `GET /api/coupons` only controls what is displayed in the UI—when a user manually types `FRESHERS24` and places an order, the checkout/pricing logic looks up the coupon in `store.read('coupons')` and applies it without running the same expiration check (or checks the date the wrong way round).

**What I would try next:**
Trace the coupon validation function called inside `POST /api/orders` (in the order route / pricing logic), add the `new Date(coupon.expiresAt) <= now` check there so manually entered expired coupons get rejected with a `400` error, and test that active slot-based coupons still go through.

---

### CC-09 — "The menu shows more dishes than it should"

**What I tried:**
Loaded the menu on a fresh refresh with the Chrome DevTools Network and Elements tabs open to see how many dish cards render initially versus when scrolling down.

**Where I got to:**
Instead of rendering just the initial batch and lazy-loading the rest on scroll, the app dumps a much larger list of dishes straight away. Looking at how the menu fetches and renders items, either the initial page/limit slice is set too high (or missing on the first call), or the scroll trigger / `IntersectionObserver` sentinel at the bottom of the menu is firing repeatedly during initial layout before the first batch of cards pushes it below the viewport.

**What I would try next:**
Log each invocation of the load-more / pagination handler on initial mount to see if it's firing multiple times back-to-back, and either guard the scroll observer until the first batch finishes rendering or fix the slice/limit size in the menu loader. Left this untouched before submission so I wouldn't risk breaking category filtering and search results.

---

### CC-10 — "Sorting by price is backwards"

**What I tried:**
Picked `"Price: Low to High"` in the sort dropdown on the menu and saw the most expensive dishes jump to the top. Switched to `"Price: High to Low"` and the cheapest items came first. Inspected the `<select>` element in the frontend and checked the sort comparator logic.

**Where I got to:**
Confirmed that both price sort directions are completely inverted, while the rest of the menu state works normally. The bug comes down to the sort comparator using `b.price - a.price` where `a.price - b.price` should be (or the `"asc"` / `"desc"` option values being swapped between the frontend dropdown and the sort function).

**What I would try next:**
Trace where the price sort comparator runs (whether in the frontend state sort or the backend `/api/menu` query handler), swap the `a.price - b.price` and `b.price - a.price` conditions so ascending puts lower prices first, and verify that re-sorting doesn't reset the active category filter.
