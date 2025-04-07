---
title: "Running Local Cloudflare Workers to Gather News Information - Part 3"
date: 2025-04-07T22:54:51+03:00
draft: false
tags: ["cloudflare", "workers", "web scraping", "typescript", "serverless"]
categories: ["Development", "Cloud"]
---

## Introduction

In the [first part](/posts/running-local-cloudflare-workers-news-part1) of this series, we explored the basics of Cloudflare Workers and set up our project. The [second part](/posts/running-local-cloudflare-workers-news-part2) covered core implementation details like cookie management and article parsing.

Now, in this final installment, we'll dive into the advanced features that make our news scraping worker robust and maintainable:

1. Multiple pattern matching techniques for resilient scraping
2. Comprehensive debugging endpoints
3. Deployment strategies and maintenance considerations

## Multiple Pattern Matching for Robust Scraping

One of the biggest challenges in web scraping is handling website changes. News sites frequently update their layouts and HTML structure, which can break simple scraping approaches. To build a resilient solution, I implemented a multi-tiered approach to article extraction.

### The Problem with Single Pattern Matching

Initially, I used a simple regex pattern to extract articles:

```typescript
const articleRegex = /<h3 class="article-title.*?"><a href="(.*?)">(.*?)<\/a>/g;
```

This worked fine until the site changed their HTML structure, causing the worker to return zero results. Instead of constantly updating the regex when the site changes, I implemented a more robust approach.

### Implementing Multiple Pattern Strategy

The core of this approach is a function called `tryMultiplePatterns` that attempts several different extraction methods in succession:

```typescript
async function tryMultiplePatterns(html: string, env: Env): Promise<{ title: string; url: string }[]> {
  // Store sample data for debugging
  await env.KV.put("debug_html_sample", html.substring(0, 10000));
  await env.KV.put("debug_timestamp", new Date().toISOString());
  
  // Try multiple approaches to extract articles...
}
```

The function includes three distinct approaches:

1. **Article Element Parsing**: First, it tries to find all `<article>` elements and extract title/URL pairs from them
2. **H3 Teaser Parsing**: If that fails, it looks for heading elements with specific classes 
3. **Broad Pattern Fallback**: As a last resort, it uses a more general pattern to capture content

Here's how the first approach looks:

```typescript
// Find all article elements
const articleElements = [];
const articleRegex = /<article[^>]*>([\s\S]*?)<\/article>/gi;
let articleMatch;

while ((articleMatch = articleRegex.exec(html)) !== null) {
  articleElements.push(articleMatch[0]);
}

// Process each article element
for (const articleHtml of articleElements) {
  try {
    // Extract URL and title from the article element
    const urlMatch = articleHtml.match(/<a[^>]*href="([^"]*)"[^>]*>/);
    if (!urlMatch) continue;
    
    const titleMatch = articleHtml.match(/<h3[^>]*>([\s\S]*?)<\/h3>/);
    if (!titleMatch) continue;
    
    // Clean up the title...
    
    articles.push({
      url: urlMatch[1].startsWith("http") ? urlMatch[1] : `https://nation.africa${urlMatch[1]}`,
      title: title
    });
  } catch (error) {
    console.error("Error processing article element:", error);
  }
}
```

### Post-Processing for Quality Results

After extracting articles with any of the methods, I apply additional filtering to ensure quality results:

```typescript
// Filter out navigation links
const filteredArticles = articles.filter(article => {
  // Skip navigation links which typically contain just a single word or section name
  const isNavigationLink = article.title.trim().split(/\s+/).length <= 2 && 
                         (article.url.includes("/news/") || 
                          article.url.includes("/section/") || 
                          article.url.includes("/category/"));
  return !isNavigationLink;
});

// Remove duplicates by URL
const uniqueArticles = Array.from(
  new Map(filteredArticles.map(article => [article.url, article])).values()
);
```

This multi-pattern approach with post-processing ensures that our worker continues to extract articles even when the site undergoes design changes. If one pattern fails, another will likely succeed.

## Comprehensive Debugging Endpoints

For a production service, especially one that runs on a schedule, having robust debugging capabilities is crucial. I implemented several endpoints to help diagnose issues without having to deploy new code.

### Debug Endpoint

The most powerful debugging feature is a dedicated `/debug` endpoint that provides comprehensive information about the current state:

```typescript
if (url.pathname === "/debug") {
  try {
    const cookie = await getSessionCookie(env);
    console.log("Using cookie (first 30 chars):", cookie.substring(0, 30) + "...");
    
    const response = await fetch(NEWS_URL, {
      headers: {
        "Cookie": cookie,
        "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36"
      },
    });
    
    console.log("Article fetch response status:", response.status);
    
    if (!response.ok) {
      const text = await response.text();
      console.error("Error response preview:", text.substring(0, 200) + "...");
      throw new Error(`Failed to fetch articles: ${response.status}`);
    }
    
    const html = await response.text();
    const articles = await tryMultiplePatterns(html, env);
    
    return new Response(JSON.stringify({
      status: "success",
      articleCount: articles.length,
      articles: articles,
      debugInfo: {
        responseStatus: response.status,
        htmlLength: html.length,
        timestamp: new Date().toISOString()
      }
    }), {
      status: 200,
      headers: { "Content-Type": "application/json" },
    });
  } catch (error) {
    return new Response(JSON.stringify({ 
      error: "Debug failed", 
      details: error.message 
    }), {
      status: 500,
      headers: { "Content-Type": "application/json" }
    });
  }
}
```

This endpoint not only fetches and processes articles but also returns detailed diagnostic information including:

- Response status from the news site
- Number of articles found
- Full article data
- HTML length
- Timestamp of the request

### Storing Debug Information in KV

In addition to the real-time debug endpoint, I store samples of key data in KV storage for later analysis:

```typescript
// Store a sample of the HTML in KV for debugging
await env.KV.put("debug_html_sample", html.substring(0, 10000));
await env.KV.put("debug_timestamp", new Date().toISOString());

// Store h3 samples for debugging
const h3Tags = html.match(/<h3[^>]*>.*?<\/h3>/gs) || [];
await env.KV.put("debug_h3_samples", h3Tags.slice(0, 10).join('\n\n'));

// Store some article elements for debugging
await env.KV.put("debug_article_elements", articleElements.slice(0, 5).join('\n\n---\n\n'));
await env.KV.put("debug_article_count", articleElements.length.toString());

// Store extracted articles
await env.KV.put("debug_extracted_articles", JSON.stringify({
  count: articles.length,
  articles: articles.slice(0, 20) // Store first 20 articles for debugging
}));
```

This approach creates a historical record that can be examined if issues arise later, even if they're intermittent or cannot be reproduced on demand.

## Deployment and Maintenance Considerations

Running a Cloudflare Worker in production requires careful thought about deployment, monitoring, and maintenance.

### Deployment Strategy

For deploying the worker, I use Wrangler's deployment capabilities:

```bash
# Deploy to production
wrangler deploy
```

However, to ensure reliable deployments, I've implemented a few best practices:

1. **Version Control**: All changes are committed to git before deployment
2. **Environment Variables**: Sensitive information is stored as environment variables or KV entries
3. **Testing**: Local testing before deployment using `wrangler dev`

### Monitoring and Alerting

Cloudflare provides basic monitoring for Workers, but for more comprehensive monitoring, consider:

- Setting up status checks that ping your worker endpoints
- Implementing a logging service to capture console logs
- Creating a dashboard for KV storage status

I've implemented simple self-monitoring through KV storage timestamps that track when the worker last ran successfully.

### Cookie Management Maintenance

The most maintenance-intensive part of this worker is cookie management. Since cookies expire and login methods can change, I've separated this concern:

1. **External Cookie Management**: Using a separate `extract-cookies.js` script for refreshing cookies
2. **Cookie Expiration Checks**: Regular checks to warn about soon-to-expire cookies
3. **Manual Refresh Process**: Documented steps for updating cookies when needed

Here's the workflow I use for cookie maintenance:

```
1. Run the extract-cookies.js script locally:
   node extract-cookies.js

2. The script:
   - Opens a headless browser
   - Navigates to the login page
   - Completes authentication
   - Extracts cookies
   - Updates the Worker's KV storage

3. Verify with the /cookie-status endpoint
```

This separation of concerns makes maintenance more manageable, as the cookie extraction process can be updated independently of the worker logic.

### Handling Site Changes

News sites change frequently, and here's my strategy for dealing with that:

1. **Monitor Extraction Results**: Set up regular checks of article count
2. **Update Patterns as Needed**: If extraction fails, add new patterns to the `tryMultiplePatterns` function
3. **Resilient Design**: The multiple-pattern approach often adapts automatically

### Cost Considerations

Cloudflare Workers offers generous free tier limits, but it's good to be aware of usage:

- 100,000 requests per day on the free plan
- 10ms CPU time per request (adequate for our parsing needs)
- KV storage limits (1GB storage, 100,000 reads/day, 1,000 writes/day)

Our implementation uses minimal resources, scheduling runs only every 12 hours, keeping us well within free tier limits.

## Conclusion

Throughout this series, we've built a robust Cloudflare Worker that reliably collects news headlines from Nation Africa. We've implemented:

- A solid foundation with proper configuration and environment setup
- Reliable cookie management and article parsing
- Multiple pattern matching for resilience against site changes
- Comprehensive debugging capabilities
- Sensible deployment and maintenance practices

This approach can be adapted to many other web scraping tasks where you need reliable, scheduled data collection with minimal maintenance overhead.

The complete code is available in my GitHub repository, and I hope this series has provided you with insights into building and maintaining serverless web scrapers using Cloudflare Workers.

Happy coding!
