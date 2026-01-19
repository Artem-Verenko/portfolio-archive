---
title: "Building Modern Web Applications with Vue 3 and GitHub API"
date: "2024-01-19"
tags: ["Vue.js", "GitHub API", "Web Development", "JavaScript"]
summary: "A comprehensive guide to building dynamic web applications that leverage GitHub as a content management system, featuring Vue 3 composition API and modern development practices."
---

# Building Modern Web Applications with Vue 3 and GitHub API

## Introduction

In the evolving landscape of web development, the separation of content from code has become increasingly important. This article explores how to build a modern, performant web application using **Vue 3** and the **GitHub API** for content management.

## Why Use GitHub as a CMS?

GitHub offers several compelling advantages when used as a content management system:

- **Version Control**: Every content change is tracked with full Git history
- **Collaboration**: Multiple contributors can work simultaneously with pull requests
- **Free Hosting**: Content storage is free for public repositories
- **API Access**: Robust RESTful API with excellent documentation
- **Markdown Support**: Write content in a developer-friendly format
- **Image Hosting**: Store images alongside your content

## Architecture Overview

Our architecture follows a clean separation of concerns:

```javascript
// Service Layer Pattern
const contentService = {
  async fetchArticle(id) {
    // 1. Check cache first
    const cached = cache.get(`article_${id}`);
    if (cached) return cached;

    // 2. Fetch from GitHub
    const markdown = await github.getFile(`articles/article-${id}.md`);

    // 3. Parse and cache
    const parsed = markdownIt.render(markdown);
    cache.set(`article_${id}`, parsed);

    return parsed;
  },
};
```

### Key Components

1. **GitHub Client** - Wraps Octokit for API calls
2. **Cache Layer** - LocalStorage with TTL support
3. **Markdown Parser** - Converts markdown to HTML
4. **Vue Composables** - TanStack Query integration

## Implementation Steps

### Step 1: Setting Up GitHub Integration

First, create a fine-grained personal access token:

```bash
# Required permissions:
# - Repository: Contents (Read)
# - No other permissions needed
```

Configure your environment:

```env
VITE_GITHUB_TOKEN=github_pat_YOUR_TOKEN
VITE_GITHUB_OWNER=your-username
VITE_GITHUB_REPO=content-repo
VITE_GITHUB_BRANCH=main
```

### Step 2: Creating the Content Structure

Organize your content repository:

```
content-repo/
├── articles/
│   ├── articles.json      # Manifest with metadata
│   ├── article-1.md       # Markdown content
│   ├── article-2.md
│   └── images/
│       └── diagram.png
├── books/
│   ├── db_books.json
│   └── reviews/
│       └── book-1.md
└── cv/
    └── resume.pdf
```

### Step 3: Implementing the Cache Layer

Caching is crucial for performance and offline support:

```javascript
class CacheService {
  set(key, value, ttl = 3600000) {
    const item = {
      value,
      expiry: Date.now() + ttl,
      timestamp: Date.now(),
    };
    localStorage.setItem(key, JSON.stringify(item));
  }

  get(key) {
    const item = JSON.parse(localStorage.getItem(key));
    if (!item) return null;

    if (Date.now() > item.expiry) {
      localStorage.removeItem(key);
      return null;
    }

    return item.value;
  }
}
```

## Best Practices

> **Important**: Always implement proper error handling and fallback mechanisms when working with external APIs.

### Error Handling Strategy

Follow these principles for robust error handling:

1. **Graceful Degradation** - Show cached content when API fails
2. **User Feedback** - Clear loading states and error messages
3. **Retry Logic** - Automatic retry with exponential backoff
4. **Rate Limiting** - Respect GitHub's API limits (5000 req/hour)

### Performance Optimization

- ✅ Implement aggressive caching (1-24 hour TTL)
- ✅ Use TanStack Query for automatic background refetching
- ✅ Lazy load content with Vue's `defineAsyncComponent`
- ✅ Compress images before uploading to GitHub
- ✅ Use GitHub's CDN for image delivery

## Code Examples

### Vue Composable with TanStack Query

```javascript
import { useQuery } from "@tanstack/vue-query";

export function useArticle(id) {
  return useQuery({
    queryKey: ["article", id],
    queryFn: () => ArticleService.getArticle(id),
    staleTime: 5 * 60 * 1000, // 5 minutes
    cacheTime: 60 * 60 * 1000, // 1 hour
    retry: 2,
  });
}
```

### Component Usage

```vue
<template>
  <div v-if="isLoading">Loading...</div>
  <div v-else-if="error">{{ error.message }}</div>
  <article v-else class="markdown-body" v-html="article.html" />
</template>

<script setup>
import { useArticle } from "@/composables/useContent";

const props = defineProps(["id"]);
const { data: article, isLoading, error } = useArticle(props.id);
</script>
```

## Advanced Features

### Image Path Resolution

Automatically resolve relative image paths in markdown:

```javascript
// Before: ![Diagram](./images/diagram.png)
// After: https://raw.githubusercontent.com/user/repo/main/articles/images/diagram.png

markdownIt.renderer.rules.image = (tokens, idx) => {
  const src = tokens[idx].attrs[0][1];
  if (isRelativePath(src)) {
    return resolveGitHubUrl(src);
  }
  return defaultRender(tokens, idx);
};
```

### Syntax Highlighting

Use Prism.js or highlight.js for beautiful code blocks:

```python
def fibonacci(n):
    """Generate Fibonacci sequence up to n"""
    a, b = 0, 1
    while a < n:
        print(a, end=' ')
        a, b = b, a + b
    print()

fibonacci(100)
```

## Common Pitfalls

### Authentication Issues

**Problem**: 401 Unauthorized errors

**Solution**:

- Verify token hasn't expired
- Check token has Contents (Read) permission
- Ensure token has access to repository

### Rate Limiting

**Problem**: 403 Rate Limit Exceeded

**Solution**:

- Implement caching to reduce API calls
- Use authenticated requests (5000/hour vs 60/hour)
- Monitor rate limit headers in responses

### CORS Issues

**Problem**: CORS errors in development

**Solution**:

- GitHub API supports CORS out of the box
- Ensure you're making requests to `api.github.com`
- Don't proxy requests unnecessarily

## Performance Metrics

Based on real-world testing:

| Metric            | Without Cache | With Cache |
| ----------------- | ------------- | ---------- |
| First Load        | ~800ms        | ~800ms     |
| Subsequent Loads  | ~800ms        | ~50ms      |
| Offline Support   | ❌            | ✅         |
| API Calls/Session | ~50           | ~5         |

## Deployment Considerations

### Environment Variables

Never commit your `.env.local` file! Use your hosting platform's environment variable system:

- **Vercel**: Project Settings → Environment Variables
- **Netlify**: Site Settings → Build & Deploy → Environment
- **GitHub Pages**: Secrets in repository settings

### Build Optimization

```javascript
// vite.config.js
export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          "github-api": ["@octokit/rest"],
          markdown: ["markdown-it"],
          "vue-query": ["@tanstack/vue-query"],
        },
      },
    },
  },
});
```

## Testing Strategy

### Unit Tests

```javascript
import { describe, it, expect, vi } from "vitest";
import { ArticleService } from "@/services/content/ArticleService";

describe("ArticleService", () => {
  it("should fetch article from cache", async () => {
    const cacheService = vi.mock("CacheService");
    cacheService.get.mockReturnValue({ title: "Test" });

    const article = await ArticleService.getArticle("1");
    expect(article.title).toBe("Test");
  });
});
```

### Integration Tests

Use Playwright or Cypress to test the full workflow:

```javascript
test("should display article from GitHub", async ({ page }) => {
  await page.goto("/blog/articles/1");
  await expect(page.locator("h1")).toContainText("Article Title");
  await expect(page.locator(".markdown-body")).toBeVisible();
});
```

## Conclusion

Using GitHub as a CMS for your Vue application provides a robust, version-controlled, and developer-friendly solution. The combination of Vue 3's Composition API, TanStack Query's data management, and GitHub's reliable infrastructure creates a powerful stack for modern web applications.

### Key Takeaways

- 🎯 **Separation of Concerns**: Content lives separately from code
- 🚀 **Performance**: Aggressive caching reduces API calls
- 🔄 **Version Control**: Full Git history for content changes
- 💰 **Cost-Effective**: Free for public repositories
- 🛠️ **Developer-Friendly**: Write in Markdown, manage with Git

## Further Reading

- [Vue 3 Composition API Documentation](https://vuejs.org/guide/extras/composition-api-faq.html)
- [TanStack Query Guide](https://tanstack.com/query/latest)
- [GitHub REST API Reference](https://docs.github.com/en/rest)
- [Markdown-it Documentation](https://github.com/markdown-it/markdown-it)
- [Fine-grained Tokens Guide](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token)

## About the Author

This article demonstrates the exact architecture used in my portfolio application. Feel free to explore the source code and adapt it for your own projects.

---

**Published**: January 19, 2024  
**Last Updated**: January 19, 2024  
**Reading Time**: ~15 minutes

_Questions or feedback? Open an issue on GitHub!_
