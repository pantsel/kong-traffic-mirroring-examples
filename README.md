# Mirroring Traffic to a Remote Server with Kong

## Overview
This document provides two methods to mirror incoming traffic to a remote server when using Kong Gateway. Traffic mirroring is particularly useful for debugging, monitoring, or testing purposes.

### Methods:
1. **Using the built-in `ngx_http_mirror_module`**
2. **Using a custom plugin**

---

## Method 1: Using the `ngx_http_mirror_module`
The [`ngx_http_mirror_module`](https://nginx.org/en/docs/http/ngx_http_mirror_module.html) is a standard module included in the Kong Nginx package. It enables mirroring of incoming requests to a remote server.

To use this module, you will need to configure the Nginx directives within the Kong configuration.

### Step 1: Create the mirror server configuration
Create a file named `mirror-inject.conf` with the following content:

```nginx
location = /mirror {
    internal;

    # Set the upstream to the mirror server
    proxy_pass https://my-mirror-server.com;

    # Rewrite the path and pass to the mirror backend
    rewrite ^/mirror(.*)$ $request_uri$1 break;

    # Pass SSL/TLS information (Optional)
    proxy_set_header X-SSL-Protocol $ssl_protocol;
    proxy_set_header X-SSL-Cipher $ssl_cipher;
    proxy_set_header X-SSL-Server-Name $ssl_server_name;
    proxy_set_header X-SSL-Session-ID $ssl_session_id;

    # Pass client certificate information (if mutual TLS is enabled)
    proxy_set_header X-SSL-Client-Cert $ssl_client_cert;
    proxy_set_header X-SSL-Client-S-DN $ssl_client_s_dn;
    proxy_set_header X-SSL-Client-Fingerprint $ssl_client_fingerprint;
    proxy_set_header X-SSL-Client-Serial $ssl_client_serial;

    # Pass original request headers, query strings, and body
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Request-ID $request_id;

    # Pass the request as-is
    proxy_pass_request_body on;
    proxy_pass_request_headers on;
}
```

> **Note:** Ensure this file is placed in the `/usr/local/kong/mirror-inject.conf` directory on the Kong server.

### Step 2: Start Kong with the mirroring configuration
Update the Kong configuration to include the mirroring file.

**Using `kong.conf`:**
```bash
nginx_proxy_include="mirror-inject.conf"
nginx_location_mirror="/mirror"
nginx_location_mirror_request_body="on"
```

**Using environment variables:**
```bash
KONG_NGINX_PROXY_INCLUDE=mirror-inject.conf
KONG_NGINX_LOCATION_MIRROR=/mirror
KONG_NGINX_LOCATION_MIRROR_REQUEST_BODY=on
```

### Run the example environment
To test the configuration, run the example environment using Docker Compose:

```bash
docker-compose up -d
```

This will:
- Spin up a DB-less Kong Gateway with a sample service and route (`./kong.yaml`).
- Launch an echo server to receive mirrored requests.
- Mount the `mirror-inject.conf` file into the Kong container.

> **Refer to the `docker-compose.yaml` file for details.**

#### Test traffic mirroring
Check the echo server logs to view mirrored requests:
```bash
docker-compose logs -f echo
```

Issue a sample request to the Kong Gateway:

```bash
curl -i -X POST http://localhost:8000/custom-path?param1=value1&param2=value2 \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer mytoken" \
    -d '{
          "key1": "value1",
          "key2": "value2"
        }'
```

You should see the mirrored request in the echo server logs, including the original headers, query strings, and request body.

---

## Method 2: Using a Custom Plugin
Kong does not provide a built-in plugin for traffic mirroring, but you can use community plugins like [kong-plugin-shadow](https://github.com/Tieske/kong-plugin-shadow) to achieve this functionality.

> **Note:** `kong-plugin-shadow` is a community plugin and not officially supported by Kong. For issues or feature requests, refer to the plugin's GitHub repository.

### Step 1: Install the `shadow` plugin
Follow the installation instructions provided in the plugin's [GitHub repository](https://github.com/Tieske/kong-plugin-shadow).

### Step 2: Configure the `shadow` plugin globally
Create a configuration file (e.g., `kong-plugin-shadow.yaml`) and define the plugin settings:

```yaml
plugins:
  - name: shadow
    config:
      weight: 100
      shadow_request_body: true
      shadow_endpoints:
        # Specify one or more endpoints to mirror the traffic
        - http://my-mirror-server.com
```

### Step 3: Start Kong with the `shadow` plugin enabled

**Using `kong.conf`:**
```bash
plugins=bundled,shadow
```

**Using environment variables:**
```bash
KONG_PLUGINS=bundled,shadow
```

### Run the example environment
Clone the `kong-plugin-shadow` repository:

```bash
git clone https://github.com/Tieske/kong-plugin-shadow
```

Start the environment with Docker Compose:
```bash
docker-compose -f docker-compose.shadow.yaml up -d
```

This will:
- Launch a DB-less Kong Gateway preconfigured with the `shadow` plugin (`./kong-plugin-shadow.yaml`).
- Start an echo server to receive mirrored requests.

> **Refer to the `docker-compose.shadow.yaml` file for details.**

#### Test traffic mirroring
Check the echo server logs to view mirrored requests:
```bash
docker-compose logs -f echo
```

Issue a sample request to the Kong Gateway:

```bash
curl -i -X POST http://localhost:8000/custom-path?param1=value1&param2=value2 \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer mytoken" \
    -d '{
          "key1": "value1",
          "key2": "value2"
        }'
```

You should see the mirrored request in the echo server logs, including the original headers, query strings, and request body.

---

## Conclusion
Both methods allow you to mirror traffic to a remote server for monitoring and debugging purposes:
1. Use the `ngx_http_mirror_module` for a native Nginx-based approach.
2. Use the `kong-plugin-shadow` plugin for a custom plugin-based solution.

Choose the method that best suits your use case and environment.
