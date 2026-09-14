# Household finances — setup

Two parts: GitHub Pages serves the page, Supabase holds the data behind a login.
Budget about 20 minutes. No credit card, no cost.

Files:

- `index.html` — the app
- `config.js` — where your Supabase details go
- `SETUP.md` — this file

---

## Before you start: what's public and what isn't

A GitHub Pages site is reachable by anyone who knows the URL, even if the
repository itself is private. So none of your financial data goes in these
files. The page is an empty shell; the numbers live in Supabase and only
load after one of you signs in.

The `anonKey` in `config.js` is designed to be published. On its own it
grants nothing — the database rules in Step 6 are what restrict access to
your two email addresses. Don't skip Step 6.

---

## Part 1 — Put the site online

**1. Create the repository**

On github.com, click **+** (top right) → **New repository**.
Name it `household` (or anything). Public is fine — see the note above.
Tick **Add a README file**. Click **Create repository**.

**2. Upload the files**

In the repository, click **Add file** → **Upload files**. Drag in
`index.html`, `config.js`, and `SETUP.md`. Click **Commit changes**.

**3. Turn on Pages**

Go to **Settings** → **Pages** (left sidebar).
Under *Source* choose **Deploy from a branch**, set the branch to **main**
and the folder to **/ (root)**. Click **Save**.

Wait a minute or two, then refresh. GitHub shows the address, which looks
like `https://yourname.github.io/household/`. Open it — you'll see the app
with a yellow "saving to this browser only" banner. That's expected until
Part 2 is done.

---

## Part 2 — Add the shared database

**4. Create a Supabase project**

Go to supabase.com, sign up, click **New project**.
Give it a name, set a database password (save it in your password manager —
you won't need it day to day), and pick the **Sydney** region for speed.
It takes a couple of minutes to spin up.

**5. Create the table**

In the left sidebar click **SQL Editor** → **New query**. Paste this in,
replacing the two email addresses with yours and your wife's, then click
**Run**:

```sql
create table household_data (
  id          text primary key,
  data        jsonb not null default '{}'::jsonb,
  updated_at  timestamptz not null default now(),
  updated_by  text
);

insert into household_data (id, data)
values ('main', '{"txns":[],"recurring":[],"balance":0}'::jsonb);

alter table household_data enable row level security;

create policy "household members only"
  on household_data for all
  to authenticated
  using      (auth.jwt() ->> 'email' in ('you@example.com', 'her@example.com'))
  with check (auth.jwt() ->> 'email' in ('you@example.com', 'her@example.com'));

alter publication supabase_realtime add table household_data;
```

That last line is what makes a change on your phone show up on her laptop
without a refresh.

**6. Check the security rules took**

Still in the SQL editor, run:

```sql
select relrowsecurity from pg_class where relname = 'household_data';
```

It must return `true`. If it returns `false`, run the
`alter table ... enable row level security;` line again. Without this,
anyone who signs up could read your data.

**7. Copy your project details into `config.js`**

In Supabase go to **Project Settings** → **API**. Copy the **Project URL**
and the **anon public** key.

Back in your GitHub repository, click `config.js`, then the pencil icon to
edit. Replace the two placeholder values, keeping the quote marks. Commit
the change. Give Pages a minute to redeploy.

**8. Create your two accounts**

Open the site. It now asks you to sign in.
Enter your email and a password, click **Create account**. Do the same on
your wife's device with her email — or create hers here and pass it on.

Supabase sends a confirmation email by default. If you'd rather skip that,
go to **Authentication** → **Providers** → **Email** and turn off *Confirm
email* before signing up.

**9. Close the door**

Once you're both in: **Authentication** → **Sign In / Providers**, and turn
off **Allow new users to sign up**. The email allowlist in Step 5 already
blocks strangers from reading anything, but this stops them creating
accounts at all.

---

## Using the dashboard together

**It updates live.** When one of you adds a spend, it appears on the other's
screen within a second or so, and the section that changed flashes briefly.
No refreshing.

**You can see when you're both on it.** A green dot next to your email shows
who else has the page open right now.

**Drag across the cashflow line** to read the projected balance on any date,
along with any bill or pay landing that day. Works with a finger on a phone.

**Tap any category** in the *This month* list to jump to just those
transactions. The chip at the top clears the filter.

**Every entry records who made it**, shown under the description and in the
CSV export.

## Day to day

Bookmark the site on both phones. On iPhone, Safari's **Share** → **Add to
Home Screen** makes it behave like an app.

Edits save automatically and appear on the other person's screen within a
second or two. If you both edit at the same moment, the last save wins —
fine for two people, worth knowing.

**Download CSV** in the footer exports everything if you ever want it in a
spreadsheet or want a backup.

---

## Things that will eventually catch you out

**Supabase pauses free projects after 7 days with no database activity.**
Your data is safe, but the site won't load until someone clicks *Restore*
in the Supabase dashboard, and the first request after that takes around
30 seconds. If you're both using it weekly this never comes up. If you go
away for a few weeks, expect to restore it when you're back.

**Free tier ceilings** are 500 MB of database and 5 GB of transfer a month.
A household ledger uses a rounding error of that — you will not get near it.

**No automatic backups on the free plan.** Hit *Download CSV* occasionally
and keep the file somewhere.

---

## If something doesn't work

**Site loads but still shows the yellow banner** — `config.js` still has the
placeholder text, or Pages hasn't redeployed yet. Check the file on GitHub
shows your real values.

**"Invalid login credentials"** — the account doesn't exist yet, or the
email isn't in the allowlist from Step 5. Check spelling in the SQL.

**Signed in but no data, or an error in the footer** — the `household_data`
table or the `insert` line didn't run. Go back to Step 5.

**Changes don't appear on the other device** — the `alter publication` line
in Step 5 was missed. Run it on its own. Switching away from the tab and
back also forces a reload.

**Everything broke after an edit** — GitHub keeps every version. Open the
file, click **History**, and revert to the last working commit.
