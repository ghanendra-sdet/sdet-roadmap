<div align="center" markdown="1">

# 🖥️ Playwright UI Automation

![Status](https://img.shields.io/badge/status-active-success.svg)
![Level](https://img.shields.io/badge/level-intermediate--advanced-orange.svg)
![Duration](https://img.shields.io/badge/duration-3%20weeks-blue.svg)

**Real apps don't have one text box and one button. They have modals, dynamic tables, dropdowns pretending to be `<select>` elements, and at least one animation that hates you personally.**

</div>

---

## 📑 Table of Contents

- [Why Real Apps Are Harder Than Tutorials](#-why-real-apps-are-harder-than-tutorials)
- [Component Composition (Not Just Page Objects)](#-component-composition-not-just-page-objects)
- [Real Example: A Reusable Table Component](#-real-example-a-reusable-table-component)
- [Handling Modals & Dialogs](#-handling-modals--dialogs)
- [Search, Filters & Dynamic Content](#-search-filters--dynamic-content)
- [Data-Driven UI Tests with Faker](#-data-driven-ui-tests-with-faker)
- [Cross-Browser Testing](#-cross-browser-testing)
- [File Uploads & Downloads](#-file-uploads--downloads)
- [Visual/Snapshot Testing Basics](#-visualsnapshot-testing-basics)
- [Taming Flaky Animations](#-taming-flaky-animations)
- [Hands-On Exercise](#-hands-on-exercise)
- [Common Mistakes](#-common-mistakes)
- [Next Steps](#-next-steps)

---

## 🤔 Why Real Apps Are Harder Than Tutorials

Every Playwright tutorial teaches you to click a button and check some text. Great. Now go test a
real merchant dashboard: a table with server-side pagination, a search bar with three dropdown
filters, a "view details" action that opens a slide-out drawer, an "export" button that triggers a
file download, and a confirmation modal before anything destructive happens.

None of that is exotic — it's just what *every* real product looks like past week one. This
module is about the patterns that show up constantly in production apps but never in a 10-minute
YouTube tutorial.

---

## 🧩 Component Composition (Not Just Page Objects)

In Module 03 you learned Page Objects: one class per page. But real pages share pieces —
*every* page in an app has the same table component, the same search bar, the same modal
behavior. Copy-pasting table logic into 15 different page objects is exactly the mistake POM was
supposed to fix, just moved up one level.

The fix: **reusable components**, composed *into* page objects, the same way a page composes
HTML elements.

```typescript
export class TransactionPage extends BasePage {
  // The page doesn't reimplement table logic — it OWNS an instance of it
  readonly table: Table;
  readonly searchFilter: Search;

  constructor(page: Page) {
    super(page);
    this.table = new Table(page, '[data-testid="transaction-table"], table.transaction-list-table');
    this.searchFilter = new Search(page, '[data-testid="txn-search-bar"], .txn-filter-container');
  }

  async goto(): Promise<void> {
    await this.navigate('/collection/transactions');
  }
}
```

A test then reads naturally, composed the same way the page object is:

```typescript
test('should filter transactions by status', async ({ collectionTransactionPage }) => {
  await collectionTransactionPage.filterByStatus('SUCCESS');

  const rowCount = await collectionTransactionPage.getTransactionCount();
  for (let i = 0; i < Math.min(rowCount, 5); i++) {
    const status = await collectionTransactionPage.table.getCellValueByColumnName(i, 'Status');
    expect(status.toUpperCase()).toBe('SUCCESS');
  }
});
```

`collectionTransactionPage.table.getCellValueByColumnName(...)` — the page owns a table, the table
owns its own logic. Neither one knows about the other's internals. This is the same "single
responsibility" idea from clean code, applied to test automation.

---

## 🔍 Real Example: A Reusable Table Component

Here's the actual `Table` component from the real framework
([`playwright-starter-framework`](https://github.com/ghanendra-sdet/playwright-starter-framework))
— notice it has zero idea what data it's displaying. It doesn't know about transactions or
beneficiaries or merchants. It only knows "how to interact with an HTML table":

```typescript
export class Table extends BaseComponent {
  private readonly rows = this.container.locator('tbody tr');
  private readonly headers = this.container.locator('thead th');
  private readonly emptyMsg = this.container.locator(
    '[data-testid="table-empty"], .no-data, text=/no data|no records/i'
  );

  async getRowCount(): Promise<number> {
    if (await this.isTableEmpty()) return 0;
    return await this.rows.count();
  }

  async getCellValueByColumnName(row: number, columnName: string): Promise<string> {
    const headers = await this.getColumnHeaders();
    const colIndex = headers.findIndex(h => h.toLowerCase() === columnName.toLowerCase());
    if (colIndex === -1) {
      throw new Error(`Column "${columnName}" not found in table headers: ${headers.join(', ')}`);
    }
    return await this.getCellValue(row, colIndex);
  }

  async isTableEmpty(): Promise<boolean> {
    return await this.emptyMsg.isVisible();
  }

  async clickActionButton(rowIndex: number, actionName: string): Promise<void> {
    const row = this.rows.nth(rowIndex);
    const actionBtn = row
      .locator(`[data-testid*="${actionName.toLowerCase()}"], button:has-text("${actionName}"), [title*="${actionName}"]`)
      .first();
    await actionBtn.click();
    await this.page.waitForLoadState('networkidle');
  }
}
```

A few details worth studying closely:

- **`getCellValueByColumnName` instead of hardcoded column indices.** Tests read
  `getCellValueByColumnName(0, 'Status')` instead of the meaningless `getCellValue(0, 4)`. If a
  column gets reordered in the UI, tests written against column *names* don't break. Tests written
  against column *positions* silently start reading the wrong data — which is far worse than a
  visible failure, because you might not notice for a while.
- **`isTableEmpty()` is checked before every count-dependent operation.** A table with zero rows
  isn't a bug — it's a real, valid state (a merchant with no transactions yet). A naive
  implementation that just calls `.count()` on `tbody tr` would return `0` either way, silently
  hiding the difference between "empty state rendered correctly" and "table failed to render at
  all." Testing that distinction is exactly the kind of thing a careless test misses.
- **`clickActionButton` tries three different selector strategies at once** (`data-testid`
  contains, button text, `title` attribute) — the same resilience pattern you saw with `.or()`
  chains in Module 03, just written as a single combined CSS selector this time.

Build this exact component yourself as part of the hands-on exercise below — recreating it (not
copy-pasting it) is how the pattern actually sticks.

---

## 🪟 Handling Modals & Dialogs

Confirmation dialogs ("Are you sure you want to delete this?") are everywhere in real apps, and
they're a classic source of flaky tests if you don't wait for them properly.

```typescript
export class Popup extends BaseComponent {
  private readonly confirmBtn = this.container.locator(
    '[data-testid="popup-confirm"], button:has-text("Confirm"), button:has-text("Yes"), button:has-text("OK")'
  );

  constructor(page: any, containerSelector = '[data-testid="modal"], .modal, role=dialog') {
    super(page, containerSelector);
  }

  async waitForPopup(timeout = 5000): Promise<void> {
    await this.container.waitFor({ state: 'visible', timeout });
  }

  async confirm(): Promise<void> {
    await this.confirmBtn.click();
    await this.page.waitForLoadState('networkidle');
  }
}
```

```typescript
test('deleting a beneficiary requires confirmation', async ({ page, beneficiaryPage }) => {
  await beneficiaryPage.clickDelete(0);

  const confirmDialog = new Popup(page);
  await confirmDialog.waitForPopup();               // don't assume it's already there
  expect(await confirmDialog.getTitle()).toContain('Delete');

  await confirmDialog.confirm();
  // now assert the row is actually gone
});
```

> [!TIP]
> **The `role=dialog` fallback in the container selector isn't decoration.** Well-built modals
> expose `role="dialog"` for accessibility (screen readers need to know focus just jumped into an
> overlay). If a modal is missing that role, that's a real accessibility bug worth flagging —
> your test locator strategy just doubled as an accessibility check for free.

---

## 🔎 Search, Filters & Dynamic Content

A search component that supports both native `<select>` dropdowns *and* custom JS dropdowns
(because real design systems rarely use plain `<select>` — they build styled ones):

```typescript
async applyFilter(filterName: string, value: string): Promise<void> {
  const dropdown = this.filterDropdowns(filterName);

  if (await dropdown.evaluate(el => el.tagName === 'SELECT')) {
    await dropdown.selectOption(value);
  } else {
    // Custom div-based dropdown: click to open, then click the option
    await dropdown.click();
    const option = this.page.locator(`role=option[name="${value}"], .select-option:has-text("${value}")`);
    await option.click();
  }
  await this.page.waitForLoadState('networkidle');
}
```

The lesson: **don't assume every dropdown is a native `<select>`.** Check what you're actually
dealing with (here, via `evaluate` reading the tag name) and branch. This is a real pattern, not
theoretical — plenty of production design systems (React/Vue component libraries) render
dropdowns as styled `<div>`s with `role="listbox"`, not native form elements.

---

## 🎲 Data-Driven UI Tests with Faker

Hardcoded test data ("John Doe", "test@test.com") gets stale fast and doesn't stress unusual
input. The real framework uses [Faker.js](https://fakerjs.dev/) to generate realistic,
domain-specific data on every run:

```typescript
export class RandomData {
  static generateBeneficiaryName(): string {
    return faker.person.fullName();
  }

  static generateIFSC(): string {
    const banks = ['SBIN', 'ICIC', 'HDFC', 'BARB', 'PUNB', 'UTIB'];
    const bank = faker.helpers.arrayElement(banks);
    const code = faker.string.alphanumeric({ length: 6, casing: 'upper' });
    return `${bank}0${code}`;   // matches real Indian IFSC code format
  }

  static generateMobileNumber(): string {
    const startDigit = faker.helpers.arrayElement(['6', '7', '8', '9']); // real Indian mobile prefixes
    return `${startDigit}${faker.string.numeric(9)}`;
  }

  static generateAmount(min = 10, max = 10000): number {
    return Number(faker.finance.amount({ min, max, dec: 2 }));
  }
}
```

Notice these aren't generic "random string" generators — `generateIFSC()` produces something
*shaped like a real Indian bank IFSC code*, `generateMobileNumber()` respects the real constraint
that Indian mobile numbers start with 6, 7, 8, or 9. **Domain-aware fake data catches bugs that
generic fake data doesn't** — a beneficiary form that silently accepts an IFSC code with 3
characters instead of 11 is a real bug that random alphanumeric junk might not have triggered.

```typescript
test('should create new beneficiary', async ({ beneficiaryPage }) => {
  const benData = {
    name: RandomData.generateBeneficiaryName(),
    accountNumber: RandomData.generateAccountNumber(),
    ifsc: RandomData.generateIFSC(),
  };

  await beneficiaryPage.clickAddBeneficiary();
  await beneficiaryPage.fillBeneficiaryDetails(benData);
  await beneficiaryPage.submitBeneficiary();

  await beneficiaryPage.searchBeneficiary(benData.name);
  expect(await beneficiaryPage.isBeneficiaryPresent(benData.name)).toBe(true);
});
```

Run this test 100 times in CI and you get 100 different (but realistically-shaped) beneficiaries —
far more coverage than one hardcoded "Test Beneficiary" that's been copy-pasted into every test
run since the framework was built.

---

## 🌐 Cross-Browser Testing

This is close to free in Playwright — it's a config change, not a rewrite:

```typescript
// playwright.config.ts
projects: [
  { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
  { name: 'firefox',  use: { ...devices['Desktop Firefox'] } },
  { name: 'webkit',   use: { ...devices['Desktop Safari'] } },
],
```

```bash
npx playwright test                       # runs on every configured project
npx playwright test --project=firefox     # just one
```

The same test file runs unmodified against all three engines. When something only fails on
WebKit, that's not your test's fault — that's a real Safari-specific bug you just caught before a
Mac user did.

---

## 📁 File Uploads & Downloads

**Upload:**
```typescript
await page.getByLabel('Upload document').setInputFiles('path/to/file.pdf');
```

**Download** (real pattern from the framework — you get the download path back to assert on it):
```typescript
async exportTransactions(triggerDownloadHandler: () => Promise<string>): Promise<string> {
  return await triggerDownloadHandler();
}

// In the test:
const [download] = await Promise.all([
  page.waitForEvent('download'),
  transactionPage.exportExcelBtn.click(),
]);
expect(download.suggestedFilename()).toMatch(/transactions.*\.xlsx/);
```

`Promise.all` here isn't decoration — you have to start *listening* for the download event before
you click the button that triggers it, because the download can start firing before your next
line of code would otherwise register the listener. Get the order wrong and you'll intermittently
miss the event — a classic flaky-test cause that looks like "sometimes it just fails for no
reason."

---

## 📸 Visual/Snapshot Testing Basics

For catching unintended visual regressions (not text/logic bugs — actual "this used to look right
and now it doesn't" bugs):

```typescript
test('dashboard visual regression', async ({ page }) => {
  await page.goto('/dashboard');
  await expect(page).toHaveScreenshot('dashboard.png', {
    maxDiffPixelRatio: 0.02,   // allow ~2% diff for font rendering noise across machines
  });
});
```

First run generates the baseline image. Every run after compares against it and fails on
meaningful pixel differences. Use this sparingly — for a handful of critical, visually stable
screens (a payment confirmation page, a branded email preview) — not for every page, since visual
tests are the most maintenance-heavy kind of test you can write (any legitimate design tweak
requires regenerating the baseline).

---

## 🎬 Taming Flaky Animations

A loading spinner covering a button for 300ms causes exactly the kind of "sometimes it fails"
test that erodes trust in your whole suite. Playwright's auto-waiting handles most of this, but
for animations specifically:

```typescript
// Wait for an element to stop moving before interacting (Playwright does this automatically
// for actions, but explicit waits help for assertions on element position/state)
await page.locator('.toast-notification').waitFor({ state: 'visible' });
await expect(page.locator('.toast-notification')).toBeVisible();

// If a CSS transition is the culprit, disabling animations globally in test mode
// is a legitimate, common practice:
await page.addStyleTag({
  content: `*, *::before, *::after { animation-duration: 0s !important; transition-duration: 0s !important; }`
});
```

> [!NOTE]
> Disabling animations in tests is not "cheating." You're not testing *whether the fade-in looks
> nice* — that's a design review's job. You're testing *whether the feature works*, and a CSS
> transition timing shouldn't be able to fail your functional test.

---

## ✍️ Hands-On Exercise

Using [DemoQA](https://demoqa.com/) or [Automation Exercise](https://automationexercise.com/):

1. Build your own minimal `Table` component (model it on the real one above) with
   `getRowCount()`, `getCellValueByColumnName()`, and `isTableEmpty()`.
2. Build a minimal `Popup`/modal component with `waitForPopup()` and `confirm()`.
3. Write a test that: searches/filters a table, opens a row's detail modal, confirms an action,
   and re-checks the table reflects the change.
4. Add Faker.js (`npm install @faker-js/faker`) and rewrite any hardcoded test data to use it.
5. Run your suite against all three browser engines and note (in a comment) any browser-specific
   difference you find — even a timing difference counts.

---

## ⚠️ Common Mistakes

| Mistake | Why It Bites You | Fix |
|---------|-------------------|-----|
| Reimplementing table/modal logic in every page object | Massive duplication, one bug fixed in 15 places | Extract reusable components |
| Hardcoding table column positions | Breaks silently when columns reorder | Look up columns by header name |
| Assuming every dropdown is `<select>` | Test fails on custom/styled dropdowns | Detect element type, branch logic |
| Not waiting for a modal before interacting with it | Intermittent "element not found" flakiness | Explicit `waitFor({ state: 'visible' })` |
| Starting a download listener *after* clicking | Race condition — event fires before you're listening | `Promise.all([waitForEvent, click])` |
| Visual tests on every page | Baseline maintenance nightmare, constant false failures | Reserve for a few critical, stable screens |

---

## 🎓 Next Steps

You've now built (or at least deeply read) the individual pieces — Page Objects, components, API
clients, data generators. Next: stop thinking about individual tests and start thinking about the
**system that holds them all together** — config, fixtures, reporting, the "Hybrid" framework
pattern this entire roadmap has been building toward.

**Next Module:** → [06-framework-design](../06-framework-design/)

---

<div align="center" markdown="1">

**[⬆ Back to Top](#-playwright-ui-automation)** | **[🏠 Main README](../README.md)**

</div>
