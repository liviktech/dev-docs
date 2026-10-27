# Plan: Delivery date and time slot at checkout

Roadmap: Phase 2, item 5. Priority P1.

## What is wrong today

- The delivery step (`src/modules/checkout/components/shipping/index.tsx`) only lets the customer choose a shipping option (Express or Standard). It has no date and no time.
- Fresh vegetables are time sensitive. The shop cannot tell if the customer wants morning or evening delivery, and the customer cannot say "tomorrow morning".
- Medusa has no built in delivery slot. The cart and the order do have a `metadata` field that we can use.
- Nothing limits how many orders go into one slot, and nothing stops the customer from choosing a time that has passed.

## What we are changing

1. Decide the slot rules (owner to confirm, these are the defaults we build with):
   - Slots: `Morning 7 to 10 AM` and `Evening 5 to 8 PM`.
   - The customer can choose today (if the cut off time has not passed) up to 3 days ahead.
   - Cut off: morning slot closes at 9 PM the night before, evening slot closes at 1 PM the same day.
2. Backend: put the slot settings in one config file in `src/lib/` (slots, cut off times, days ahead, max orders per slot) so they are easy to change.
3. Backend: add a small store API route, `GET /store/delivery-slots`, that returns the slots which are still open, with date and label.
4. Backend: add a store route, `POST /store/carts/:id/delivery-slot`, that checks the slot is still open and not full, then saves `delivery_date` and `delivery_slot` in the cart metadata. The metadata moves to the order automatically when the cart completes.
5. Backend: add a check in the order placing flow (a workflow hook on cart completion) that blocks the order when no slot was chosen or the slot is no longer open.
6. Backend: show the chosen slot in the admin order page. A simple admin widget that reads the order metadata is enough.
7. Frontend: in the delivery step add a "Delivery time" section under the shipping options. It shows date chips and slot buttons from the new API. The Continue button stays disabled until a slot is picked.
8. Frontend: show the chosen date and slot in the checkout review step, the order confirmation page and the order page in the account.

## Why we are changing it

Vegetables are bought for a meal today or tomorrow. Time slots tell the packing team when to pick and pack, cut down missed deliveries, and let the shop plan van trips.

## Files we expect to touch

Backend:
- New `src/lib/delivery-slots.ts`
- New `src/api/store/delivery-slots/route.ts`
- New `src/api/store/carts/[id]/delivery-slot/route.ts`
- `src/api/middlewares.ts` (validation)
- New workflow hook file in `src/workflows/hooks/`
- New admin widget in `src/admin/widgets/`

Frontend:
- New `src/modules/checkout/components/delivery-slot/index.tsx`
- `src/modules/checkout/components/shipping/index.tsx`
- `src/modules/checkout/components/review/index.tsx`
- `src/lib/data/cart.ts` (new server action)
- Order confirmed and order detail templates in `src/modules/order/`

## Risks and things to watch

- Time zone: always work in India time (`Asia/Kolkata`) on the server. Never trust the browser clock.
- Two people can take the last place in a slot at the same moment. Check the count again inside the complete cart step.
- Counting orders per slot means reading order metadata. That is fine at this size, but if it gets slow we can move slots into their own table later.
- **We are not building slot capacity screens or holiday closing days in this plan. They are listed for later.**
- This plan works best after `india-region-setup` (so the right shipping options and country exist).

## Out of scope

- Live order tracking and WhatsApp messages (Phase 3).
- Minimum order value and free delivery rules (Phase 3).
- Route planning for drivers.

## How to test

- Open checkout at 3 different times of day. Closed slots are hidden or disabled.
- Pick a slot, place the order, the admin order page shows date and slot.
- Try to place the order without a slot: it is refused with a clear message.
- Fill a slot to its max, the next customer sees it as full.
- Review and confirmation pages show the same slot.
