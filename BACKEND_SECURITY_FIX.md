# Backend Security Fix Required

## Problem

The deployed frontend at `https://pages.opencodingsociety.com` is getting **Mixed Content** errors when calling the Spring backend at `spring.opencodingsociety.com`.

### What's happening

1. Frontend makes a fetch to `https://spring.opencodingsociety.com/api/...` (correct, HTTPS)
2. Spring Security intercepts unauthenticated requests and issues a **302 redirect** to its login page
3. The redirect URL is generated as `http://spring.opencodingsociety.com/login` (HTTP, not HTTPS)
4. The browser blocks this because loading an HTTP resource from an HTTPS page is **Mixed Content**
5. All calendar API calls fail with `TypeError: Failed to fetch`

### Console errors seen

```
Mixed Content: The page at 'https://pages.opencodingsociety.com/student/calendar'
was loaded over HTTPS, but requested an insecure resource
'http://spring.opencodingsociety.com/login'. This request has been blocked.
```

## Root Cause

Spring Security doesn't know it's being served behind HTTPS (likely behind a reverse proxy like Nginx or a load balancer). It generates redirect URLs using `http://` by default.

## Backend Fix (Spring Boot)

### Option 1: Configure Spring to trust forwarded headers (Recommended)

In `application.properties` or `application.yml`:

```properties
server.forward-headers-strategy=framework
```

This tells Spring Boot to trust `X-Forwarded-Proto`, `X-Forwarded-Host`, etc. headers from the reverse proxy so it generates `https://` URLs.

### Option 2: Force HTTPS in Spring Security config

In your `SecurityConfig.java` (or `SecurityConfiguration.java`):

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        // ... existing config ...
        .requiresChannel(channel -> channel
            .anyRequest().requiresSecure()
        );
    // ... rest of config ...
    return http.build();
}
```

### Option 3: Ensure the reverse proxy (Nginx) sends forwarded headers

If using Nginx, make sure it forwards the protocol:

```nginx
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header X-Forwarded-Host  $host;
proxy_set_header X-Forwarded-Port  $server_port;
```

Combined with **Option 1** above.

## What's NOT broken on the frontend

- `config.js` correctly uses `https://spring.opencodingsociety.com`
- All fetch calls use `...fetchOptions` with `credentials: 'include'`
- No hardcoded `http://` URLs exist in the calendar or include files
- The calendar SCSS, holiday injection, priority colors, etc. are all fine

The fix is **entirely on the backend/infrastructure side**.
