# AI Photo Reading: Complete Process & Field Guide

> **Target Audience:** Business owners, the sales and order booking team, factory office staff, and anyone who wants to understand how the app reads a handwritten order photo, without technical words.

---

## 1. What is AI Photo Reading?

Think of it as a fast junior clerk who reads the customer's handwritten sheet and types up a draft for you, but is never allowed to submit anything until a senior person has checked every line.

**AI Photo Reading** lets staff upload a photo of a handwritten glass order. The app sends the photo to an AI service that can understand pictures, gets back a list of glass pieces (width, height, quantity, holes, cutouts, glass type, edge work), and shows that list next to the photo for staff to check and fix. Only after a person reviews and saves each product does anything reach the order booking.

### Key Rules:
* 📷 **Two screens only:** Photos are attached on the **Enquiry** screen and read on the **Order Booking** screen. Work Order and Proforma Invoice have no photo reading.
* 🧑‍💼 **A person always decides:** Nothing read from a photo is saved by itself. Every product read from a photo must be opened, checked and saved by staff, and the booking cannot be saved while any of them is still waiting.
* 💰 **The same photo is never paid for twice:** The app recognises a photo it has already read (even if it was uploaded again by someone else, or on another screen) and reuses the earlier result for free.
* 🗂️ **Nothing is thrown away:** Photos and their readings are kept for good today. A cleanup job will come later, once the client agrees how long order sheets must be kept.
* 🔌 **Works without AI too:** If the AI service is not set up yet, staff can still attach and view photos. The screen simply says reading is not set up.

---

## 2. Key Components

```text
┌───────────────────────────────────────────────────────────────┐
│                  READ AN ORDER FROM A PHOTO                   │
├───────────────────────────┬───────────────────────────────────┤
│  THE PHOTO                │  WHAT WAS READ (grouped by glass) │
│  (zoom in / fit)          │                                   │
│                           │  ▸ 12mm clear toughened           │
│                           │    3 rows · 7 pieces · 4 holes    │
│                           │    W1  1200 x 900  Qty 2  ...     │
│                           │    W2   685 x 760  Qty 1  ...     │
│                           │    Remarks: Ø22mm, 150 from side  │
│                           │  ▸ 5mm frosted                    │
│                           │    ...                            │
│                           │  Yellow box = please check        │
├───────────────────────────┴───────────────────────────────────┤
│  [Back to photos] [Read again]        [Add N row(s) to products]│
└───────────────────────────────────────────────────────────────┘
```

1. **The photo:** The original sheet, shown beside what was read so staff can compare line by line.
2. **Groups by glass:** One sheet often covers more than one glass (for example "5mm Clear Glass" and "5mm Frosted Glass"). Each glass gets its own group, and each group becomes its own product on the booking.
3. **Rows:** One row per kind of piece: code (W1, W16...), the four sides (top width, bottom width, left length, right length), quantity, thickness, edge work, holes, centre holes, cutouts. Long notes (like hole sizes and positions) show on a "Remarks" line under the row.
4. **Yellow boxes:** Values the AI was not sure about. Staff must look at these. Once staff edit a value, it is no longer marked unsure.
5. **Total check:** If the sheet has a total written on it and the rows do not add up to it, a warning is shown.
6. **Read again:** Sends the photo to the AI one more time (this costs money) instead of reusing the earlier result. Use it when a reading looks wrong.
7. **Turn the photo first:** After choosing or taking a photo, staff see a preview and can turn it left or right until the writing reads the right way up. Only then is it saved and sent to the AI. The AI reads the photo exactly as shown, so the way you leave it is the right way up for good. A photo saved this way keeps that direction forever.
8. **Recents:** On an order booking, the window shows the 3 newest photos in one row under the upload boxes. **See more** opens a list of every photo for the order.

---

## 3. Step-by-Step Process

```text
  [1. Attach photo on Enquiry]  ──►  [2. Make Order Booking]  ──►  [3. Read from photo]
                                                                          │
                                                                          ▼
  [6. Save the booking]  ◄──  [5. Review each draft product]  ◄──  [4. Check the rows]
```

### Step 1: Attach the photo on the Enquiry
Each enquiry row has a **Photos** button. Staff upload a photo or take one with the phone camera. The photo simply stays with the enquiry. Nothing is read here and nothing is sent to the AI. Uploading is only allowed while the enquiry is **OPEN**; after that the photos can still be viewed but not added.

### Step 2: Make the Order Booking
When an order booking is made **From Enquiry**, it can see every photo attached to that enquiry. A **Direct** booking has no enquiry, so it only sees its own photos.

### Step 3: Read from photo
On the booking's **Products** table there is a **Read from photo** button. It lists every photo this order can use, each with a small label:
* **From enquiry:** carried over from the enquiry, not uploaded here.
* **Already read · N row(s):** a reading exists and will be reused for free.
* **Rows used:** its rows have already been added to the products table once.
* **Reading failed:** the last try did not work.

Staff pick a photo or upload a new one. The app shows "Reading photo...". A photo read before comes back instantly and free. A new photo usually takes 10 to 20 seconds, and up to about a minute.

### Step 4: Check the rows
Staff compare the rows with the photo, fix anything wrong, untick or delete rows they do not want, add missing rows to a group, and rename a group's glass if needed. Then they click **Add N row(s) to products**.

### Step 5: Review each draft product
The Products table now has one **draft** row per glass. A draft lives only on this page; nothing is in the database yet. Each draft shows how many size rows and pieces were found, how many holes, and whether the glass matched our product master. Staff click the edit button on a draft. The normal **Add Product** form opens already filled in: the product (when one clearly fits), the thickness, every size row with its hole counts, and the rate from the product master. Staff check it and save. The draft then becomes a real product on the booking.

### Step 6: Save the booking
**Save**, **Order Preview** and **Submit** stay switched off while any draft is still waiting, and the reasons are listed in red under the table. Once every draft has been reviewed and saved, they switch back on.

---

## 4. Field-by-Field Explanation

---

### 4.1 What is read for each row
* **Code:** The piece code written on the sheet, like W1 or W16. Goes into **Glass Size Code**.
* **The four sides:** Top width, bottom width, left length and right length, as the photo is held. A plain rectangle has the same figure on both sides of each pair. When a sheet gives different figures for opposite sides, each side keeps its own. Always shown in millimetres. If the sheet was in inches, the app converts it (53½ inches becomes 1359 mm) and shows the original inch value under each side so staff can check.
* **Qty:** How many pieces of this exact size and finish. When the sheet does not write a count, the AI counts the drawn pieces.
* **Thick (mm):** The glass thickness, like 5, 8 or 12. Anything above 30 mm is thrown out and flagged, because it is really a size that landed in the wrong box.
* **Edge work:** Polish, bevel, rounded corners and similar, for example "2 inch radius on 2 corners (top left, top right)".
* **Holes:** Holes drilled in one piece (not multiplied by quantity).
* **Centre:** How many of those holes are in the middle of the piece, only when the sheet says so ("centre hole", "CH").
* **Cutouts:** Notches or cutouts in one piece.
* **Remarks:** Everything else the sheet says about the piece: hole sizes and positions, "fixed" or "door", how a size was worked out.

**Blank is not the same as 0.** A blank hole count means the sheet said nothing. A 0 means the sheet said there are none.

---

### 4.2 What lands on the saved product
* **Top Width, Bottom Width, Left Length, Right Length:** from the four sides (in mm). **Actual Width** is the larger of top and bottom. **Actual Length** is the larger of left and right.
* **Qty:** from quantity.
* **Glass Size Code:** from the code.
* **Hole Qty, Centre Hole Qty, Cutout Qty:** from the counts, per single piece. Adding holes also switches the **Hole** work centre on by itself (normal app behaviour).
* **Remark (on each size row):** glass type and thickness, edge work, notes, and the original inch size, joined into one line, so nothing the customer wrote is lost.
* **Product:** pre-selected only when one product clearly fits the glass named on the sheet.
* **Rate:** from the product master, as for any product typed by hand.
* **Big Hole, Big Centre Hole, Big Cutout:** left empty on purpose. Whether a hole is "big" is a factory rule about diameter, not something a customer writes.
* **Glass type and thickness of the product:** not overwritten from the sheet. They are set in Product Details by staff.

---

### 4.3 What the product match notes mean
* **Green note:** "Product auto-filled as TUFF GLASS from "12MM Clear Tufft" on the photo. Change it if that is not right." One product clearly fit and was selected for you.
* **Amber note:** "The product name "R/G Frosted" was found on the photo but is not recorded in our master." Nothing on file looks like it. Choose the product yourself.
* **No note:** The sheet did not name the glass.
* If two products fit equally well, the app does not guess; it shows the amber note.
* A product staff already picked is never overwritten.

---

### 4.4 What the AI is told to watch for
* **Hole or cutout:** A mark that touches the edge of the glass (half circle, U, notch, box on the border) is a **cutout**. A complete closed mark fully inside the glass is a **hole**, even near an edge. An X inside the glass counts as a hole but is flagged.
* **Crossed out writing** is treated as deleted and never copied.
* **The photo is already the right way up.** Staff turned it before saving, so the AI does not turn it again.
* **A heading** like "8mm Plain Tuff + Polish" applies to every line under it.
* **A note written once** ("2 inch Radius", "all cut out") applies to every mark of that kind.
* **Pieces that differ in anything** (size, holes, cutouts, edge work) become separate rows. Identical pieces share one row with a quantity.
* **Sizes the AI had to work out** (for example a door pane from the overall height minus the fixed pane) are flagged and explained in the remarks.

---

## 5. Frequently Asked Questions (FAQ)

### Q1: Can the app save an order straight from the photo without anyone checking it?
**No.** That is on purpose. Handwriting is often unclear (700 can look like 730). Every product read from a photo has to be opened in the Add Product form, checked and saved by a person, and the booking cannot be saved until that is done for all of them.

### Q2: We read the same photo on the enquiry, the booking and again later. Do we pay each time?
**No.** The app recognises the exact same picture and reuses the earlier result at no cost. You only pay again if someone clicks **Read again** on purpose.

### Q3: I deleted a row by mistake. How do I get it back?
Open **Read from photo** again and add the same photo's rows again. It is safe: rows still on the table are left as they are, a glass already saved on the booking is skipped, and only the missing rows come back.

### Q4: The AI read something wrong. What should I do?
Fix the value in the review panel or in the Add Product form before saving. If the whole reading looks wrong, click **Read again** for a fresh read (this costs a little money), or try a clearer photo.

### Q5: Why can't I upload photos to some enquiries?
Uploading is only allowed while the enquiry is **OPEN**. You can still view the photos already attached.

### Q6: Can I read a photo on the Enquiry screen?
**No.** The Enquiry screen only keeps the photos. Reading happens on the Order Booking made from that enquiry, where the rows have a product to go into.
