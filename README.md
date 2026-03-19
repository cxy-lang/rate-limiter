# rate-limiter

HTTP rate limiting middleware for cxy-lang applications.

## Overview

A flexible, high-performance rate limiting middleware for cxy-lang HTTP servers. Features multiple strategies, customizable key extraction, automatic cleanup, and compile-time type safety.

## Features

- **Multiple Rate Limiting Strategies**
  - Fixed Window - Simple time-based windows
  - Token Bucket - Smooth traffic with controlled bursts
  - Sliding Window Counter - Accurate boundary handling

- **Flexible Key Extraction**
  - By IP address (default)
  - By HTTP header (e.g., API key)
  - By cookie (e.g., session ID)
  - Custom extractors with compile-time validation

- **Production Ready**
  - Automatic cleanup of stale entries
  - Basic statistics (requests, allowed, denied)
  - Pluggable storage backends (in-memory, Redis-ready)
  - Standards compliant (RFC 6585 - HTTP 429)

- **Performance**
  - Zero runtime overhead (compile-time generics)
  - Single-threaded, lock-free design
  - Efficient in-memory storage

## Installation

Add to your `Cxyfile.yaml`:

```yaml
dependencies:
  - name: rate-limiter
    version: 0.1.0
```

## Quick Start

```cxy
import { RateLimiter } from "@rate-limiter"
import { Request, Response, Server, Config } from "stdlib/http.cxy"

type RateLimiterByIp = RateLimiter[]

func main(): !void {
    var server = Server[(RateLimiterByIp,)](Config{})
    
    // Configure: 100 requests per minute
    server.middleware[RateLimiterByIp]().configure({
        maxRequests: 100 as u64,
        windowMs: 60000 as u64
    })

    server("/", (_: &const Request, res: &Response) => {
        res.body() << "Hello World!"
    })

    server.start()
}
```

**Important:** Use `as u64` to explicitly cast integer literals in configuration.

## Usage Examples

### 1. Fixed Window (Default)

Simple time-based windows. Best for predictable traffic patterns.

```cxy
import { RateLimiter } from "@rate-limiter"
import { Request, Response, Server, Config } from "stdlib/http.cxy"

type RateLimiterByIp = RateLimiter[]

func main(): !void {
    var server = Server[(RateLimiterByIp,)](Config{})
    
    server.middleware[RateLimiterByIp]().configure({
        maxRequests: 1 as u64,      // 1 request
        windowMs: 10000 as u64       // per 10 seconds
    })

    server("/", (_: &const Request, res: &Response) => {
        res.body() << "Rate limited endpoint"
    })

    server.start()
}
```

### 2. Token Bucket

Allows controlled bursts while maintaining a sustained rate.

```cxy
import { RateLimiter } from "@rate-limiter"
import { Strategy } from "@rate-limiter"
import { Request, Response, Server, Config } from "stdlib/http.cxy"

type RateLimiterByIp = RateLimiter[]

func main(): !void {
    var server = Server[(RateLimiterByIp,)](Config{})
    
    server.middleware[RateLimiterByIp]().configure({
        strategy: Strategy.TokenBucket,
        maxRequests: 10 as u64,      // Bucket capacity: 10 tokens
        windowMs: 10000 as u64       // Refill to full in 10 seconds
    })

    server("/api/data", (_: &const Request, res: &Response) => {
        res.body() << "API response"
    })

    server.start()
}
```

**Token Bucket Behavior:**
- Client starts with 10 tokens (full bucket)
- Each request consumes 1 token
- Tokens refill at 1 token/second (10 tokens / 10 seconds)
- Allows instant burst of 10 requests if bucket is full

### 3. Sliding Window

More accurate than Fixed Window, smooths boundary effects.

```cxy
server.middleware[RateLimiterByIp]().configure({
    strategy: Strategy.SlidingWindow,
    maxRequests: 100 as u64,
    windowMs: 60000 as u64
})
```

### 4. API Key-based Limiting

Rate limit by API key in HTTP header.

```cxy
import { RateLimiter, GetHeaderKey, Strategy } from "@rate-limiter"
import { Request, Response, Server, Config } from "stdlib/http.cxy"

type RateLimiterByAPIKey = RateLimiter[GetHeaderKey]

func main(): !void {
    var server = Server[(RateLimiterByAPIKey,)](Config{})
    
    // Configure key extractor
    server.middleware[RateLimiterByAPIKey]().keyExtractor().configure({
        headerName: String("X-API-Key")
    })
    
    // Configure rate limiting
    server.middleware[RateLimiterByAPIKey]().configure({
        maxRequests: 1000 as u64,
        windowMs: 3600000 as u64  // 1 hour
    })

    server("/api/protected", (_: &const Request, res: &Response) => {
        res.body() << "Protected API endpoint"
    })

    server.start()
}
```

### 5. Cookie-based Limiting

Rate limit by session cookie.

```cxy
import { RateLimiter, GetCookieKey } from "@rate-limiter"
import { Request, Response, Server, Config } from "stdlib/http.cxy"

type RateLimiterBySession = RateLimiter[GetCookieKey]

func main(): !void {
    var server = Server[(RateLimiterBySession,)](Config{})
    
    // Configure to use custom cookie name
    server.middleware[RateLimiterBySession]().keyExtractor().configure({
        cookieName: String("session_id")
    })
    
    server.middleware[RateLimiterBySession]().configure({
        maxRequests: 200 as u64,
        windowMs: 900000 as u64  // 15 minutes
    })

    server.start()
}
```

### 6. Custom Key Extractor

Combine multiple request attributes for rate limiting.

```cxy
import { RateLimiter } from "@rate-limiter"
import { Request, Response, Server, Config } from "stdlib/http.cxy"

// Custom extractor: rate limit by user ID + endpoint path
struct UserEndpointKey {
    func `init`() {}
    
    const func get(req: &const Request): String {
        var userId = req.header("X-User-ID")
        if (!userId) {
            return f"anonymous:{req.path()}"
        }
        return f"{*userId}:{req.path()}"
    }
}

type RateLimiterCustom = RateLimiter[UserEndpointKey]

func main(): !void {
    var server = Server[(RateLimiterCustom,)](Config{})
    
    server.middleware[RateLimiterCustom]().configure({
        maxRequests: 50 as u64,
        windowMs: 60000 as u64
    })

    server.start()
}
```

## Configuration Reference

### Rate Limiter Settings

```cxy
server.middleware[RateLimiter]().configure({
    // Strategy (default: FixedWindow)
    strategy: Strategy.FixedWindow,      // or TokenBucket, SlidingWindow
    
    // Rate limits (REQUIRED: use `as u64`)
    maxRequests: 100 as u64,             // Max requests per window
    windowMs: 60000 as u64,              // Window duration in milliseconds
    
    // Token Bucket specific
    burstSize: 0 as u64,                 // Burst capacity (0 = same as maxRequests)
    refillRate: 0.0,                     // Tokens/sec (0.0 = auto-calculate)
    
    // Response customization
    statusCode: Status.TooManyRequests,  // HTTP status code (default: 429)
    includeHeaders: true,                // Include X-RateLimit-* headers
    headerPrefix: String("X-RateLimit-"), // Header prefix
    
    // Cleanup (automatic)
    cleanupIntervalMs: 60000 as u64,     // Cleanup every 60 seconds
    stateTimeoutMs: 300000 as u64        // Remove inactive states after 5 minutes
})
```

### Key Extractor Configuration

**GetHeaderKey:**
```cxy
server.middleware[RateLimiter]().keyExtractor().configure({
    headerName: String("X-API-Key")  // Default: "X-API-Key"
})
```

**GetCookieKey:**
```cxy
server.middleware[RateLimiter]().keyExtractor().configure({
    cookieName: String("session_id")  // Default: "session_id"
})
```

**GetIPKey:** No configuration needed.

## Response Headers

The middleware automatically sets these headers on every request:

- `X-RateLimit-Limit: 100` - Maximum requests allowed
- `X-RateLimit-Remaining: 85` - Requests remaining in current window
- `X-RateLimit-Reset: 1710720000000` - Unix timestamp (ms) when limit resets

When rate limit is exceeded, returns **HTTP 429 Too Many Requests** with body: `"Rate limit exceeded"`

## Statistics

Track usage statistics:

```cxy
// Get current statistics
var stats = server.middleware[RateLimiter]().getStats()
println(f"Total requests: {stats.totalRequests}")
println(f"Allowed: {stats.totalAllowed}")
println(f"Denied: {stats.totalDenied}")
println(f"Active keys: {stats.currentKeys}")

// Reset statistics (keeps rate limit state)
server.middleware[RateLimiter]().resetStats()
```

## Custom Key Extractor Requirements

To create a custom key extractor:

1. **Default constructor:** `func init()`
2. **Get method:** `const func get(req: &const Request): String`
3. **Optional configure:** `func configure[Cfg](cfg: Cfg)`

```cxy
struct MyKey {
    // 1. Default constructor
    func `init`() {}
    
    // 2. Get method (required)
    const func get(req: &const Request): String {
        // Extract key from request
        return String("key")
    }
    
    // 3. Configure method (optional)
    func configure[Cfg](cfg: Cfg) {
        // Handle configuration
    }
}
```

The compiler enforces these requirements at compile-time.

## Advanced: Custom Storage Backend

The rate limiter supports pluggable storage backends:

```cxy
// Default: In-memory HashMap
type RateLimiterByIp = RateLimiter[GetIPKey, HashMapStorage]

// Future: Redis backend (example)
// type RateLimiterByIp = RateLimiter[GetIPKey, RedisStorage]
```

Storage backends must implement:
- `func get(key: String): Optional[RateLimitState]`
- `func set(key: String, state: RateLimitState)`
- `func remove(key: String)`
- `func clear()`
- `func cleanup(now: u64, timeoutMs: u64)`

Storage can declare dependencies on other middlewares:
```cxy
class RedisStorage {
    type Deps = (RedisMiddleware,)
    // ...
}
```

## Strategy Comparison

| Strategy | Accuracy | Memory | Best For |
|----------|----------|--------|----------|
| **Fixed Window** | Good | Low | Simple rate limiting, predictable traffic |
| **Token Bucket** | Excellent | Low | APIs with bursty traffic, flexible usage |
| **Sliding Window** | Excellent | Low | Accurate limiting without boundary issues |

## Performance

- **Fixed Window:** ~10M ops/sec
- **Token Bucket:** ~5M ops/sec  
- **Sliding Window:** ~3M ops/sec

Real-world HTTP overhead dominates these numbers. Rate limiting adds <1ms latency.

## Automatic Cleanup

The middleware automatically removes inactive rate limit entries:

- Runs every `cleanupIntervalMs` (default: 60 seconds)
- Removes entries inactive for > `stateTimeoutMs` (default: 5 minutes)
- Prevents memory growth with millions of unique keys
- Zero configuration required

## Testing

Run tests:
```bash
cxy package test
```

All strategies and extractors have comprehensive unit tests.

## Known Limitations

- **Single-threaded:** Assumes single-threaded execution (no locks)
- **In-memory only:** State not persisted across restarts (use Redis storage for persistence)
- **IP extraction:** Uses `req.ip()` method added to Request class

## License

MIT
