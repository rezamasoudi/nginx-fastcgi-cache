# Nginx FastCGI Cache

This repository provides a sample configuration for enabling **FastCGI caching** in **Nginx** to improve application performance by caching PHP responses.

---

## Nginx Configuration

```nginxconf
fastcgi_cache_path /var/cache/nginx/my_app levels=1:2 keys_zone=my_app:100m inactive=60m;

fastcgi_cache_key "$request_method$host$request_uri";

# Define cache exclusion rules
map $request_uri $cache_bypass {
    default 1;
    "~^/orders/[0-9]$" 0;
}

server {

    ...

    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php8.3-fpm.sock;
        fastcgi_index index.php;
        include fastcgi_params;

        # Caching directives
        fastcgi_cache my_app;                             # Reference to cache zone
        fastcgi_cache_valid 200 60m;                      # Cache 200 responses for 60 minutes
        fastcgi_cache_methods GET;                        # Only cache GET requests
        add_header X-Cache $upstream_cache_status;        # Add cache hit/miss header
        fastcgi_cache_bypass $cache_bypass;               # Skip cache for certain URIs
        fastcgi_no_cache $cache_bypass;                   # Avoid storing those responses in cache
    }

    ...

}
```

## Purging Nginx Cache from Application (PHP)
If needed, you can manually remove a specific cache file using the following PHP function:

```php
function removeNginxCache(string $cache_dir, string $cache_level, string $key){

    $hash = md5($key);
    $dir = $cache_dir;
    $levels = explode(':', $cache_level);

    $easrsed_hash = $hash;
    foreach ($levels as $level) {
        $level_dir = substr($easrsed_hash, -$level);
        $easrsed_hash = substr($easrsed_hash, 0, -$level);
        $dir = $dir . DIRECTORY_SEPARATOR . $level_dir;
    }

    $cache_file = $dir . DIRECTORY_SEPARATOR . $hash;
    if (file_exists($cache_file))
        unlink($cache_file);
}

```

This function:
- Hashes the cache key (same logic used by Nginx internally)
- Navigates through the nested directory structure defined by levels
- Removes the corresponding cache file if it exists

### Example usage

```php

removeNginxCache('/var/cache/nginx/my_app', '1:2', 'GETlocalhost/orders/1');

```