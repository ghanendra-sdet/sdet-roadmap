# 🎭 Playwright API Testing Example

![Playwright](https://img.shields.io/badge/Playwright-1.40+-green.svg)
![Node.js](https://img.shields.io/badge/Node.js-18+-blue.svg)
![Last Updated](https://img.shields.io/badge/last%20updated-Feb%202026-orange.svg)

> **Learn API testing with Playwright's powerful APIRequestContext**

---

## 📑 Table of Contents

- [Overview](#-overview)
- [Setup](#-setup)
- [Basic API Testing](#-basic-api-testing)
- [GET Request](#-get-request-example)
- [POST Request](#-post-request-example)
- [PUT Request](#-put-request-example)
- [DELETE Request](#-delete-request-example)
- [Authentication](#-authentication)
- [Assertions](#-assertions)
- [Best Practices](#-best-practices)

---

## 🎯 Overview

**Playwright** provides built-in API testing with:
- ✅ APIRequestContext for HTTP requests
- ✅ Built-in assertions
- ✅ Auto-wait and retry mechanisms
- ✅ Request/Response interception
- ✅ Parallel execution
- ✅ Combined UI + API testing

---

## 🔧 Setup

### Installation

```bash
# Install Playwright
npm init playwright@latest

# Or add to existing project
npm install -D @playwright/test
```

### Configuration

**playwright.config.ts:**
```typescript
import { defineConfig } from '@playwright/test';

export default defineConfig({
  testDir: './tests',
  use: {
    baseURL: 'https://jsonplaceholder.typicode.com',
    extraHTTPHeaders: {
      'Accept': 'application/json',
      'Content-Type': 'application/json',
    },
  },
});
```

---

## 📝 Basic API Testing

**tests/api-basic.spec.ts:**
```typescript
import { test, expect } from '@playwright/test';

test.describe('JSONPlaceholder API Tests', () => {
  
  test('GET - Fetch all posts', async ({ request }) => {
    const response = await request.get('/posts');
    
    expect(response.ok()).toBeTruthy();
    expect(response.status()).toBe(200);
    
    const posts = await response.json();
    expect(posts).toHaveLength(100);
  });
  
});
```

---

## 🔵 GET Request Example

```typescript
test.describe('GET Requests', () => {
  
  test('Get all posts', async ({ request }) => {
    const response = await request.get('/posts');
    
    // Status validation
    expect(response.ok()).toBeTruthy();
    expect(response.status()).toBe(200);
    
    // Headers validation
    expect(response.headers()['content-type'])
      .toContain('application/json');
    
    // Body validation
    const posts = await response.json();
    expect(Array.isArray(posts)).toBeTruthy();
    expect(posts).toHaveLength(100);
    
    // First post validation
    expect(posts[0]).toHaveProperty('userId');
    expect(posts[0]).toHaveProperty('id');
    expect(posts[0]).toHaveProperty('title');
    expect(posts[0]).toHaveProperty('body');
  });
  
  test('Get single post by ID', async ({ request }) => {
    const postId = 1;
    const response = await request.get(`/posts/${postId}`);
    
    expect(response.status()).toBe(200);
    
    const post = await response.json();
    expect(post.id).toBe(postId);
    expect(post.userId).toBeGreaterThan(0);
    expect(post.title).toBeTruthy();
    expect(post.body).toBeTruthy();
  });
  
  test('Get posts with query parameters', async ({ request }) => {
    const response = await request.get('/posts', {
      params: {
        userId: 1
      }
    });
    
    expect(response.ok()).toBeTruthy();
    
    const posts = await response.json();
    posts.forEach((post: any) => {
      expect(post.userId).toBe(1);
    });
  });
  
  test('Handle 404 error', async ({ request }) => {
    const response = await request.get('/posts/99999');
    
    expect(response.status()).toBe(404);
  });
  
});
```

---

## 🟢 POST Request Example

```typescript
test.describe('POST Requests', () => {
  
  test('Create new post', async ({ request }) => {
    const newPost = {
      title: 'Test Post',
      body: 'This is a test post',
      userId: 1
    };
    
    const response = await request.post('/posts', {
      data: newPost
    });
    
    expect(response.status()).toBe(201);
    
    const createdPost = await response.json();
    expect(createdPost).toHaveProperty('id');
    expect(createdPost.title).toBe(newPost.title);
    expect(createdPost.body).toBe(newPost.body);
    expect(createdPost.userId).toBe(newPost.userId);
  });
  
  test('Create post and validate response type', async ({ request }) => {
    const response = await request.post('/posts', {
      data: {
        title: 'New Post',
        body: 'Content',
        userId: 1
      }
    });
    
    const post = await response.json();
    
    expect(typeof post.id).toBe('number');
    expect(typeof post.title).toBe('string');
    expect(typeof post.body).toBe('string');
    expect(typeof post.userId).toBe('number');
  });
  
  test('Validate required fields', async ({ request }) => {
    const response = await request.post('/posts', {
      data: {
        // Missing title and body
        userId: 1
      }
    });
    
    // JSONPlaceholder is lenient, but in real API:
    // expect(response.status()).toBe(400);
  });
  
});
```

---

## 🟡 PUT Request Example

```typescript
test.describe('PUT Requests', () => {
  
  test('Update existing post', async ({ request }) => {
    const updatedPost = {
      id: 1,
      title: 'Updated Title',
      body: 'Updated content',
      userId: 1
    };
    
    const response = await request.put('/posts/1', {
      data: updatedPost
    });
    
    expect(response.status()).toBe(200);
    
    const post = await response.json();
    expect(post.title).toBe(updatedPost.title);
    expect(post.body).toBe(updatedPost.body);
    expect(post.id).toBe(1);
  });
  
  test('Partial update with PATCH', async ({ request }) => {
    const response = await request.patch('/posts/1', {
      data: {
        title: 'Partially Updated'
      }
    });
    
    expect(response.ok()).toBeTruthy();
    
    const post = await response.json();
    expect(post.title).toBe('Partially Updated');
  });
  
});
```

---

## 🔴 DELETE Request Example

```typescript
test.describe('DELETE Requests', () => {
  
  test('Delete post', async ({ request }) => {
    const response = await request.delete('/posts/1');
    
    expect(response.status()).toBe(200);
  });
  
  test('Delete and verify', async ({ request }) => {
    // Delete post
    const deleteResponse = await request.delete('/posts/1');
    expect(deleteResponse.ok()).toBeTruthy();
    
    // Verify deletion (in real API, should return 404)
    // const getResponse = await request.get('/posts/1');
    // expect(getResponse.status()).toBe(404);
  });
  
});
```

---

## 🔐 Authentication

### Bearer Token

```typescript
test('API with Bearer token', async ({ request }) => {
  const response = await request.get('/protected-endpoint', {
    headers: {
      'Authorization': 'Bearer YOUR_TOKEN_HERE'
    }
  });
  
  expect(response.ok()).toBeTruthy();
});
```

### Basic Auth

```typescript
test('API with Basic Auth', async ({ playwright }) => {
  const context = await playwright.request.newContext({
    httpCredentials: {
      username: 'user',
      password: 'pass'
    }
  });
  
  const response = await context.get('/secure-endpoint');
  expect(response.ok()).toBeTruthy();
});
```

### API Key

```typescript
test('API with API Key', async ({ request }) => {
  const response = await request.get('/api/data', {
    headers: {
      'X-API-Key': 'your-api-key'
    }
  });
  
  expect(response.ok()).toBeTruthy();
});
```

---

## ✅ Assertions

### Complete Validation Example

```typescript
test('Complete API validation', async ({ request }) => {
  const response = await request.get('/posts/1');
  
  // Status code
  expect(response.ok()).toBeTruthy();
  expect(response.status()).toBe(200);
  expect(response.statusText()).toBe('OK');
  
  // Headers
  const headers = response.headers();
  expect(headers['content-type']).toContain('application/json');
  
  // Response time (approximate)
  const startTime = Date.now();
  await request.get('/posts/1');
  const duration = Date.now() - startTime;
  expect(duration).toBeLessThan(2000);
  
  // Body
  const post = await response.json();
  expect(post).toMatchObject({
    id: expect.any(Number),
    userId: expect.any(Number),
    title: expect.any(String),
    body: expect.any(String)
  });
  
  // Specific values
  expect(post.id).toBe(1);
  expect(post.title.length).toBeGreaterThan(0);
});
```

### Custom Matchers

```typescript
test('Custom assertions', async ({ request }) => {
  const response = await request.get('/posts');
  const posts = await response.json();
  
  // Array assertions
  expect(posts.length).toBeGreaterThan(0);
  expect(posts.length).toBeLessThanOrEqual(100);
  
  // Every item check
  posts.forEach((post: any) => {
    expect(post).toHaveProperty('id');
    expect(post).toHaveProperty('title');
  });
  
  // Schema validation
  expect(posts[0]).toEqual(
    expect.objectContaining({
      userId: expect.any(Number),
      id: expect.any(Number),
      title: expect.any(String),
      body: expect.any(String)
    })
  );
});
```

---

## 🔄 Advanced Features

### Request Context Reuse

```typescript
test.describe('With shared context', () => {
  let apiContext;
  
  test.beforeAll(async ({ playwright }) => {
    apiContext = await playwright.request.newContext({
      baseURL: 'https://jsonplaceholder.typicode.com',
      extraHTTPHeaders: {
        'Accept': 'application/json'
      }
    });
  });
  
  test.afterAll(async () => {
    await apiContext.dispose();
  });
  
  test('Use shared context', async () => {
    const response = await apiContext.get('/posts/1');
    expect(response.ok()).toBeTruthy();
  });
});
```

### Combined UI + API Testing

```typescript
test('Create post via API, verify in UI', async ({ page, request }) => {
  // Create via API
  const response = await request.post('/posts', {
    data: {
      title: 'New Post',
      body: 'Content',
      userId: 1
    }
  });
  
  const createdPost = await response.json();
  
  // Verify in UI (hypothetical)
  await page.goto(`/posts/${createdPost.id}`);
  await expect(page.locator('h1')).toContainText('New Post');
});
```

---

## 🧠 Best Practices

### ✅ DO:
- ✔️ Use `request` fixture for API tests
- ✔️ Validate status, headers, and body
- ✔️ Use TypeScript for type safety
- ✔️ Group related tests in describe blocks
- ✔️ Reuse request context when possible
- ✔️ Combine API and UI testing when needed

### ❌ DON'T:
- ✖️ Hardcode authentication tokens
- ✖️ Skip error scenario testing
- ✖️ Ignore response validation
- ✖️ Create test dependencies

### Helper Functions

```typescript
// helpers/api-helpers.ts
export async function createPost(request, postData) {
  const response = await request.post('/posts', {
    data: postData
  });
  return await response.json();
}

export async function deletePost(request, postId) {
  return await request.delete(`/posts/${postId}`);
}

// Use in tests
test('Using helpers', async ({ request }) => {
  const post = await createPost(request, {
    title: 'Test',
    body: 'Content',
    userId: 1
  });
  
  expect(post.id).toBeDefined();
  
  await deletePost(request, post.id);
});
```

---

## 🚀 Running Tests

```bash
# Run all tests
npx playwright test

# Run specific file
npx playwright test tests/api-basic.spec.ts

# Run with UI mode
npx playwright test --ui

# Generate HTML report
npx playwright test --reporter=html

# Debug mode
npx playwright test --debug
```

---

## 🎓 Next Steps

1. Practice with real APIs
2. Implement data-driven testing
3. Add request/response logging
4. Integrate with CI/CD
5. Combine UI and API testing

**Resources:**
- [Playwright API Testing](https://playwright.dev/docs/api-testing)
- [Playwright Documentation](https://playwright.dev/)

---

<div align="center" markdown="1">

**⭐ Great job! You've completed all three API testing examples! ⭐**

Made with ❤️ for QA-to-SDET Journey | April 2026

</div>
