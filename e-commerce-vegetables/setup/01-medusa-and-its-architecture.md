# 01. What is Medusa and How Does It Work?

Read this file first. Then read `02-medusa-end-to-end-setup.md` to build it on your own computer.

This file explains Medusa like you are talking to a school kid. You do not need to know anything about Medusa before you start. When a technical word is needed, it is explained right away. At the end there is a word list and a small quiz.

**What you will learn in this file:**

1. What an online shop is made of
2. What Medusa is and why people use it
3. What "headless" and "API" mean
4. The pieces that run on your computer
5. The building blocks inside Medusa
6. The shop words that confuse beginners
7. What happens when a customer buys something
8. What the project folders look like
9. What "inject data" means

---

## 1. The big story: our vegetable shop

Imagine you want to open a **vegetable shop on the internet**. People should be able to:

- look at tomatoes, carrots and spinach
- put some in a basket
- pay for them
- get them delivered

Now think about a real shop in the street. It has two sides:

| Real shop | Internet shop |
|---|---|
| The **shop window and shelves** that customers see | The **storefront** (the website people visit) |
| The **back room**: stock list, price list, order book, boss's office | The **backend** (the brain that stores and handles everything) |

A customer never walks into the back room. But the shop cannot work without it. Somebody must count the stock, write down the orders and know the prices.

**Medusa is the back room.** It is the brain of the shop.

---

## 2. What is Medusa?

Medusa is a free, open source tool that gives you a ready-made **brain for an online shop**.

"Open source" means anyone can read its code and use it for free.

Without Medusa, you would have to build all of this yourself:

- a place to save products
- a basket (called a **cart**)
- a way to take orders
- a way to remember customers
- a way to handle payments, shipping, discounts and taxes

That is months of work. Medusa gives you all of it on day one. After that, you change it to fit your own shop.

Medusa is written in **JavaScript/TypeScript** and runs on **Node.js**. (Node.js is a program that lets JavaScript run on your computer, not only inside a web browser.)

This guide uses **Medusa version 2**. Version 1 is old and works differently, so if you read an old blog post, check which version it talks about.

---

## 3. What does "headless" mean?

You will hear this word a lot, so let us understand it.

Think of a person. The **body** does the work (thinks, remembers, moves). The **head** is the face you look at.

- In many old shop tools, the body and head are glued together. You cannot change the face without touching the body.
- In a **headless** shop, the body (backend) and the head (website) are **separate**.

Medusa is **headless**. It is only the body. It does not force any website design on you.

Why is this good?

- Today your face can be a website.
- Tomorrow you can add a phone app. It talks to the **same body**.
- You can change how the website looks without breaking how orders work.

The body and the head talk to each other using **APIs**.

### What is an API?

An API is like a **waiter in a restaurant**.

- You (the website) tell the waiter: "I want the list of vegetables."
- The waiter walks to the kitchen (Medusa), gets the list and brings it back.

You never go into the kitchen yourself. You only talk to the waiter. In real life the "waiter" is a web address, like `http://localhost:9000/store/products`.

What is `localhost`? It means "this very computer". So `localhost:9000` means "the program on my own computer that listens at door number 9000".

---

## 4. The pieces of a Medusa shop

When you run a full Medusa shop on your computer, there are **4 pieces** (plus one optional piece):

```
   Customer's browser                    Shop owner's browser
         |                                        |
         v                                        v
  +----------------+                    +-------------------+
  |  STOREFRONT    |                    |  ADMIN DASHBOARD  |
  |  (Next.js)     |                    |  (control panel)  |
  |  port 8000     |                    |  port 9000/app    |
  +-------+--------+                    +---------+---------+
          |  asks the waiter (API)                |
          +------------------+--------------------+
                             v
                    +------------------+
                    |  MEDUSA SERVER   |   <- the brain (port 9000)
                    |  (Node.js)       |
                    +--------+---------+
                             |
                  +----------+-----------+
                  v                      v
          +--------------+      +------------------+
          | POSTGRESQL   |      | REDIS (optional  |
          | the database |      | for local use)   |
          +--------------+      +------------------+
```

Read the picture from top to bottom. The people are at the top. The data is at the bottom. The Medusa server sits in the middle and passes messages up and down.

### Piece 1: Medusa Server (the brain)

- This is the main program. It lives in a folder on your computer.
- It listens on **port 9000**.
- A **port** is like a door number on your computer. Different programs use different doors so they do not bump into each other.
- It holds all the shop rules: how a cart works, how an order is made, how prices are picked.

### Piece 2: Admin Dashboard (the control panel for the shop owner)

- A website that comes **built into** Medusa. You do not build it.
- Open it at `http://localhost:9000/app`.
- The shop owner uses it to add vegetables, set prices, see orders, manage stock and make discount codes.
- Think of it as the **boss's office computer**.

### Piece 3: Storefront (the shop window)

- This is the website customers see.
- Medusa gives you a **starter storefront** made with **Next.js**. (Next.js is a popular tool for building websites with React.)
- Open it at `http://localhost:8000`.
- It is a separate project in a separate folder. It only **asks** the Medusa server for data.
- This is the "default storefront": a ready-made shop window, so you do not have to design one from zero.

### Piece 4: PostgreSQL (the database, the big notebook)

- A **database** is a super organized notebook where the computer writes things down so it does not forget them.
- Medusa uses **PostgreSQL** (people say "Postgres"). It is free.
- Every product, price, customer and order is saved here.
- If you switch off your computer and switch it on again, the data is still there, because it is written in this notebook.
- A "local database" means **a Postgres notebook living on your own computer**. Nobody else can see it.

### Piece 5: Redis (optional helper)

- A very fast short-term memory. It is used for sending messages between parts and for caching. (Caching means keeping an answer close by, so you do not have to look it up again.)
- **For local work you do not need it.** Medusa uses a simple built-in version by default.
- On a real live website, you would usually add it. Do not worry about it now.

---

## 5. The building blocks inside Medusa

Now we open the brain and look inside. Medusa is built like **Lego**. There are small blocks, and each block does one job.

### 5.1 Modules (the Lego blocks)

A **module** is one block that does one job. Medusa comes with many ready-made ones, called **Commerce Modules**:

| Module | What it does (in vegetable shop words) |
|---|---|
| **Product** | Knows about tomatoes, carrots, their photos, descriptions and sizes |
| **Pricing** | Knows that 1 kg tomatoes costs 3 dollars |
| **Inventory** | Counts how many kg of carrots are left on the shelf |
| **Stock Location** | Knows where the stock is kept (for example "Main Warehouse") |
| **Cart** | The shopping basket a customer fills |
| **Order** | The finished purchase, saved like a receipt |
| **Customer** | Remembers who bought things |
| **Payment** | Takes the money (card, etc.) |
| **Fulfillment** | Packs and ships the order (shipping options, delivery) |
| **Tax** | Works out the tax |
| **Promotion** | Discount codes like "FRESH10" |
| **Region** | A shopping area with its own money and rules (example: "Europe uses Euro") |
| **Sales Channel** | A place where you sell (website, mobile app, market stall) |
| **Auth** | Checks who is who when logging in |
| **User** | The shop staff (admins) |
| **API Key** | Special keys that let a website talk to Medusa |
| **Store** | Settings of the shop itself (name, default currency) |

Each module is separate. This is what makes Medusa easy to extend. You can add your own block. For example, a **"Farm"** module that stores which farm grew each vegetable.

### 5.2 API Routes (the waiter's menu)

An **API route** is one thing the waiter can do. Each one has a web address.

There are two kinds:

- **Store API** (`/store/...`): for customers. Example: `GET /store/products` means "show me the products".
- **Admin API** (`/admin/...`): for shop staff. Example: `POST /admin/products` means "create a new product".

`GET` means "give me something". `POST` means "here is something new, please save it".

The admin dashboard itself uses the Admin API. The storefront uses the Store API.

### 5.3 Workflows (a recipe with steps)

A **workflow** is a **recipe**. Many things in a shop need several steps in a row. Example, "create an order":

1. check that stock is there
2. take the money
3. write the order in the notebook
4. reduce the stock number

If step 3 fails, what happens to the money from step 2? A workflow knows how to **undo** earlier steps (this is called **rollback**) so nothing is left half-done. This keeps the shop safe from mistakes.

You will use a workflow in the setup file to add vegetables.

### 5.4 Events and Subscribers (the doorbell)

When something happens, Medusa rings a bell. This is an **event**. Example: `order.placed` rings when someone buys.

A **subscriber** is a little helper who listens for a bell and then does something. Example: when `order.placed` rings, send a thank you email.

### 5.5 Scheduled Jobs (the alarm clock)

A **scheduled job** runs by itself at a set time. Example: "Every night at 2 AM, mark vegetables that expired as out of stock."

### 5.6 Links (a string between two Lego blocks)

Blocks are kept apart on purpose. But sometimes they must know about each other. A **link** ties two blocks together. Example: link a **Product** to your custom **Farm** so you can ask "which farm grew this tomato?"

### 5.7 Admin Extensions (add to the control panel)

You can add your own pages and widgets into the Admin Dashboard, for example a page that lists all farms.

---

## 6. Important shop ideas you must know

Medusa uses a few words that confuse beginners. Here they are made simple.

### Product, Variant and Option

- **Product** = the general item. Example: "Tomato".
- **Option** = a choice you can make. Example: "Weight".
- **Variant** = one exact thing you can buy. Example: "Tomato, 500 g" and "Tomato, 1 kg".

The customer buys a **variant**. Price and stock belong to the variant.

### Category and Collection

- **Category** = a shelf label. Example: "Leafy Greens", "Root Vegetables".
- **Collection** = a hand-picked group. Example: "Summer Specials".

### Region

A region says: "In this area, we use this money and these rules." Example: Region "Europe" uses Euro and ships to Germany, France and so on. A customer's cart is always in a region.

### Sales Channel

Where you sell. Example: "Website". A product must be added to a sales channel or the website will not show it. This is a very common beginner mistake.

### Publishable API Key (very important)

The storefront needs a **key** to talk to Medusa, like a visitor badge at a school gate. This key is called a **Publishable API Key**. It also links the storefront to a sales channel, so Medusa knows which products to show.

If this key is missing or wrong, the storefront shows errors or no products.

### Stock Location, Shipping Profile and Fulfillment

- **Stock location** = where the vegetables are stored.
- **Shipping profile** = a group of items that ship the same way (for example "default" or "cold items").
- **Fulfillment** = the packing and sending part. A product needs a shipping profile.

### Money amounts

In Medusa version 2, prices are written in **normal units**. So `3` means 3 dollars, not 3 cents.

---

## 7. What happens when a customer buys a tomato?

Follow the story:

1. Customer opens `localhost:8000`. The storefront asks the Medusa server: "Give me products." (Store API)
2. Medusa reads the notebook (Postgres) and sends back the list. The customer sees tomatoes.
3. Customer taps "Add to cart". The storefront tells Medusa: "Put this variant in a cart." Medusa saves the cart.
4. Customer enters an address and picks a shipping option. Medusa stores those.
5. Customer pays. Medusa asks the Payment module to take the money.
6. Medusa runs the "complete cart" **workflow**. It makes an **Order** and lowers stock.
7. The bell `order.placed` rings. A subscriber (if you made one) sends an email.
8. The shop owner opens the Admin Dashboard and sees the new order. They pack and ship it.

---

## 8. What the project folders look like

A Medusa shop has **two projects**: the brain (backend) and the shop window (storefront).

| Project | What it is | Runs at |
|---|---|---|
| **Backend** | The Medusa server and the Admin Dashboard | http://localhost:9000 (Admin at `/app`) |
| **Storefront** | The Next.js starter shop window | http://localhost:8000 |

The Medusa setup tool creates both for you. By default it puts them together in one parent folder, like this:

```
medusa-store/                <- the parent folder (you can name it anything)
  apps/
    backend/                 <- the brain
    storefront/              <- the shop window
  package.json, turbo.json   <- glue files that tie the two together
```

This kind of parent folder is called a **monorepo** (one repository that holds many projects). The glue tool is called **Turborepo**.

Some teams prefer **two separate Git repositories**, one for the backend and one for the storefront. This is allowed, because the storefront only talks to the backend through the API. File 02 shows how to split them if you want that. The folder insides below are the same in both ways.

### 8.1 Inside the backend project

```
backend/
  src/
    api/            <- your own API routes (extra waiter tasks)
    modules/        <- your own Lego blocks (like "farm")
    workflows/      <- your own recipes
    subscribers/    <- your own listeners for events
    jobs/           <- your own alarm-clock tasks
    links/          <- your own strings between modules
    admin/          <- your own extra admin pages
    scripts/
      seed.ts       <- a script that fills the shop with sample data
  medusa-config.ts  <- the main settings file (database address, modules)
  .env              <- secret settings (database password etc.). Never share it.
  package.json      <- list of tools the project uses and the commands you can run
  node_modules/     <- downloaded tools (huge folder, ignore it)
```

The three files to remember:

- `medusa-config.ts`: the main settings.
- `.env`: your private settings, such as the database address.
- `src/scripts/seed.ts`: the file that injects sample data.

### 8.2 Inside the storefront project

The Next.js starter storefront looks roughly like this:

```
storefront/
  src/
    app/            <- the pages people visit (home, store, cart, checkout, account)
    modules/        <- the building pieces of those pages (product card, cart, nav bar)
    lib/            <- helpers that call the Medusa API (the "waiter" code)
  public/           <- images and icons
  .env.local        <- private settings: backend address and publishable API key
  next.config.js    <- Next.js settings
  package.json      <- tools and commands (npm run dev)
  node_modules/     <- downloaded tools (huge folder, ignore it)
```

The two things to remember:

- `.env.local`: holds `NEXT_PUBLIC_MEDUSA_BACKEND_URL` and `NEXT_PUBLIC_MEDUSA_PUBLISHABLE_KEY`. Without these the storefront cannot find the brain.
- `src/app/`: change a page here to change how the shop looks.

### 8.3 Keep secrets out of Git

`.env` (backend) and `.env.local` (storefront) hold passwords and keys. They must **not** be pushed to GitHub. The Medusa and Next.js starters already list them in `.gitignore`, which is the file that tells Git what to skip. Only the template files (`.env.template`) go to Git.

---

## 9. What does "inject data" mean?

An empty shop has no vegetables. **Injecting data** (also called **seeding**) means filling the notebook with starting data: products, prices, categories, regions.

There are three ways, and you will meet all of them in file 02:

1. **Seed script**: a file of code that adds many items in one go. Fastest. Good for developers.
2. **Admin Dashboard**: click and type each product by hand. Good for a few items.
3. **CSV import**: upload a spreadsheet of products in the Admin Dashboard.

---

## 10. Words list (cheat sheet)

| Word | Simple meaning |
|---|---|
| Backend | The brain and notebook of the shop |
| Frontend / Storefront | The shop window people see |
| Headless | Brain and face are separate |
| API | A waiter that carries messages |
| Port | A door number on your computer |
| localhost | "This very computer" |
| Node.js | The program that runs JavaScript on your computer |
| npm | A helper that downloads tools your project needs |
| PostgreSQL | The database (the big notebook) |
| Migration | An instruction that builds the empty tables in the notebook |
| Seed | Filling the notebook with starting data |
| Module | One Lego block with one job |
| Workflow | A recipe of steps that can be undone |
| Event | A bell that rings when something happens |
| Subscriber | A listener that reacts to an event |
| Region | An area with its own money and rules |
| Sales channel | A place where you sell |
| Publishable API key | The visitor badge the storefront shows to Medusa |
| Monorepo | One Git repository that holds more than one project |
| `.env` | A file of private settings |

---

## 11. Quick check: can you answer these?

If you can answer these, you understood the architecture:

1. Which piece is the "brain" and which is the "shop window"?
2. Why is Medusa called "headless"?
3. What is the difference between a product and a variant?
4. Why does the storefront need a publishable API key?
5. What does a workflow do if step 3 fails?
6. What does "seeding" mean?

**Answers:**

1. The Medusa server is the brain. The Next.js storefront is the shop window.
2. Because the backend (body) does not come glued to a website (head). They are separate and talk through an API.
3. A product is the general item (Tomato). A variant is one exact thing you can buy (Tomato, 1 kg). Price and stock belong to the variant.
4. It is the visitor badge. It lets the storefront talk to Medusa and tells Medusa which sales channel (which products) to show.
5. It undoes the earlier steps (rollback), so nothing is left half-done.
6. Filling the empty database with starting data such as products and prices.

Next step: open `02-medusa-end-to-end-setup.md` and build it for real.
