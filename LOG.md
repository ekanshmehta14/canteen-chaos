# Debug log

### CC-01: "The search suggestions are behind everything"

**Reproduced:** With suggestions open, the search wrapper was at stacking level 1 while the category strip was at level 40. The popup's own higher z-index could not escape its parent's lower stacking context.

**Cause:** The popup was confined to a lower stacking context than the category strip, allowing the strip to cover suggestions where they overlapped.

**Fix:** Raised the search wrapper above the category strip so suggestions can paint and receive clicks over following menu controls.

**Checked:** Live style inspection confirmed the search wrapper is at level 50, above the category strip at level 40. The suggestion center hit-tests to the suggestion button, and activating it opens the dish detail dialog.

**Time:** about 30 minutes.

### CC-02: "Can't read anything in dark mode"

**Reproduced:** Switched to Dark Theme and Dish name and price were not visible.

**Cause:** The card background color blended with the name and price colour causing to be invisible this wa due to line 294 of style.css where the colour of dish name and price was not declared as the variable specified initially in dark theme but was set to a specific colour due to which it remained the same even after changing the theme to dark .

**Fix:** The specific colour was changed to variable ink.

**Checked:** The name and price of dishes are now visible in dark theme.

**Time:** about 20 minutes, iniatially thought there was a mistake in colour defining in the dark theme defining part but later notice that the dish body had a specific colour mentioned instead of a variable.

### CC-03: "The menu is wider than my phone"

**Reproduced:** Inspected the responsive dish-card image rule used below 760px. It set images to `width: auto` and only constrained their minimum width, allowing intrinsic image width to exceed a narrow card.

**Cause:** Dish images were not capped to the card width in the mobile and tablet layout.

**Fix:** Set dish images to fill the card width while allowing them to shrink, with a 100% maximum width.

**Checked:** The responsive rule now uses `width: 100%`, `min-width: 0`, and `max-width: 100%`. The embedded browser remained at 640px.

**Time:** about 10 minutes.



### CC-04: "The buttons don't work on my tablet"

**Reproduced:** At tablet widths from 761px to 900px, tapping Add to Cart and the save star did not activate the controls, while they worked at laptop and phone widths.

**Cause:** A tablet-only CSS media rule created transparent `::after` overlays above the bottom of every dish card and above the favorite-star area. Although invisible, these pseudo-elements captured pointer taps before they could reach the buttons.

**Fix:** Removed the unnecessary pseudo-element overlays so taps reach the dish controls directly.

**Checked:** Confirmed the tablet-width overlay selectors are gone. In the browser, clicking a favorite star changed its pressed state and Add to Cart updated the cart count; the embedded browser stayed at 640px, so the interactive check did not run at tablet width.

**Time:** about 20 minutes.

### CC-05: "The category bar scrolls away on my phone"

**Reproduced:** On a narrow mobile viewport, the category and filter strip sat at the top of the page but then disappeared as the menu scrolled, because the sticky header and sticky filters were both anchored to the same top offset.

**Cause:** The header and filter bar both used `position: sticky` with `top: 0`, so the browser treated them as competing sticky elements and the category bar slid out from under the fixed header instead of staying under it while the menu moved.

**Fix:** Offset the menu filter block below the full sticky header height using a shared CSS variable, so the category strip stays pinned underneath the header while users scroll through the dishes on mobile.

**Checked:** The category bar now remains visible at the top of the menu on a phone-sized viewport while scrolling.

**Time:** about 20 minutes.

### CC-06: "I ordered more than they had"

**Reproduced:** I placed an order for 3 samosas when only 2 were in stock but the program proceeded and added it to cart through check out and order was placed the stock of samosa went to -1.

**Cause:** The shared cart quantity helper enforced only the maximum-per-dish limit and did not compare the requested quantity with available stock. The menu, dish detail, and cart also left their increment controls enabled at the stock limit. Stock validation checked only for zero stock and max-per-dish limits, but never compared the requested quantity against the dish's remaining stock. The reservation step also threw an exception instead of returning a normal validation error.

**Fix:** Added a stock check to the shared quantity helper and disabled increment buttons at the lower of available stock and the per-dish maximum. Decrements remain available if a saved cart already exceeds current stock. Added a check in validateLine so qty cannot exceed the current available stock, and updated reserveStock to return a validation error instead of throwing when stock is insufficient.

**Checked:** I added Hakka Noodles to its full stock of 7. The menu quantity stopped at 7, its increment button disabled, and an extra direct increment was rejected with a stock warning without changing the cart quantity.The low-stock regression check now rejects orders above the remaining stock count.

**Time:** about 45-50 minutes.


### CC-07: "Cancelling makes it worse"

**Reproduced:** After cancelling an order, the corresponding dish stock decreased again instead of returning to the previous value.

**Cause:** The cancellation logic used `stock: dish.stock - line.qty` when releasing stock, which subtracted the quantity a second time.

**Fix:** Switched the release logic to add the order quantity back to stock.

**Checked:** The stock is now increasing instead of decreasing.

**Time:** about 10 minutes.

### CC-08: "An old coupon still works"

**Reproduced:** Sent an order with `couponCode: 'FRESHERS24'` after the coupon's `expiresAt` date had passed. The API still accepted it and returned a discount instead of rejecting the coupon.

**Cause:** Coupon validation checked `usesLeft` and slot rules, but never checked whether the coupon had expired. The list endpoint filtered expired coupons, but the order calculator still accepted expired codes from the full coupon list.

**Fix:** Added an expiration check in the coupon application logic so expired coupons return an error before any discount is calculated.

**Checked:** The smoke check for expired coupons now passes and the old coupon is rejected.

**Time:** about 10 minutes.

### CC-09: "The menu shows more dishes than it should"

**Reproduced:** The menu endpoint was returning the entire dish list instead of a page-sized slice. On the menu page, the app dumped a much longer list immediately instead of showing only a few dishes and loading more as the user scrolled or moved to the next page.

**Cause:** The pagination helper in the server was building its response with `items: list` instead of the actual sliced page, so every request used the full list regardless of the requested limit or page.

**Fix:** Changed the paginator to return the sliced `items` array for the current page and preserved the total count separately. This keeps the menu limited to the requested page size while still allowing the client to know how many dishes exist in total.

**Checked:** The smoke tests for page differences and page count now pass, and the menu no longer sends an oversized list in one response.

**Time:** about 15 minutes.


### CC-10: "Sorting by price is backwards"

**Reproduced:** The menu endpoint returned the full list on every page and the price sort was backwards, which made `page=1` and `page=2` look identical and put the most expensive dishes first when low-to-high was requested.

**Cause:** The pagination helper returned `items: list` instead of the sliced page, and the price sort comparator had the price directions flipped.

**Fix:** Changed the paginator to return the actual page slice, and corrected the `price-asc` / `price-desc` comparators to match the requested ordering.

**Checked:** The smoke checks for page differences and price sorting both pass.

**Time:** about 20 minutes.


### CC-11 — "Orders created for me do not show up under My Orders"

**Reproduced:** Placed an order while the browser was using a saved member/device id and then opened the My Orders view. The same order was not listed even though it had been created successfully and saved with the right member id.

**Cause:** The server-side list endpoint filtered orders with `o.memberId === req.query.userId`, but the request was sending `memberId`. Because the field names did not match, the filter always returned no rows for the current member.

**Fix:** Updated the query filter to compare against `req.query.memberId`, so the API returns only the orders belonging to the current member as intended.

**Checked:** The order now includes the order in the My Orders list.

**Time:** about 15 minutes.



## CC-12 — "Orders can’t go from ready to picked"

**Reproduced:** When I placed an order it advance from placed to accepted to preparing to ready, but the API refused the final step to picked. The code for rating also failed to run because the order never reached the picked state.

**Cause:** The state machine treated ready as terminal, so canTransition rejected the valid ready to picked move.

**Fix:** Removed ready from the terminal set while keeping picked, cancelled, and refunded terminal; this matches the intended order lifecycle and the existing history test.

**Checked:** The order flow from placed to picked followed by rting work properly.

**Time:** about 35-40 minutes.


