# nginx-rate-limit-sliding-window
Nginx rate limit con algoritmo de Ventana Deslizante (Sliding Window).


1. Algoritmo de Ventana Deslizante

Utiliza limit_req con burst y nodelay para simular el comportamiento de ventana deslizante
El parámetro burst permite acumular requests que se procesan inmediatamente con nodelay
Esto crea un efecto similar a una ventana deslizante donde los requests se van "recargando" gradualmente

2. Headers personalizados en error 429

X-RateLimit-Remaining: Indica cuántas peticiones quedan disponibles (0 cuando se alcanza el límite)
X-RateLimit-Policy: Describe la política de rate limiting aplicada (ej: "100 requests per minute, sliding window")
Retry-After: Header adicional que indica cuándo reintentar (60 segundos)

3. Zonas de rate limiting diferenciadas

General: 300 requests/minuto para navegación normal
API: 100 requests/minuto para endpoints de API
Login: 5 requests/minuto para prevenir ataques de fuerza bruta

4. Manejo personalizado del error 429

Respuesta JSON estructurada con información del error
Headers consistentes usando always para garantizar su presencia
Información clara sobre la política aplicada

5. Configuración de Burst

General: burst=50 (permite ráfagas de hasta 50 requests)
API: burst=20 (ráfagas más pequeñas para APIs)
Login: burst=2 (muy restrictivo para seguridad)

Cómo funciona el sliding window:

Cada IP tiene un "bucket" que se llena a la velocidad definida (rate)
El burst define cuántos requests se pueden acumular
Con nodelay, los requests se procesan inmediatamente si hay espacio en el bucket
El bucket se va recargando continuamente a la velocidad especificada
Esto crea un efecto de ventana deslizante donde los límites se van renovando gradualmente

Para aplicar esta configuración:

Guarda el archivo como /etc/nginx/sites-available/cms
Crea un enlace simbólico: ln -s /etc/nginx/sites-available/cms /etc/nginx/sites-enabled/
Prueba la configuración: nginx -t
Recarga Nginx: systemctl reload nginx

Los headers X-RateLimit-Remaining y X-RateLimit-Policy aparecerán tanto en respuestas exitosas como en los errores 429, proporcionando transparencia completa sobre el estado del rate limiting.

```nginx
# Nginx configuration for PHP CMS with sliding window rate limiting

# Define rate limiting zones using sliding window algorithm
# Zone for API requests - 10MB can track ~160,000 IP addresses
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=100r/m;

# Zone for general requests - 10MB for tracking
limit_req_zone $binary_remote_addr zone=general_limit:10m rate=300r/m;

# Zone for login attempts - more restrictive
limit_req_zone $binary_remote_addr zone=login_limit:5m rate=5r/m;

# Map to calculate remaining requests (approximation)
map $limit_req_status $limit_remaining {
    default         100;
    PASSED          $upstream_http_x_ratelimit_remaining;
    REJECTED        0;
    REJECTED_DRY_RUN 0;
}

server {
    listen 80;
    server_name cms.example.com;
    root /var/www/cms/public;
    index index.php;

    # Custom error page for rate limiting
    error_page 429 = @rate_limit_exceeded;
    
    # Location for rate limit error handling
    location @rate_limit_exceeded {
        # Add rate limit headers
        add_header X-RateLimit-Remaining 0 always;
        add_header X-RateLimit-Policy "100 requests per minute, sliding window" always;
        add_header Retry-After 60 always;
        add_header Content-Type application/json always;
        
        # Return JSON error response
        return 429 '{"error": "Too Many Requests", "message": "Rate limit exceeded. Please try again later.", "retry_after": 60}';
    }

    # General site locations
    location / {
        # Apply general rate limiting with burst and nodelay for sliding window effect
        limit_req zone=general_limit burst=50 nodelay;
        limit_req_status 429;
        
        # Add rate limit headers on successful requests
        add_header X-RateLimit-Remaining $limit_req_remaining always;
        add_header X-RateLimit-Policy "300 requests per minute, sliding window" always;
        
        try_files $uri $uri/ /index.php?$query_string;
    }

    # API endpoints with stricter rate limiting
    location ~ ^/api/ {
        # Apply API rate limiting with smaller burst for sliding window
        limit_req zone=api_limit burst=20 nodelay;
        limit_req_status 429;
        
        # Add rate limit headers
        add_header X-RateLimit-Remaining $limit_req_remaining always;
        add_header X-RateLimit-Policy "100 requests per minute, sliding window" always;
        
        try_files $uri /index.php?$query_string;
    }

    # Login endpoint with very strict rate limiting
    location = /login {
        # Apply login rate limiting
        limit_req zone=login_limit burst=2 nodelay;
        limit_req_status 429;
        
        # Add rate limit headers
        add_header X-RateLimit-Remaining $limit_req_remaining always;
        add_header X-RateLimit-Policy "5 requests per minute, sliding window" always;
        
        try_files $uri /index.php?$query_string;
    }

    # PHP processing
    location ~ \.php$ {
        # Check if file exists
        try_files $uri =404;
        
        # FastCGI parameters
        fastcgi_pass unix:/var/run/php/php8.2-fpm.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
        
        # Add rate limit information to PHP
        fastcgi_param HTTP_X_RATELIMIT_REMAINING $limit_req_remaining;
        fastcgi_param HTTP_X_RATELIMIT_POLICY "varies by endpoint, sliding window";
    }

    # Static assets
    location ~* \.(jpg|jpeg|png|gif|ico|css|js|woff|woff2|ttf|svg|eot)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
        access_log off;
    }

    # Deny access to hidden files
    location ~ /\. {
        deny all;
        access_log off;
        log_not_found off;
    }

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
}

# HTTPS configuration
server {
    listen 443 ssl http2;
    server_name cms.example.com;
    root /var/www/cms/public;
    index index.php;

    # SSL configuration
    ssl_certificate /etc/nginx/ssl/cms.crt;
    ssl_certificate_key /etc/nginx/ssl/cms.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    # Custom error page for rate limiting
    error_page 429 = @rate_limit_exceeded;
    
    # Location for rate limit error handling
    location @rate_limit_exceeded {
        # Add rate limit headers
        add_header X-RateLimit-Remaining 0 always;
        add_header X-RateLimit-Policy "100 requests per minute, sliding window" always;
        add_header Retry-After 60 always;
        add_header Content-Type application/json always;
        
        # Return JSON error response
        return 429 '{"error": "Too Many Requests", "message": "Rate limit exceeded. Please try again later.", "retry_after": 60}';
    }

    # General site locations
    location / {
        # Apply general rate limiting with burst and nodelay for sliding window effect
        limit_req zone=general_limit burst=50 nodelay;
        limit_req_status 429;
        
        # Add rate limit headers on successful requests
        add_header X-RateLimit-Remaining $limit_req_remaining always;
        add_header X-RateLimit-Policy "300 requests per minute, sliding window" always;
        
        try_files $uri $uri/ /index.php?$query_string;
    }

    # API endpoints with stricter rate limiting
    location ~ ^/api/ {
        # Apply API rate limiting with smaller burst for sliding window
        limit_req zone=api_limit burst=20 nodelay;
        limit_req_status 429;
        
        # Add rate limit headers
        add_header X-RateLimit-Remaining $limit_req_remaining always;
        add_header X-RateLimit-Policy "100 requests per minute, sliding window" always;
        
        try_files $uri /index.php?$query_string;
    }

    # Login endpoint with very strict rate limiting
    location = /login {
        # Apply login rate limiting
        limit_req zone=login_limit burst=2 nodelay;
        limit_req_status 429;
        
        # Add rate limit headers
        add_header X-RateLimit-Remaining $limit_req_remaining always;
        add_header X-RateLimit-Policy "5 requests per minute, sliding window" always;
        
        try_files $uri /index.php?$query_string;
    }

    # PHP processing
    location ~ \.php$ {
        # Check if file exists
        try_files $uri =404;
        
        # FastCGI parameters
        fastcgi_pass unix:/var/run/php/php8.2-fpm.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
        
        # Add rate limit information to PHP
        fastcgi_param HTTP_X_RATELIMIT_REMAINING $limit_req_remaining;
        fastcgi_param HTTP_X_RATELIMIT_POLICY "varies by endpoint, sliding window";
    }

    # Static assets
    location ~* \.(jpg|jpeg|png|gif|ico|css|js|woff|woff2|ttf|svg|eot)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
        access_log off;
    }

    # Deny access to hidden files
    location ~ /\. {
        deny all;
        access_log off;
        log_not_found off;
    }

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
}

# Redirect HTTP to HTTPS
server {
    listen 80;
    server_name www.cms.example.com;
    return 301 https://cms.example.com$request_uri;
}
```
