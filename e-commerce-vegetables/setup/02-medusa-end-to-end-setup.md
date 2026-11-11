# 02. Medusa End to End Setup (Windows, Local Computer)

Goal of this file: by the end you will have

- a **local database** (PostgreSQL) on your computer
- the **Medusa backend** running
- the **Admin Dashboard** open in your browser
- the **default storefront** running and showing **vegetables** you injected

Read `01-medusa-and-its-architecture.md` first so the words make sense (backend, storefront, API, port, seed and so on).

Time needed: about 1 to 2 hours the first time.

> Medusa updates often. If a command here behaves differently, check the official guide at https://docs.medusajs.com (search "Installation"). The ideas stay the same.

> The commands are written for **Windows PowerShell**. On Mac or Linux, most commands are the same. Only the file copy commands and the PostgreSQL install are different.

---

## The plan in 8 small steps

1. Install the tools (Node.js, Git, PostgreSQL)
2. Create the Medusa project (the tool also creates the database for you)
3. (Optional) Split the backend and storefront into two repos
4. Check the settings file `.env`
5. Start the backend and create an admin user
6. Put sample data in the shop (seed)
7. Connect the storefront with the publishable API key
8. Add our own vegetables

Step 3 is optional. If you skip it, you only have 7 steps.

Throughout this file, `medusa-store` is the name of the project folder we create. You can pick another name. Every path below starts from inside that folder, so it does not matter where on your computer you put it.

---

## Step 1: Install the tools

You need three things. Think of them as the kitchen tools before cooking.

### 1.1 Node.js (version 20 or newer)

Check if you already have it. Open **PowerShell** and type:

```powershell
node -v
```

If it prints `v20.x.x` or higher (like `v22.x.x`), good. If you get an error or an old version, download the **LTS** version from https://nodejs.org and install it with the default options. Close PowerShell and open it again, then run `node -v` again.

`npm` comes with Node.js. Check it:

```powershell
npm -v
```

### 1.2 Git

Medusa's setup tool uses Git to copy the storefront. Check:

```powershell
git --version
```

If missing, install from https://git-scm.com.

### 1.3 PostgreSQL (the database)

Check:

```powershell
psql --version
```

If you get an error, install PostgreSQL:

1. Download from https://www.postgresql.org/download/windows/ (the installer by EDB).
2. Run it. Keep the default port **5432**.
3. It asks you to set a **password for the user `postgres`**. Pick something simple for local work, like `postgres123`, and **write it down**. The Medusa setup tool will ask for it in Step 2.
4. You can skip optional extras like Stack Builder.
5. After install, it may not be in your PATH. If `psql --version` still fails, add the PostgreSQL `bin` folder to PATH. It is usually `C:\Program Files\PostgreSQL\<version>\bin`. Close and reopen PowerShell after that.

**Make sure the PostgreSQL service is running.** Press the Windows key, type `services`, open it and look for `postgresql-x64-<version>`. Its status should say **Running**.

> You do **not** need to create the database yourself. The Medusa setup tool in Step 2 creates it for you. You only need PostgreSQL installed and running, and you must know the user name (`postgres`) and its password.

> Tip: if your password has special characters like `@` or `#`, the database address can break. For local work use a simple password with only letters and numbers.

---

## Step 2: Create the Medusa project

Pick a folder where you keep your projects and open PowerShell there. Then run:

```powershell
npx create-medusa-app@latest medusa-store
```

`npx` downloads and runs a tool without installing it forever. The first time it may ask "Ok to proceed? (y)". Type `y`.

The tool now asks you questions. Answer like this:

| Question | What to answer |
|---|---|
| Would you like to install the Next.js Starter Storefront? | **Yes** (this is the default storefront) |
| Database user name | `postgres` (the user you made when installing PostgreSQL) |
| Database password | The password you wrote down in Step 1.3 |
| Database name | Any name you like, for example `medusa_vegetables` |

The tool connects to PostgreSQL with that user and password, and **creates the database with the name you typed**. That is why we do not make the database by hand.

What the tool does by itself:

1. Builds the project folder `medusa-store`
2. Puts the backend in `medusa-store/apps/backend` and the storefront in `medusa-store/apps/storefront`
3. Installs all the tools (this can take several minutes, be patient)
4. Creates the database
5. Builds the tables inside the database (these steps are called **migrations**)
6. Writes the settings file `.env` for the backend with your database address
7. Starts the server and may open your browser on a page to create your admin account

When it finishes, you have:

```
medusa-store/
  turbo.json, package.json     <- glue files that tie both apps together
  apps/
    backend/                   <- the brain (Medusa server and Admin Dashboard)
    storefront/                <- the shop window (Next.js)
```

If the tool starts the server and opens a browser, you can leave it running and jump to Step 5.2. Otherwise stop it with `Ctrl + C` in PowerShell. We will start it ourselves later.

### If `npm install` stops with `ERESOLVE`

Sometimes the install says "unable to resolve dependency tree". It means two Medusa packages disagree on a small version number. It is harmless. Go into the app that failed and run the install again with this flag:

```powershell
npm install --legacy-peer-deps
```

To make this permanent for everyone who uses the project, create a file named `.npmrc` in that app folder with this line:

```
legacy-peer-deps=true
```

A message that says "install scripts blocked" is only a warning. You can ignore it for now.

### If the tool fails halfway

Do not panic. See the **Troubleshooting** part at the end. You can also do the steps by hand with the commands from Step 4 and Step 5.

---

## Step 3 (optional): Split into two repos

Skip this step if you are happy with one project folder. Use it only if your team wants **one Git repository for the backend and another one for the storefront**.

The idea is simple:

1. Create two empty Git repositories, one for the backend and one for the storefront.
2. Copy `medusa-store/apps/backend` into the backend repo, so that files like `medusa-config.ts` and `package.json` sit at the **top level** of the repo.
3. Copy `medusa-store/apps/storefront` into the storefront repo the same way.
4. Do not copy `.git`, `node_modules`, `.medusa` or `.next`. Install the tools again in each repo.

On Windows, `robocopy` can do the copy. Run it from the folder that contains `medusa-store`. Replace `my-backend-repo` and `my-storefront-repo` with the real folder names of your two repos:

```powershell
robocopy medusa-store\apps\backend my-backend-repo /E /XD .git node_modules .medusa
robocopy medusa-store\apps\storefront my-storefront-repo /E /XD .git node_modules .next
```

`robocopy` prints a summary. Exit codes 0 to 7 mean success. That is normal for this command.

> Never run `create-medusa-app` directly inside an existing repo. It would put both the backend and the storefront into that repo, under `apps/backend` and `apps/storefront`.

After the copy, do three things:

1. Install the tools in each repo:

   ```powershell
   cd my-backend-repo
   npm install
   cd ..\my-storefront-repo
   npm install
   ```

2. In the backend repo, open `.env`. If it has a line named `MEDUSA_ADMIN_ONBOARDING_NEXTJS_DIRECTORY`, point it to the storefront repo folder, for example `..\my-storefront-repo`. This line is only used by the Admin's first-time guide, so the shop still works without it.

3. Check that `.env` (backend) and `.env.local` (storefront) are listed in each repo's `.gitignore`. Run `git status` in each repo and confirm those two files do **not** appear. Passwords must never be pushed.

When you are sure both repos are complete, you can delete the `medusa-store` folder. The database is not touched by this, because it lives inside PostgreSQL.

From here on, in this file:

- **backend folder** means `medusa-store/apps/backend` (or your backend repo)
- **storefront folder** means `medusa-store/apps/storefront` (or your storefront repo)

---

## Step 4: Check the settings file `.env`

Open the file `.env` inside the **backend folder** in your editor. The setup tool wrote it for you. It should have lines like these:

```env
MEDUSA_ADMIN_ONBOARDING_TYPE=default
STORE_CORS=http://localhost:8000,https://docs.medusajs.com
ADMIN_CORS=http://localhost:5173,http://localhost:9000,https://docs.medusajs.com
AUTH_CORS=http://localhost:5173,http://localhost:9000,http://localhost:8000,https://docs.medusajs.com
REDIS_URL=redis://localhost:6379
JWT_SECRET=supersecret
COOKIE_SECRET=supersecret
DATABASE_URL=postgres://postgres:YOUR_PASSWORD@localhost:5432/medusa_vegetables
```

What each line means:

- `DATABASE_URL`: where the notebook (Postgres) is. It is built from the user name, password and database name you gave the setup tool. The format is `postgres://USER:PASSWORD@localhost:5432/DATABASE_NAME`. **Make sure the password and database name are right.**
- `STORE_CORS`, `ADMIN_CORS`, `AUTH_CORS`: lists of websites that are allowed to talk to Medusa. CORS is a safety rule of browsers. Your storefront runs at `localhost:8000`, so that address must be in the list.
- `JWT_SECRET`, `COOKIE_SECRET`: secret words used for logins. `supersecret` is fine on your own laptop. **Never use it on a real website.**
- `REDIS_URL`: you can ignore this for local work. If Medusa complains that it cannot reach Redis, delete this line or put a `#` in front of it.

If your `.env` is missing, copy the template and then edit the `DATABASE_URL` line:

```powershell
copy .env.template .env
```

---

## Step 5: Start the backend and make an admin user

Run these commands inside the **backend folder**.

### 5.1 Build the tables (only if the setup tool did not do it)

```powershell
npx medusa db:migrate
```

A **migration** builds the empty tables in the notebook (a products table, an orders table, and so on). If it works, you see a list of migrations running with no red errors.

If this command says the database does not exist, it means the setup tool did not finish creating it. Create it by hand once, using the same name as in `DATABASE_URL`:

```powershell
psql -U postgres -c "CREATE DATABASE medusa_vegetables;"
```

It asks for your Postgres password. Then run `npx medusa db:migrate` again.

### 5.2 Make an admin user (the login for the Admin Dashboard)

There are two ways. Pick one.

**Option A: the command line (always works)**

```powershell
npx medusa user -e admin@vegetables.local -p Admin12345
```

- `-e` is the email.
- `-p` is the password.

Pick your own and remember them. You use them to log in at `http://localhost:9000/app`.

**Option B: the sign-up form in the browser (invite link)**

The Admin Dashboard has a "Welcome to Medusa, Create your account below" form. It asks for email, first name, last name and password. You do not reach it by just opening `http://localhost:9000/app`, because that page is the login page. You reach it with an **invite link** that looks like this:

```
http://localhost:9000/app/invite?token=eyJhbGciOi...(a very long token)
```

Where the link comes from:

- The Medusa setup tool (Step 2) often opens this link in your browser by itself when it finishes. If that happened, you already saw the form.
- Someone who is already logged in can make one: Admin Dashboard, then **Settings, Users, Invite user**. Medusa gives you the invite link to open.

Fill the form and click **Create account**. After that, log in with the email and password you chose.

The invite link works once and then stops working. If it does not work any more, use Option A.

### 5.3 Start the backend

```powershell
npm run dev
```

Wait until you see a line saying the server is started on port 9000. Keep this PowerShell window **open**. The server runs as long as the window is open. To stop it, press `Ctrl + C`.

### 5.4 Check it works

Open your browser:

- Health check: http://localhost:9000/health  (should show `OK`)
- Admin Dashboard: http://localhost:9000/app  (log in with the email and password from 5.2)

If you see the Admin Dashboard, **your backend works**. Congratulations.

---

## Step 6: Put sample data in the shop (seed)

Medusa includes a ready seed script: `src/scripts/seed.ts`. It creates:

- a store with currencies
- a region (Europe) with countries
- a stock location (warehouse)
- a shipping option
- a default sales channel
- a publishable API key
- some sample products (T-shirts, sweatshirts and so on)

Open a **second** PowerShell window in the backend folder (leave the first one running), or stop the server with `Ctrl + C`, then run:

```powershell
npm run seed
```

You should see log lines such as "Seeding region data...", "Seeding product data..." and "Finished seeding".

> Run the seed **only once** on a fresh database. Running it again can fail with "already exists" errors or make duplicate data. To start over, drop and recreate the database (see Troubleshooting) and run migrations again.

After seeding, go to the Admin Dashboard, then **Products**. You will see the sample products. Now the shop is not empty.

---

## Step 7: Connect the storefront

The storefront needs the **publishable API key** (the visitor badge, see file 01).

### 7.1 Find the key

1. Open http://localhost:9000/app and log in.
2. Click **Settings** (bottom left).
3. Click **Publishable API Keys**.
4. You will see one created by the seed (often named "Webshop"). Copy its **token**. It starts with `pk_`.

### 7.2 Put it in the storefront settings

Open the file `.env.local` inside the **storefront folder**. If it does not exist, copy the template:

```powershell
copy .env.template .env.local
```

Set these values:

```env
NEXT_PUBLIC_MEDUSA_BACKEND_URL=http://localhost:9000
NEXT_PUBLIC_MEDUSA_PUBLISHABLE_KEY=pk_paste_your_key_here
NEXT_PUBLIC_BASE_URL=http://localhost:8000
NEXT_PUBLIC_DEFAULT_REGION=dk
```

What they mean:

- `NEXT_PUBLIC_MEDUSA_BACKEND_URL`: where the brain lives.
- `NEXT_PUBLIC_MEDUSA_PUBLISHABLE_KEY`: the visitor badge from 7.1.
- `NEXT_PUBLIC_BASE_URL`: where the storefront itself lives.
- `NEXT_PUBLIC_DEFAULT_REGION`: a two letter country code that **must exist in a region** in your shop. The seed makes a Europe region that includes `de` (Germany), `gb`, `fr`, `es`, `it`, `dk`, `se`. If you pick a country that no region has, the storefront crashes.

### 7.3 Start the storefront

Inside the storefront folder, install the tools if they are not installed yet:

```powershell
npm install
```

Then:

```powershell
npm run dev
```

Open http://localhost:8000. You should see the shop with products. Try adding something to the cart and open the cart. If that works, your backend and storefront are talking to each other.

You now have **3 things running**:

| What | Address | Window |
|---|---|---|
| Backend | http://localhost:9000 | PowerShell window 1 |
| Admin Dashboard | http://localhost:9000/app | (part of the backend) |
| Storefront | http://localhost:8000 | PowerShell window 2 |

---

## Step 8: Add our own vegetables

The seed gave us T-shirts. We sell vegetables, so we add our own. There are three ways. Pick one.

### Way A: By hand in the Admin Dashboard (best for learning)

1. Admin Dashboard, then **Products**, then **Create**.
2. Fill in title (example: `Tomato`), description and handle (`tomato`).
3. Add an **option**: name `Weight`, values `500g` and `1kg`.
4. Under variants, set a **price** for each variant (example: 2 and 3.5 in EUR).
5. In **Sales channels**, make sure `Default Sales Channel` is ticked. If not, the storefront will not show it.
6. Set status to **Published** and save.
7. Refresh http://localhost:8000/store. Your tomato should be there.

### Way B: Seed script for many vegetables (best for developers)

Make a new file `src/scripts/seed-vegetables.ts` inside the **backend folder** and paste:

```ts
import { ExecArgs } from "@medusajs/framework/types"
import { Modules, ProductStatus } from "@medusajs/framework/utils"
import {
  createProductCategoriesWorkflow,
  createProductsWorkflow,
} from "@medusajs/medusa/core-flows"

export default async function seedVegetables({ container }: ExecArgs) {
  const logger = container.resolve("logger")
  const fulfillmentService = container.resolve(Modules.FULFILLMENT)
  const salesChannelService = container.resolve(Modules.SALES_CHANNEL)

  // 1. find the shipping profile and sales channel the first seed created
  const [shippingProfile] = await fulfillmentService.listShippingProfiles({
    type: "default",
  })
  const [salesChannel] = await salesChannelService.listSalesChannels({
    name: "Default Sales Channel",
  })

  // 2. make the shelf labels (categories)
  logger.info("Creating vegetable categories...")
  const { result: categories } = await createProductCategoriesWorkflow(
    container
  ).run({
    input: {
      product_categories: [
        { name: "Leafy Greens", is_active: true },
        { name: "Root Vegetables", is_active: true },
        { name: "Fruit Vegetables", is_active: true },
      ],
    },
  })
  const idOf = (name: string) => categories.find((c) => c.name === name)!.id

  // 3. a small helper to make a vegetable with two sizes
  const veg = (
    title: string,
    handle: string,
    description: string,
    categoryName: string,
    price500g: number,
    price1kg: number
  ) => ({
    title,
    handle,
    description,
    status: ProductStatus.PUBLISHED,
    category_ids: [idOf(categoryName)],
    shipping_profile_id: shippingProfile.id,
    sales_channels: [{ id: salesChannel.id }],
    options: [{ title: "Weight", values: ["500g", "1kg"] }],
    variants: [
      {
        title: "500g",
        sku: `${handle}-500g`,
        manage_inventory: false,
        options: { Weight: "500g" },
        prices: [
          { amount: price500g, currency_code: "eur" },
          { amount: price500g, currency_code: "usd" },
        ],
      },
      {
        title: "1kg",
        sku: `${handle}-1kg`,
        manage_inventory: false,
        options: { Weight: "1kg" },
        prices: [
          { amount: price1kg, currency_code: "eur" },
          { amount: price1kg, currency_code: "usd" },
        ],
      },
    ],
  })

  // 4. the vegetables
  logger.info("Creating vegetables...")
  await createProductsWorkflow(container).run({
    input: {
      products: [
        veg("Spinach", "spinach", "Fresh green spinach leaves.", "Leafy Greens", 2, 3.5),
        veg("Lettuce", "lettuce", "Crunchy lettuce for salads.", "Leafy Greens", 1.5, 2.5),
        veg("Carrot", "carrot", "Sweet orange carrots.", "Root Vegetables", 1, 1.8),
        veg("Potato", "potato", "Good for boiling and frying.", "Root Vegetables", 1, 1.6),
        veg("Tomato", "tomato", "Juicy red tomatoes.", "Fruit Vegetables", 2, 3.5),
        veg("Cucumber", "cucumber", "Cool and crunchy cucumbers.", "Fruit Vegetables", 1.2, 2),
      ],
    },
  })

  logger.info("Finished seeding vegetables.")
}
```

Run it inside the backend folder:

```powershell
npx medusa exec ./src/scripts/seed-vegetables.ts
```

What the script does, in kid words:

1. Finds the shipping profile and the sales channel that already exist.
2. Makes three shelf labels (categories).
3. Makes six vegetables, each with two sizes and a price in Euro and Dollar.
4. Puts each vegetable in the sales channel so the storefront can see it.

`manage_inventory: false` means "do not count stock for this item". That keeps things simple while you learn. Later you can turn on stock counting and set quantities in the Admin Dashboard under **Inventory**.

Refresh http://localhost:8000/store to see your vegetables. If you do not see them, check the **Troubleshooting** table below.

> Seed scripts use Medusa's internal names. If a name changed in a newer Medusa version, the error message will tell you which one. The official seed file `src/scripts/seed.ts` in your project is the best example to copy from.

### Way C: CSV import

1. Admin Dashboard, then **Products**, then the **Import** button.
2. Download the template CSV it offers, fill in your vegetables and upload it.

Good for big lists from a spreadsheet.

---

## Everyday commands (cheat sheet)

Run these inside the **backend folder**:

| What you want | Command |
|---|---|
| Start the backend | `npm run dev` |
| Build the tables | `npx medusa db:migrate` |
| Make an admin user | `npx medusa user -e EMAIL -p PASSWORD` |
| Fill with sample data | `npm run seed` |
| Run your own script | `npx medusa exec ./src/scripts/FILE.ts` |

Run this inside the **storefront folder**:

| What you want | Command |
|---|---|
| Start the storefront | `npm run dev` |

Start order every day: **1)** make sure the PostgreSQL service is running, **2)** start the backend, **3)** start the storefront.

---

## Troubleshooting

| Problem | What it means and what to do |
|---|---|
| `node` is not recognized | Node.js is not installed or PowerShell was not reopened after install. Install it, then open a new PowerShell. |
| `psql` is not recognized | The PostgreSQL `bin` folder is not in PATH. Add `C:\Program Files\PostgreSQL\<version>\bin` to PATH. |
| Setup tool says `password authentication failed for user "postgres"` | The Postgres password you typed is wrong. Run the tool again with the right password. |
| `ECONNREFUSED 127.0.0.1:5432` | PostgreSQL is not running. Open Windows Services and start `postgresql-x64-xx`. |
| `database "..." does not exist` | The database name in `DATABASE_URL` does not match a real database. Fix the name in `.env`, or create it (see 5.1). |
| Error about Redis / `ECONNREFUSED 6379` | Remove or comment out the `REDIS_URL` line in `.env`. Redis is not needed locally. |
| Port 9000 or 8000 already in use | Another program uses it. Close it, or stop old `npm run dev` windows. |
| Cannot log in to Admin | Make a new user with `npx medusa user -e ... -p ...`. |
| Storefront shows error about publishable key | `NEXT_PUBLIC_MEDUSA_PUBLISHABLE_KEY` is missing or wrong. Redo Step 7.1 and 7.2, then restart the storefront. |
| Storefront crashes: "no region found for country" | `NEXT_PUBLIC_DEFAULT_REGION` is a country that is not in any region. Use `dk` or `de` after the default seed, or add that country to a region in Admin, **Settings, Regions**. |
| Storefront shows no products | The product is not **Published**, not in the **Default Sales Channel**, or has no price in the region's currency. Check these three in the Admin Dashboard. |
| CORS error in the browser console | Add `http://localhost:8000` to `STORE_CORS` and `AUTH_CORS` in the backend `.env`, then restart the backend. |
| Changed `.env` but nothing changed | Restart the backend or storefront. They read `.env` only when they start. |
| Seed says "already exists" | You seeded twice. Start fresh, see below. |

### Start from a fresh empty database

This **deletes all data** in that database. It is safe on your own laptop, but never do it on a shared or live database. Replace `medusa_vegetables` with your database name:

```powershell
psql -U postgres -c "DROP DATABASE medusa_vegetables;"
psql -U postgres -c "CREATE DATABASE medusa_vegetables;"
npx medusa db:migrate
npx medusa user -e admin@vegetables.local -p Admin12345
npm run seed
```

Run the last three commands inside the backend folder. Then redo the key from Step 7 (a new database means a new publishable key).

---

## Checklist: am I done?

- [ ] `node -v` shows 20 or higher
- [ ] PostgreSQL service is running and the setup tool created my database
- [ ] The backend `.env` has the correct `DATABASE_URL`
- [ ] http://localhost:9000/health shows OK
- [ ] I can log in at http://localhost:9000/app
- [ ] Seed ran and I can see products in Admin
- [ ] Storefront `.env.local` has the publishable key
- [ ] http://localhost:8000 opens and shows products
- [ ] My vegetables appear in the storefront
- [ ] I can add a vegetable to the cart

If every box is ticked, you are done: **Medusa runs as the backend, with the default storefront on top, and data injected into a local database.**

## What to learn next

- Read the Medusa docs section "Learn" at https://docs.medusajs.com
- Make a custom module (for example a `farm` module) and link it to products
- Add a payment provider (Stripe has a test mode) and a real shipping setup
- Turn on inventory counting and set stock levels
- Change the storefront look (colors, logo, text)
