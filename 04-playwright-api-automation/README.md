<div align="center" markdown="1">

# ⚡ Playwright API Automation

![Status](https://img.shields.io/badge/status-active-success.svg)
![Level](https://img.shields.io/badge/level-intermediate-orange.svg)
![Duration](https://img.shields.io/badge/duration-2%20weeks-blue.svg)

**Same tool, no browser. This is where your tests get 10x faster and you stop clicking through 6 screens just to create test data.**

</div>

---

## 📑 Table of Contents

- [Why This Module Is Its Own Thing](#-why-this-module-is-its-own-thing)
- [The `request` Fixture](#-the-request-fixture)
- [Building an API Client Layer](#-building-an-api-client-layer-not-raw-requests-everywhere)
- [Real Example: BaseAPI + CollectionAPI](#-real-example-baseapi--collectionapi)
- [The Real Power Move: API for Setup, UI for the Actual Test](#-the-real-power-move-api-for-setup-ui-for-the-actual-test)
- [Authentication in API Tests](#-authentication-in-api-tests)
- [Data-Driven API Tests](#-data-driven-api-tests)
- [Hands-On Exercise](#-hands-on-exercise)
- [Common Mistakes](#-common-mistakes)
- [Next Steps](#-next-steps)

---

## 🤔 Why This Module Is Its Own Thing

You already learned API testing concepts in [Module 02](../02-api-testing/) with Postman and
REST-assured, and you already met Playwright's `request` fixture briefly in that module's
examples. So why a whole separate module?

Because in Module 02, you were testing an API **as the entire point of the test**. In this
module, the mindset flips: **the API becomes a tool you use inside a Playwright automation
framework** — mostly to set up and tear down data fast, so your UI tests don't waste 15 seconds
clicking through a signup form just to get a logged-in user to test something completely
unrelated to signup.

> [!IMPORTANT]
> This is the single biggest speed and reliability upgrade most junior automation engineers never
> learn: **use the API to get to the state you want to test, use the UI to test the thing you
> actually care about.** A checkout test doesn't need to test signup, login, and adding items to
> cart via clicks every single run — it needs a logged-in user with items already in the cart, and
> the API can build that in 200ms.

---

## 🔧 The `request` Fixture

Playwright gives every test a `request` fixture — an `APIRequestContext` — independent of
whether the test also uses a `page`. You can write pure API tests with zero browser overhead:

```typescript
import { test, expect } from '@playwright/test';

test('GET returns a valid post', async ({ request }) => {
  const response = await request.get('https://jsonplaceholder.typicode.com/posts/1');

  expect(response.ok()).toBeTruthy();
  expect(response.status()).toBe(200);

  const post = await response.json();
  expect(post).toHaveProperty('id', 1);
});
```

No `page.goto()`, no browser launch. This test runs in milliseconds instead of seconds — multiply
that across a suite of a few hundred tests and the difference between "API-first setup" and
"click through everything" is the difference between a 3-minute CI run and a 25-minute one.

---

## 🏗️ Building an API Client Layer (Not Raw Requests Everywhere)

Just like Module 03 taught you not to scatter locators across test files (that's what Page
Objects solve), you shouldn't scatter raw `request.get('/api/...')` calls across test files
either. The same "one place owns the how" principle applies to APIs.

**Without a client layer (works, but repeats itself everywhere):**

```typescript
test('fetch transactions', async ({ request }) => {
  const res = await request.get('/api/collection/transactions', {
    headers: { 'Authorization': `Bearer ${token}`, 'Content-Type': 'application/json' },
  });
  const data = await res.json();
  expect(data.length).toBeGreaterThan(0);
});
```

Every single test that hits an API now needs to remember the headers, the auth token logic, the
base path. Change how auth works once, and you're editing dozens of files.

**With a client layer:** one `BaseAPI` class owns HTTP mechanics, and one class per domain
(`CollectionAPI`, `PayoutAPI`, ...) owns the actual endpoints. This is the exact same idea as
`BasePage` from Module 03, applied to APIs instead of pages.

---

## 🔍 Real Example: BaseAPI + CollectionAPI

Straight from the real fintech framework
([`playwright-starter-framework`](https://github.com/ghanendra-sdet/playwright-starter-framework)),
here's `BaseAPI` — the shared layer every API client extends:

```typescript
export class BaseAPI {
  protected request: APIRequestContext;
  protected authToken: string = '';

  constructor(request: APIRequestContext) {
    this.request = request;
  }

  setAuthToken(token: string): void {
    this.authToken = token;
  }

  protected buildHeaders(customHeaders?: Record<string, string>): Record<string, string> {
    const headers: Record<string, string> = {
      'Content-Type': 'application/json',
      'Accept': 'application/json',
    };
    if (this.authToken) {
      headers['Authorization'] = `Bearer ${this.authToken}`;
    }
    return { ...headers, ...customHeaders };
  }

  async get(endpoint: string, headers?: Record<string, string>): Promise<APIResponse> {
    const response = await this.request.get(endpoint, { headers: this.buildHeaders(headers) });
    return response;
  }

  async post(endpoint: string, data?: any, headers?: Record<string, string>): Promise<APIResponse> {
    const response = await this.request.post(endpoint, { data, headers: this.buildHeaders(headers) });
    return response;
  }

  // put, patch, delete follow the exact same shape

  async getResponseBody<T = any>(response: APIResponse): Promise<T> {
    return await response.json() as T;
  }
}
```

Notice: **auth headers are built in one place** (`buildHeaders`). Set the token once with
`setAuthToken()`, and every single request from every domain client automatically carries it.
Change the auth header format tomorrow (say, from `Bearer` to a custom scheme), and you change
one method, not a hundred test files.

Now, `CollectionAPI` extends this and adds only the endpoints that actually belong to the
Collection domain — no HTTP plumbing, just business meaning:

```typescript
export class CollectionAPI extends BaseAPI {
  async getTransactions(params?: Record<string, string>): Promise<any> {
    const query = params ? '?' + new URLSearchParams(params).toString() : '';
    const res = await this.get(`/api/collection/transactions${query}`);
    return await this.getResponseBody(res);
  }

  async getTransactionById(txnId: string): Promise<any> {
    const res = await this.get(`/api/collection/transactions/${txnId}`);
    return await this.getResponseBody(res);
  }

  async initiateRefund(txnId: string, amount: number): Promise<any> {
    const res = await this.post('/api/collection/refunds', { transactionId: txnId, amount });
    return await this.getResponseBody(res);
  }
}
```

A test using this reads like plain English, with zero HTTP mechanics visible:

```typescript
test('refund appears in transaction history', async ({ request }) => {
  const collectionApi = new CollectionAPI(request);
  collectionApi.setAuthToken(process.env.API_TOKEN!);

  const refund = await collectionApi.initiateRefund('TXN-88213', 500);
  expect(refund.status).toBe('INITIATED');

  const txn = await collectionApi.getTransactionById('TXN-88213');
  expect(txn.refundStatus).toBe('PENDING');
});
```

This is the pattern you'll build yourself, generalized across every module in your project, in
[Module 06: Framework Design](../06-framework-design/).

---

## 🎯 The Real Power Move: API for Setup, UI for the Actual Test

Here's the pattern that separates "someone who wrote some Playwright tests" from "someone who
designed a fast, reliable framework":

```typescript
test('refunded transaction shows correct status badge in UI', async ({ page, request }) => {
  // ── ARRANGE: use the API to get to the state you need, fast ──────────────
  const collectionApi = new CollectionAPI(request);
  collectionApi.setAuthToken(process.env.API_TOKEN!);
  const refund = await collectionApi.initiateRefund('TXN-88213', 500);

  // ── ACT: now use the UI for the ONE thing you're actually testing ────────
  const transactionPage = new TransactionPage(page);
  await transactionPage.goto();
  await transactionPage.searchTransaction('TXN-88213');

  // ── ASSERT: does the UI correctly reflect what the API just did? ─────────
  const details = await transactionPage.getTransactionDetails();
  expect(details['Status']).toBe('Refund Pending');
});
```

Think about what this test is actually verifying: **"does the UI correctly render a state that
already exists?"** — not "can a user click through 8 screens to create a refund." Those are two
different tests with two different purposes. Testing them separately, with the API doing the
heavy lifting for setup, is both faster *and* more precise about what failed when something
breaks.

> [!TIP]
> If your UI test takes 45 seconds and 40 of those seconds are just "getting to the screen you
> actually wanted to test," that's not thoroughness — that's a framework that hasn't learned this
> lesson yet. Reserve the browser for the behavior you're actually asserting on.

---

## 🔐 Authentication in API Tests

Three patterns you'll hit constantly:

```typescript
// 1. Bearer token — most common in modern APIs
const response = await request.get('/api/data', {
  headers: { 'Authorization': `Bearer ${token}` }
});

// 2. A dedicated authenticated request context (reused across many calls)
test.beforeAll(async ({ playwright }) => {
  apiContext = await playwright.request.newContext({
    baseURL: process.env.API_BASE_URL,
    extraHTTPHeaders: { 'Authorization': `Bearer ${token}` },
  });
});

// 3. Basic auth, for older/internal APIs
const context = await playwright.request.newContext({
  httpCredentials: { username: 'user', password: 'pass' },
});
```

> [!WARNING]
> Never hardcode a real token or password in a test file, even for a "harmless" staging
> environment. Use environment variables (`process.env.API_TOKEN`) loaded from a `.env` file that
> is `.gitignore`'d. We cover proper environment/secrets management in
> [Module 07](../07-tools-and-environment/) — but the habit starts now, not later.

---

## 📊 Data-Driven API Tests

Instead of writing one test per scenario, loop over a data set:

```typescript
const refundScenarios = [
  { amount: 500,   txnAmount: 1000, expectedStatus: 'INITIATED' },
  { amount: 1000,  txnAmount: 1000, expectedStatus: 'INITIATED' },  // full refund
  { amount: 1500,  txnAmount: 1000, expectedStatus: 'REJECTED' },   // over-refund
  { amount: -100,  txnAmount: 1000, expectedStatus: 'REJECTED' },   // negative amount
  { amount: 0,     txnAmount: 1000, expectedStatus: 'REJECTED' },   // zero amount
];

for (const scenario of refundScenarios) {
  test(`refund of ${scenario.amount} against a ${scenario.txnAmount} txn → ${scenario.expectedStatus}`, async ({ request }) => {
    const collectionApi = new CollectionAPI(request);
    const result = await collectionApi.initiateRefund('TXN-TEST', scenario.amount);
    expect(result.status).toBe(scenario.expectedStatus);
  });
}
```

Notice the last three rows: **over-refund, negative amount, zero amount.** A junior tester writes
the happy path (`500` against `1000`, refund works). A senior tester writes the four rows *after*
that — because that's where the money-losing bugs actually hide in a real payments system. This
is the exact same Equivalence Partitioning / Boundary Value thinking you learned as "bookish
theory" in [Module 01](../01-manual-testing/) — it wasn't bookish, it was the foundation for
everything you're doing right now.

---

## ✍️ Hands-On Exercise

Using [ReqRes](https://reqres.in/) (a free practice API with users and auth endpoints):

1. Build a small `UsersAPI` class extending a `BaseAPI` you write yourself (model it directly on
   the real `BaseAPI`/`CollectionAPI` shown above).
2. Add methods: `getUsers(page)`, `getUserById(id)`, `createUser(data)`, `deleteUser(id)`.
3. Write a data-driven test that tries creating a user with 4 different payloads: valid data,
   missing required field, empty strings, and an extremely long name (boundary testing again).
4. Write one **hybrid** test: create a user via the API, then (hypothetically — ReqRes won't
   actually reflect this in a UI) write the UI assertion as a comment describing exactly what you
   *would* assert if this were a real app with a user-list page.

---

## ⚠️ Common Mistakes

| Mistake | Why It Bites You | Fix |
|---------|-------------------|-----|
| Scattering raw `request.get()` calls in every test file | Auth/header changes require editing dozens of files | Build a `BaseAPI` + domain client layer |
| Only testing the happy path via API | Misses the exact bugs APIs are best at catching (bad input handling) | Data-drive negative and boundary cases |
| Using the UI to set up data for every test | Slow, flaky suite; failures don't tell you what actually broke | API for setup, UI for the behavior under test |
| Hardcoding tokens/URLs in test files | Secrets leak into git history; breaks across environments | Environment variables, `.env` files |
| Not cleaning up data created via API | Tests pollute shared environments, later tests fail mysteriously | Delete/reset in `afterEach`/`afterAll` |

---

## 🎓 Next Steps

You can now drive both a browser and an API with the same tool, and you know when to use which.
Next: go deep on the UI side — the patterns real apps throw at you that a toy demo never will
(dynamic tables, modals, file uploads, flaky animations).

**Next Module:** → [05-playwright-ui-automation](../05-playwright-ui-automation/)

---

<div align="center" markdown="1">

**[⬆ Back to Top](#-playwright-api-automation)** | **[🏠 Main README](../README.md)**

</div>
