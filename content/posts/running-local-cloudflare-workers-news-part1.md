---
title: "Running Local Cloudflare Workers to Gather News Information - Part 1"
date: 2025-04-07T22:52:51+03:00
draft: false
tags: ["cloudflare", "workers", "web scraping", "typescript", "serverless"]
categories: ["Development", "Cloud"]
---

## Introduction to Cloudflare Workers

Cloudflare Workers represent a paradigm shift in how we build and deploy applications on the web. Unlike traditional server-based applications, Cloudflare Workers run on Cloudflare's edge network, meaning they execute closer to your users and provide impressive performance benefits.

Key advantages of Cloudflare Workers include:

- **Edge Execution**: Code runs on Cloudflare's global network, reducing latency
- **Serverless Architecture**: No servers to manage or scale
- **Cost-Effective**: Pay only for what you use with generous free tier
- **JavaScript/TypeScript Native**: Write in familiar languages
- **Powerful API Access**: Built-in fetch, KV storage, and more

In this series, I'll walk through how I built a Cloudflare Worker that collects news articles from Nation Africa (https://nation.africa) for personal use, and how you might adapt this approach for other sites.

## Project Overview and Goals

The goal of this project was to create a reliable way to fetch recent news headlines from Nation Africa, a popular news source in Kenya. I wanted to:

1. Retrieve the latest news headlines and their URLs
2. Structure the data in a clean JSON format for downstream consumption
3. Run the collection on a schedule (every 12 hours)
4. Handle authentication and maintain session cookies properly
5. Build a robust solution that can adapt to site changes

This type of edge-deployed microservice is perfect for data collection tasks that don't require heavy processing but benefit from reliability and scheduling.

## Initial Setup and Environment Configuration

To get started with Cloudflare Workers, you'll need a Cloudflare account and the Wrangler CLI tool. Here's how I set up the project:

### 1. Installing the Required Tools

First, ensure you have Node.js and npm installed, then set up the project:

```bash
# Install Wrangler CLI globally
npm install -g wrangler

# Create a new TypeScript Cloudflare Worker project
npx create-cloudflare nation-worker-ts
cd nation-worker-ts

# Login to Cloudflare
wrangler login
```

### 2. Project Structure

The project structure created by the CLI looks like this:

```
nation-worker-ts/
├── src/
│   └── index.ts       # Main worker code
├── .wrangler/
├── node_modules/
├── package.json
├── package-lock.json
├── tsconfig.json
└── wrangler.jsonc     # Worker configuration
```

### 3. Configuring wrangler.jsonc

The `wrangler.jsonc` file is where we define the Worker's configuration. Here's my basic configuration:

```json
{
  "name": "nation-worker-ts",
  "main": "src/index.ts",
  "compatibility_date": "2025-04-04",
  "triggers": {
    "crons": ["0 */12 * * *"]  // Run every 12 hours
  },
  "kv_namespaces": [
    {
      "binding": "KV",
      "id": "your-kv-namespace-id",
      "preview_id": "your-preview-kv-namespace-id"
    }
  ]
}
```

### 4. Setting Up Environment Variables

For our worker to function, we need to define a few environment variables. I've configured these in the `wrangler.jsonc` file:

```json
{
  "vars": {
    "ADMIN_KEY": "your-admin-key"
  }
}
```

This ADMIN_KEY will be used for authentication when refreshing cookies through the API.

### 5. Setting Up KV Storage

Cloudflare Workers KV is a global, low-latency key-value data store that we'll use to store our session cookies. To create a KV namespace:

```bash
# Create KV namespace 
wrangler kv:namespace create "KV"

# This will output IDs to add to your wrangler.jsonc file
```

## Basic Worker Structure

The base structure of our Cloudflare Worker includes two main entry points:

1. **HTTP Requests** - The `fetch` handler processes incoming HTTP requests
2. **Scheduled Execution** - The `scheduled` handler runs on our defined cron schedule

Here's a simplified version of our initial worker code:

```typescript
export interface Env {
  KV: KVNamespace;
  ADMIN_KEY?: string;
}

export default {
  // Handle HTTP requests
  async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise<Response> {
    // Request handling code here
    return new Response("Hello World");
  },
  
  // Handle scheduled events (cron)
  async scheduled(event: ScheduledEvent, env: Env, ctx: ExecutionContext): Promise<void> {
    console.log("Running scheduled task");
    // Scheduled task code here
  }
};
```

## Next Steps

In [Part 2](/posts/running-local-cloudflare-workers-news-part2), we'll dive into the core implementation details:

- How to manage session cookies for authenticated requests
- Techniques for fetching and parsing news articles
- Implementing error handling and debugging features

By the end of this series, you'll have a fully functional Cloudflare Worker that can reliably collect news articles and deliver them as structured JSON data.

