---
title: "Running Local Cloudflare Workers to Gather News Information - Part 2"
date: 2025-04-07T22:53:51+03:00
draft: false
tags: ["cloudflare", "workers", "web scraping", "typescript", "serverless"]
categories: ["Development", "Cloud"]
---

## Introduction

In [Part 1](/posts/running-local-cloudflare-workers-news-part1), we covered the basics of Cloudflare Workers and set up our project. Now, let's dive into the core implementation details that make our news gathering worker function.

This post focuses on three crucial aspects:

1. Cookie management for authenticated access
2. Article fetching and parsing techniques
3. Error handling and debugging strategies

## Cookie Management System

Many modern websites, including Nation Africa, use cookies for session management and paywalls. To access full content, we need to maintain valid session cookies.

### Storing Cookies in KV

We use Cloudflare's KV (Key-Value) store to persist cookies between worker invocations:

```typescript
interface Env {
  KV: KVNamespace; // For session cookie storage
  ADMIN_KEY?: string; // Optional authentication key for refreshing cookies
}

async function getSessionCookie(env: Env): Promise<string> {
  // Try to get the cookie from KV
  const cookie = await env.KV.get("session_cookie");
  
  if (!cookie) {
    throw new Error("Cookie not found. Run the extract-cookies.js script to update.");
  }
  
  // If the cookie is explicitly expired, warn but still use it
  if (isCookieExpired(cookie)) {
    console.warn("Warning: Cookie may be expired. Consider running extract-cookies.js.");
  }
  
  return cookie;
}
```

### Checking Cookie Expiration

To ensure we're using valid cookies, I implemented a function to check cookie expiration:

```typescript
function isCookieExpired(cookie: string): boolean {
  // Extract expiration from cookie if it exists
  const expiresMatch = cookie.match(/expires=([^;]+)/i);
  if (!expiresMatch) return false; // Assume not expired if no expiration date
  
  const expiresDate = new Date(expiresMatch[1]);
  const now = new Date();
  
  // If the cookie expires in less than 30 minutes, consider it expired
  const thirtyMinutesFromNow = new Date(now.getTime() + 30 * 60 * 1000);
  return expiresDate < thirtyMinutesFromNow;
}
```

### Cookie Refresh Process

Instead of handling the login process directly in the worker (which could be complex and prone to breaking), I created a separate script `extract-cookies.js` that runs locally to refresh cookies.

The workflow is:
1. Run the script locally which performs browser automation
2. The script extracts fresh cookies from a successful login
3. The cookies are uploaded to the Worker's KV storage
4. The Worker uses these cookies for subsequent requests

This separation of concerns provides greater reliability and easier maintenance.

## Article Fetching and Parsing

The core functionality of our worker is fetching and parsing news articles from the target website.

### Making Authenticated Requests

To fetch article data, we make requests to the news site with our stored cookies:

```typescript
async function fetchArticles(cookie: string): Promise<{ title: string; url: string }[]> {
  const response = await fetch(NEWS_URL, {
    headers: {
      "Cookie": cookie,
      // Add user agent to appear more like a regular browser
      "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36"
    },
  });

  if (!response.ok) {
    throw new Error(`Failed to fetch articles: ${response.status}`);
  }

  const html = await response.text();
  // Parsing logic follows...
}
```

### HTML Parsing with Regular Expressions

For this project, I opted to use regular expressions for HTML parsing. While regex isn't generally recommended for HTML parsing due to the complexity of HTML, it's suitable for our targeted extraction needs and eliminates the need for heavy dependencies.

Here's a simplified version of the parsing logic:

```typescript
const articleRegex = /<h[2-4](?:[^>]*)class="(?:[^"]*?)(?:title|heading)(?:[^"]*?)"[^>]*>(?:\s*)<a[^>]*href="([^"]*)"[^>]*>(.*?)<\/a>/gi;
const articles: { title: string; url: string }[] = [];

let match;
while ((match = articleRegex.exec(html)) !== null) {
  const url = match[1].trim();
  const title = match[2].replace(/<[^>]*>/g, '').trim(); // Strip any HTML tags inside title
  
  articles.push({
    url: url.startsWith("http") ? url : `https://nation.africa${url}`,
    title: title,
  });
}
```

This extracts article titles and URLs from heading elements that contain links.

## Error Handling and Debugging

Robust error handling is crucial for a production worker that runs on a schedule without direct supervision.

### Comprehensive Error Handling

Throughout the code, I've implemented try-catch blocks to prevent catastrophic failures:

```typescript
export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise<Response> {
    try {
      // Worker logic here
    } catch (error) {
      console.error("Worker error:", error.message);
      return new Response(JSON.stringify({ error: error.message }), {
        status: 500,
        headers: { "Content-Type": "application/json" },
      });
    }
  }
}
```

### Status Endpoint

To easily check the current state of our worker, I implemented a `/cookie-status` endpoint:

```typescript
if (url.pathname === "/cookie-status") {
  try {
    const cookie = await env.KV.get("session_cookie");
    const cookiePresent = !!cookie;
    const cookieLength = cookie ? cookie.length : 0;
    
    return new Response(JSON.stringify({
      cookiePresent,
      cookieLength,
      lastUpdated: await env.KV.get("cookie_last_updated") || "unknown",
      isExpired: cookie ? isCookieExpired(cookie) : true
    }), {
      status: 200,
      headers: { "Content-Type": "application/json" }
    });
  } catch (error) {
    return new Response(JSON.stringify({ 
      error: "Failed to check cookie status", 
      details: error.message 
    }), {
      status: 500,
      headers: { "Content-Type": "application/json" }
    });
  }
}
```

This endpoint provides quick visibility into the state of our cookie storage, which is crucial for troubleshooting.

### Logging

Cloudflare Workers supports console logging, which is helpful for monitoring and debugging:

```typescript
console.log("Scheduled task running at:", new Date().toISOString());
console.log("Using cookie (first 30 chars):", cookie.substring(0, 30) + "...");
console.log("Article fetch response status:", response.status);
console.log(`Found ${articles.length} articles`);
```

These logs can be viewed in the Cloudflare Workers dashboard or with the Wrangler CLI during development.

## Testing the Worker Locally

Before deployment, it's essential to test the worker locally. Wrangler provides a development server:

```bash
npm run dev
```

This starts a local server, typically at http://localhost:8787, where you can test your worker's HTTP endpoints.

For scheduled events, you can trigger them manually with:

```bash
curl "http://localhost:8787/__scheduled?cron=*+*+*+*+*"
```

## Conclusion

In this part, we've covered the core implementation details of our news gathering Cloudflare Worker:

- Storing and managing cookies in KV storage
- Fetching and parsing articles using regular expressions
- Implementing robust error handling and debugging endpoints

These components form the backbone of our worker's functionality.

In [Part 3](/posts/running-local-cloudflare-workers-news-part3), we'll explore advanced features and deployment considerations:

- Multiple pattern matching for robust article extraction
- Comprehensive debugging endpoints
- Deployment strategies and maintenance best practices

Stay tuned for the final part of this series!

