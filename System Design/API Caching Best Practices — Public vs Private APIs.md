
#System-design #caching #reverse-proxy #nginx
## **1. Public API Caching**

- **When to use:** Data is the same for all users and not sensitive.
    
- **Best Practices:**
    
    - **Use reverse proxy caching** (e.g., Nginx, Varnish, Cloudflare).
        
    - **Cache-Control headers:**

			Cache-Control: public, max-age=3600
    
        - `public`: Cache can be stored by any intermediary (CDN, browser, etc.).
            
        - `max-age`: How long (in seconds) the cache is valid.
            
    - **Etag/Last-Modified headers** for validation instead of full reload.
        
    - **Store in-memory or disk-based cache** for high-speed delivery.
        
    - **Leverage CDN** for global edge caching.
        

---

## **2. Private (Authenticated) API Caching**

- **When to use:** Per-user customized or sensitive data.
    
- **Challenges:**
    
    - Cache must not mix data between users.
        
    - Session or token-specific content requires separation.
        
- **Best Practices:**
    
    - **Avoid full-page caching** for private APIs — cache only specific components or computed results.
        
    - **Use key-based cache separation**:
        
        `proxy_cache_key "$scheme$request_method$host$request_uri$user_id";`
        
    - Store per-user cache **in Redis or Memcached** (not Nginx shared cache) for fast lookup and invalidation.
        
    - Short TTL (Time To Live) to reduce stale data risk.
        
    - Use `Cache-Control: private` to prevent intermediaries from caching.
        
    - For APIs with mixed public/private fields, split into **two endpoints**:
        
        - Public data → can be cached by reverse proxy.
            
        - Private data → cached per user at the application level.
            

---

## **3. Nginx Notes**

- Nginx caching is **disk-based by default**.
    
- For speed-critical cases, use:
    
    `proxy_cache_path /tmp/nginx_cache keys_zone=my_cache:10m inactive=60m;`
    
    - `keys_zone` is in-memory index (not actual data).
        
    - Data itself is stored on disk.
        
- Per-user caching requires including user identifiers in the `proxy_cache_key`.
    

---

## **4. Recommended Approach**

|API Type|Caching Location|Keying Strategy|TTL|
|---|---|---|---|
|Public|CDN / Nginx disk|URI-based|Long|
|Private|Redis / Memcache|URI + User ID / Session ID|Short|

---

## **5. Example — Mixed Approach**

```

# Public API caching
location /public/ {
    proxy_cache my_cache;
    proxy_cache_key "$scheme$request_method$host$request_uri";
    proxy_cache_valid 200 1h;
    add_header Cache-Control "public, max-age=3600";
}

# Private API caching
location /user/ {
    set $user_id $cookie_user_id; # Example from cookie
    proxy_cache my_private_cache;
    proxy_cache_key "$scheme$request_method$host$request_uri$user_id";
    proxy_cache_valid 200 10s;
    add_header Cache-Control "private, max-age=10";
}

```

