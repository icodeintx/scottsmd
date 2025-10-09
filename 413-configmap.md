Nice—since your API pod is Linux + **nginx** in front of Kestrel, the 413 is almost certainly nginx’s **`client_max_body_size`** biting you (default is 1 MB). You’ll want to raise limits in **both places**:

* nginx (reverse proxy inside the container)
* ASP.NET Core (Kestrel + FormOptions)

Here’s a clean, drop-in setup.

# 1) nginx inside the API pod

Bump body size and make uploads more resilient. You can do this either in `http {}` (global) or the `server {}`/`location {}` block.

**nginx.conf (minimal reverse proxy to Kestrel)**

```nginx
worker_processes auto;

events { worker_connections 1024; }

http {
    include       /etc/nginx/mime.types;
    sendfile      on;
    keepalive_timeout  65;

    # >>> raise upload limit (e.g., 200 MB) <<<
    client_max_body_size 200m;

    # Optional: helpful for big/slow uploads
    client_body_buffer_size 8m;
    proxy_read_timeout 600s;
    proxy_send_timeout 600s;
    proxy_connect_timeout 60s;

    # If you stream uploads directly to Kestrel to avoid temp files:
    # proxy_request_buffering off;

    upstream kestrel {
        server 127.0.0.1:5000;
        keepalive 32;
    }

    server {
        listen 80;
        server_name _;

        location / {
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            proxy_pass http://kestrel;

            # Mirror the limits here if you prefer per-location:
            # client_max_body_size 200m;
        }
    }
}
```

## Helm way (ConfigMap + mount)

If your chart doesn’t already template `nginx.conf`, do this:

**values.yaml (snippet)**

```yaml
nginx:
  clientMaxBodySize: 200m
  readTimeout: 600s
  requestBuffering: "off"   # optional
```

**templates/configmap-nginx.yaml**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "api.fullname" . }}-nginx
data:
  nginx.conf: |
    worker_processes auto;
    events { worker_connections 1024; }
    http {
      include /etc/nginx/mime.types;
      sendfile on;
      keepalive_timeout 65;
      client_max_body_size {{ .Values.nginx.clientMaxBodySize | default "200m" }};
      proxy_read_timeout {{ .Values.nginx.readTimeout | default "600s" }};
      proxy_send_timeout {{ .Values.nginx.readTimeout | default "600s" }};
      proxy_connect_timeout 60s;
      {{- if eq (lower .Values.nginx.requestBuffering | default "on") "off" }}
      proxy_request_buffering off;
      {{- end }}

      upstream kestrel { server 127.0.0.1:5000; keepalive 32; }
      server {
        listen 80;
        location / {
          proxy_set_header Host $host;
          proxy_set_header X-Real-IP $remote_addr;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_set_header X-Forwarded-Proto $scheme;
          proxy_pass http://kestrel;
        }
      }
    }
```

**templates/deployment.yaml (mount it)**

```yaml
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
          volumeMounts:
            - name: nginx-conf
              mountPath: /etc/nginx/nginx.conf
              subPath: nginx.conf
        - name: api
          image: your-registry/your-api:{{ .Chart.AppVersion }}
          env:
            - name: ASPNETCORE_URLS
              value: "http://0.0.0.0:5000"
        volumes:
          - name: nginx-conf
            configMap:
              name: {{ include "api.fullname" . }}-nginx
```

> If your container image already bakes an `nginx.conf`, just make sure `client_max_body_size` is set there (no Helm changes required).

---

# 2) ASP.NET Core limits (Kestrel + multipart)

nginx accepting the big body isn’t enough—Kestrel and the form parser must allow it.

**Program.cs**

```csharp
using Microsoft.AspNetCore.Http.Features;

var builder = WebApplication.CreateBuilder(args);

// Kestrel: allow 200 MB
builder.WebHost.ConfigureKestrel(o =>
{
    o.Limits.MaxRequestBodySize = 200 * 1024 * 1024;
});

// Multipart/form-data: allow 200 MB
builder.Services.Configure<FormOptions>(o =>
{
    o.MultipartBodyLengthLimit = 200L * 1024 * 1024;
});

var app = builder.Build();
app.MapControllers();
app.Run();
```

Or wire from config (nice for Helm/env):

**appsettings.json (.NET 8+)**

```json
{
  "Kestrel": { "Limits": { "MaxRequestBodySize": 209715200 } },
  "FormOptions": { "MultipartBodyLengthLimit": 209715200 }
}
```

**Program.cs**

```csharp
builder.Services.Configure<FormOptions>(builder.Configuration.GetSection("FormOptions"));
```

**Helm env overrides**

```yaml
env:
  - name: ASPNETCORE_Kestrel__Limits__MaxRequestBodySize
    value: "209715200"
  - name: FormOptions__MultipartBodyLengthLimit
    value: "209715200"
```

---

# 3) Timeouts & buffering notes

* Long uploads? bump upstream/read timeouts in nginx (done above).
* If disk I/O is a concern, consider `proxy_request_buffering off` to stream to Kestrel (less temp-file usage in nginx).
* If you do turn buffering off, make sure your API reads the stream efficiently and your readiness/liveness probes aren’t too aggressive.

---

# 4) Double-check the outer path

You mentioned the 413’s from backend, but if you also run **Ingress NGINX** or **App Gateway** in front, ensure *their* limits aren’t smaller than nginx-in-pod:

* Ingress NGINX annotation: `nginx.ingress.kubernetes.io/proxy-body-size: "200m"`

---

If you tell me your **target max size** (say 500 MB) and whether you’re using a **sidecar nginx** or **single container with nginx**, I’ll size the exact numbers and hand you a ready-to-paste Helm values + config that matches your chart structure.
