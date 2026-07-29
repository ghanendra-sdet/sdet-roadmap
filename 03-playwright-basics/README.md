<div align="center" markdown="1">

# 🎭 Playwright Basics

![Status](https://img.shields.io/badge/status-active-success.svg)
![Level](https://img.shields.io/badge/level-intermediate-orange.svg)
![Duration](https://img.shields.io/badge/duration-2%20weeks-blue.svg)

**Your first robot hands. This is where "I test manually" turns into "I test at 3am while I'm asleep."**

</div>

---

## 📑 Table of Contents

- [Why Playwright (and not Selenium)](#-why-playwright-and-not-selenium)
- [Setup](#-setup-15-minutes-i-promise)
- [Your First Test](#-your-first-test)
- [Locators: The Actual Hard Part](#-locators-the-actual-hard-part)
- [Auto-Waiting: Why You'll Never Write `sleep()` Again](#-auto-waiting-why-youll-never-write-sleep-again)
- [Assertions](#-assertions-web-first-not-hope-and-pray)
- [Page Object Model, Introduced Properly](#-page-object-model-introduced-properly)
- [Real Example: Reading the Framework](#-real-example-reading-the-actual-paywize-framework)
- [Debugging Without Crying](#-debugging-without-crying)
- [Hands-On Exercise](#-hands-on-exercise)
- [Common Beginner Mistakes](#-common-beginner-mistakes)
- [Next Steps](#-next-steps)

---

## 🤔 Why Playwright (and not Selenium)

You've probably heard of Selenium. It's the granddad of browser automation — respected, everywhere,
and also the reason an entire generation of QA engineers has trust issues with `Thread.sleep(3000)`.

Here's the honest comparison:

| | Selenium | Playwright |
|---|---|---|
| Waits for elements automatically | ❌ You write explicit waits everywhere | ✅ Built in, every action auto-waits |
| Multiple browsers, one API | ⚠️ Sort of, with driver quirks | ✅ Chromium, Firefox, WebKit, same code |
| Network interception | ⚠️ Bolted on | ✅ Native |
| Debugging tools | You + `print()` statements + prayer | ✅ Trace Viewer, Inspector, time-travel debugging |
| Setup | Download drivers, manage `PATH`, cry | `npm init playwright@latest` |

> [!NOTE]
> This isn't "Selenium bad." Plenty of huge companies run Selenium in production and it works
> fine. But if you're starting fresh in 2026, Playwright removes an entire category of pain
> (flaky waits) that used to eat half of every QA engineer's week. That's the whole pitch.

---

## 🔧 Setup (15 Minutes, I Promise)

```bash
# Create a new project
npm init playwright@latest

# You'll get asked 3 questions:
# ✔ TypeScript or JavaScript? → TypeScript (future-you will thank present-you)
# ✔ Tests folder name? → tests
# ✔ Add a GitHub Actions workflow? → Yes (we'll use it in Module 08)
```

This scaffolds a `playwright.config.ts`, a `tests/` folder with an example test, and installs the
three browser engines. Run the example to prove it works:

```bash
npx playwright test
npx playwright show-report
```

If a browser popped up (or a report opened showing green checkmarks), you're done. That's the
entire setup. No driver downloads, no `PATH` environment variable archaeology.

---

## ✍️ Your First Test

Delete the example test and write this instead — it's the same shape you'll use for the rest of
your automation career:

```typescript
import { test, expect } from '@playwright/test';

test('homepage has the right title', async ({ page }) => {
  await page.goto('https://playwright.dev/');
  await expect(page).toHaveTitle(/Playwright/);
});
```

Break this down, because every Playwright test you'll ever write follows this exact skeleton:

1. `test('description', async ({ page }) => { ... })` — the `page` fixture is a fresh browser tab,
   handed to you automatically. You never manually create or close it.
2. `await page.goto(url)` — navigate. Playwright waits for the page to actually load before moving on.
3. `await expect(page).toHaveTitle(...)` — a **web-first assertion**. More on why that `await`
   matters a lot in a minute.

Run it:

```bash
npx playwright test --headed   # watch it happen in an actual browser window
```

---

## 🎯 Locators: The Actual Hard Part

Anyone can write `page.goto()`. The skill that separates a good Playwright test from a flaky one
is **how you find elements**. This is 80% of the actual craft.

### The locator hierarchy (best to worst)

```typescript
// 1️⃣ BEST — role-based, matches how a screen reader "sees" the page
page.getByRole('button', { name: 'Sign in' })

// 2️⃣ GREAT — for form fields, matches the visible label
page.getByLabel('Email address')

// 3️⃣ GOOD — for placeholder-only fields
page.getByPlaceholder('Enter your email')

// 4️⃣ FINE — explicit test hooks, the dev team owns these
page.locator('[data-testid="login-button"]')

// 5️⃣ LAST RESORT — CSS/XPath tied to page structure
page.locator('div.form-wrapper > button.btn-primary')
```

**Why this order?** Roles and labels test what a *user* sees and interacts with — they survive a
CSS redesign. A CSS selector like `div.form-wrapper > button.btn-primary` breaks the moment a
frontend dev adds a wrapping `<div>` for a totally unrelated reason. You didn't get a false bug —
you got a **brittle test**, which is worse, because now nobody trusts your suite.

> [!TIP]
> **Meme-worthy but true:** if your test suite breaks every time a developer renames a CSS class,
> the problem isn't the developer. It's that your locators were describing the *implementation*
> instead of the *intent*. `getByRole('button', { name: 'Submit' })` doesn't care if the button is
> a `<button>`, a styled `<div>`, or wrapped in fourteen more `<div>`s tomorrow.

### The real-world fallback pattern

Here's something you won't find in most tutorials, straight from a production fintech framework
([`playwright-starter-framework`](https://github.com/ghanendra-sdet/playwright-starter-framework))
— locators chained with `.or()` as a fallback chain, because real apps don't always have clean
`data-testid` attributes on day one:

```typescript
this.usernameInput = this.page
  .getByLabel('Email address')
  .or(this.page.getByPlaceholder(/enter email|username/i))
  .or(this.page.locator('[data-testid="email-input"], input[type="email"]'));
```

Read that as: "Try the accessible label first. If that's not there, try the placeholder text
(case-insensitive regex, because someone will type `Enter Email` and someone else will type
`enter email`). If *that's* not there either, fall back to a test ID or the raw input type."

This isn't over-engineering — it's what real apps in active development actually look like. Product
teams don't always add `data-testid` before shipping. A resilient locator strategy means your test
doesn't break the day someone tweaks a placeholder string in a copy update.

---

## ⏳ Auto-Waiting: Why You'll Never Write `sleep()` Again

Old-school (Selenium-style) automation looks like this:

```javascript
// ❌ The old way. Please don't.
driver.findElement(By.id('submit')).click();
Thread.sleep(3000); // "3 seconds should be enough" — narrator: it was not always enough
driver.findElement(By.id('success-message'));
```

`Thread.sleep(3000)` is a bet. Sometimes the page loads in 800ms and you just wasted 2.2 seconds
per test, multiplied across thousands of test runs. Sometimes the page takes 4 seconds under load
and your test fails anyway. You can't win.

Playwright's actions **auto-wait** for the element to be visible, stable (not animating), and
receiving events, before acting:

```typescript
// ✅ The Playwright way. No sleep needed.
await page.getByRole('button', { name: 'Submit' }).click();
// Playwright already waited for it to be clickable. You're done. Move on.
```

From the same real framework, here's `BasePage.click()` — notice there's no manual wait anywhere,
because Playwright is doing it for you under the hood:

```typescript
async click(locator: Locator): Promise<void> {
  await locator.click();
}
```

The one place you *do* need an explicit wait is when something is genuinely asynchronous in a way
Playwright can't infer — e.g. waiting for a URL change after a redirect:

```typescript
async waitForUrl(urlPattern: string | RegExp): Promise<void> {
  await this.page.waitForURL(urlPattern);
}
```

That's a **condition-based** wait ("wait until the URL matches"), not a **time-based guess**
("wait 3000ms and hope"). That distinction is the entire lesson of this section.

---

## ✅ Assertions: Web-First, Not "Hope and Pray"

Playwright's `expect()` is **web-first** — it doesn't check once and fail immediately. It retries
the check until it passes or times out (default 5s, configurable). This is why you `await` every
assertion:

```typescript
// This RETRIES for up to 5 seconds, checking the text as the DOM updates.
await expect(page.getByText('Payment Successful')).toBeVisible();

// This does NOT retry — it's a snapshot check for stuff Playwright already
// waited on elsewhere (e.g. after an action). Use it for values, not visibility.
expect(await page.title()).toBe('Dashboard');
```

**Common assertions you'll use constantly:**

```typescript
await expect(locator).toBeVisible();
await expect(locator).toBeHidden();
await expect(locator).toBeEnabled();
await expect(locator).toHaveText('Exact text');
await expect(locator).toContainText('partial');
await expect(locator).toHaveValue('input value');
await expect(locator).toHaveCount(5);
await expect(page).toHaveURL(/dashboard/);
await expect(page).toHaveTitle('My App');
```

> [!WARNING]
> Forgetting the `await` on an assertion is the single most common Playwright beginner bug. Your
> test will "pass" because the assertion silently never actually ran and JavaScript moved on
> without waiting for the promise. Turn on the `@typescript-eslint/no-floating-promises` ESLint
> rule the day you set up your project — it will save you hours of "why did this pass when it
> shouldn't have" debugging later. We'll wire this up properly in Module 07.

---

## 🏗️ Page Object Model, Introduced Properly

You'll hear "POM" constantly in automation job descriptions. Here's what it actually means, no
jargon: **stop writing locators directly in your test files.**

**Without POM (fine for a 5-minute demo, a nightmare at 200 tests):**

```typescript
test('login works', async ({ page }) => {
  await page.goto('/login');
  await page.getByLabel('Email address').fill('user@test.com');
  await page.getByLabel('Password').fill('pass123');
  await page.getByRole('button', { name: /login/i }).click();
  await expect(page).toHaveURL(/dashboard/);
});
```

If the login form changes tomorrow, you go find and fix this in **every test file that logs in**.
If you have 40 test files that start with a login, that's 40 edits for one CSS change.

**With POM:** one class owns the login page's locators and actions. Every test imports it.

```typescript
// pages/LoginPage.ts
export class LoginPage {
  private readonly usernameInput: Locator;
  private readonly passwordInput: Locator;
  private readonly loginButton: Locator;

  constructor(private page: Page) {
    this.usernameInput = page.getByLabel('Email address');
    this.passwordInput = page.getByLabel('Password');
    this.loginButton = page.getByRole('button', { name: /login/i });
  }

  async login(username: string, password: string): Promise<void> {
    await this.usernameInput.fill(username);
    await this.passwordInput.fill(password);
    await this.loginButton.click();
  }
}

// tests/login.spec.ts
test('login works', async ({ page }) => {
  const loginPage = new LoginPage(page);
  await page.goto('/login');
  await loginPage.login('user@test.com', 'pass123');
  await expect(page).toHaveURL(/dashboard/);
});
```

Now the login form changes once, in `LoginPage.ts`, and every test that logs in is fixed
automatically. This is the whole point of POM: **locators live in one place, tests describe
behavior, not mechanics.**

We'll go much deeper on this — base classes, component composition, fixtures — in
[Module 06: Framework Design](../06-framework-design/). For now, just internalize the shape:
**page objects hold "how," tests hold "what."**

---

## 🔍 Real Example: Reading the Actual Paywize Framework

Let's connect this to a real, working automation framework instead of a toy demo. The following is
straight from a production Playwright framework built for a fintech platform
([`playwright-starter-framework`](https://github.com/ghanendra-sdet/playwright-starter-framework))
— a `BasePage` class that every page object in the framework extends:

```typescript
export class BasePage {
  protected readonly page: Page;

  constructor(page: Page) {
    this.page = page;
  }

  async navigate(path: string): Promise<void> {
    await this.page.goto(path);
  }

  async waitForPageLoad(): Promise<void> {
    await this.page.waitForLoadState('load');
    await this.page.waitForLoadState('networkidle').catch(() => {
      // networkidle may not fire on pages with long-polling; swallow gracefully.
    });
  }

  async click(locator: Locator): Promise<void> {
    await locator.click();
  }

  async fill(locator: Locator, value: string): Promise<void> {
    await locator.clear();
    await locator.fill(value);
  }

  async takeScreenshot(name: string): Promise<void> {
    await this.page.screenshot({ path: `test-results/${name}.png`, fullPage: true });
  }
}
```

Notice what's happening: this class doesn't know anything about *login* or *transactions* or
*payouts*. It only knows generic browser actions — navigate, click, fill, screenshot. Every
domain-specific page object (`LoginPage`, `TransactionPage`, `BeneficiaryPage`, ...) **extends**
this and adds its own locators and business methods on top:

```typescript
export class LoginPage extends BasePage {
  private readonly usernameInput: Locator;
  // ...

  constructor(page: Page) {
    super(page); // gets navigate(), click(), fill(), etc. for free
    this.usernameInput = /* ... */;
  }

  async goto(): Promise<void> {
    await this.navigate('/login'); // inherited from BasePage
  }
}
```

That `extends BasePage` line is inheritance doing real work: every page object gets consistent
navigation, waiting, and screenshot behavior without re-writing it 20 times. This exact pattern —
one shared base class, many page objects extending it — is what you'll build yourself, from
scratch, in Module 06.

---

## 🐛 Debugging Without Crying

Three tools, in order of how often you'll reach for them:

**1. UI Mode — your daily driver:**
```bash
npx playwright test --ui
```
Time-travel through every step of a test, see the DOM at each point, replay actions. This alone
will save you more debugging time than every `console.log()` you've ever written combined.

**2. Debug mode — step through line by line:**
```bash
npx playwright test --debug
```
Opens the Playwright Inspector, pauses before each action, lets you step forward manually.

**3. Trace Viewer — for CI failures you can't reproduce locally:**
```bash
npx playwright show-trace trace.zip
```
This is the one that separates people who *use* Playwright from people who *understand*
Playwright. We'll cover this properly in [Module 07](../07-tools-and-environment/), because it's
the tool that turns "the CI test failed and I have no idea why" into "oh, the button was covered
by a loading spinner for 200ms, got it" — in about 30 seconds instead of an hour.

---

## ✍️ Hands-On Exercise

**Goal:** Build a small POM-based test suite against a real, stable practice site — no login
credentials, no fintech complexity, just locators and structure.

Use [Sauce Demo](https://www.saucedemo.com/) (username: `standard_user`, password: `secret_sauce`)
or [The Internet](https://the-internet.herokuapp.com/).

1. Write a `LoginPage` class with `goto()`, `login(username, password)`, and
   `getErrorMessage()` methods — model it on the real `LoginPage.ts` shown above, including a
   `.or()` fallback chain for at least one locator.
2. Write 3 tests: successful login, login with a locked-out user, login with a blank password.
3. Use only `getByRole` / `getByLabel` / `getByPlaceholder` locators — no raw CSS selectors
   allowed for this exercise. If you can't find an accessible locator, that's a real signal worth
   noting (and exactly the kind of gap you'd raise with a dev team in a real job).
4. Run it with `--ui` mode at least once and actually watch the time-travel debugger work. Don't
   skip this step — it's the "aha" moment for most people learning Playwright.

> [!TIP]
> Push this to a GitHub repo. This is your first genuinely portfolio-worthy artifact from this
> whole roadmap — a real POM-based test suite, not a copy-pasted tutorial snippet.

---

## ⚠️ Common Beginner Mistakes

| Mistake | Why It Bites You | Fix |
|---------|-------------------|-----|
| Forgetting `await` on assertions | Test "passes" without actually checking anything | Enable `no-floating-promises` ESLint rule |
| CSS selectors tied to styling classes | Breaks on every redesign, not just bugs | Prefer `getByRole`/`getByLabel` |
| One giant test doing 10 things | One failure hides 9 other results; slow to debug | One test = one behavior |
| Hardcoding waits (`page.waitForTimeout(3000)`) | Defeats the entire point of Playwright | Use condition-based waits (`waitForURL`, `toBeVisible`) |
| Sharing state between tests (test 2 depends on test 1's leftover data) | Tests fail in random order, impossible to debug | Each test sets up its own data |

---

## 🎓 Next Steps

You can now find things on a page, interact with them reliably, and assert on what happened —
without a single `Thread.sleep()`. Next: point that same skill set at APIs directly, using
Playwright's `request` fixture — no browser required.

**Next Module:** → [04-playwright-api-automation](../04-playwright-api-automation/)

---

<div align="center" markdown="1">

**[⬆ Back to Top](#-playwright-basics)** | **[🏠 Main README](../README.md)**

</div>
