Yep, you can definitely add more entries under the same `data:` block in your ConfigMap — just list them one after another.

Each key in `data:` is basically a filename (or config key), and the value is its content.

Here’s an example showing how to add your nginx config **alongside** your existing JSON file:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-configmap
data:
  mcpweb-env-settings.json: |
    {
      "ApiUrl": "https://example.com/api",
      "Environment": "prod"
    }

  nginx.conf: |
    worker_processes auto;
    events { worker_connections 1024; }
    http {
      include       /etc/nginx/mime.types;
      sendfile      on;
      keepalive_timeout 65;
      client_max_body_size 100m;

      upstream kestrel {
        server 127.0.0.1:5000;
        keepalive 32;
      }

      server {
        listen 80;
        location / {
          proxy_pass http://kestrel;
        }
      }
    }
```

Then, in your Deployment, mount the individual file(s) you need:

```yaml
volumeMounts:
  - name: config-volume
    mountPath: /app/settings/mcpweb-env-settings.json
    subPath: mcpweb-env-settings.json
  - name: nginx-conf
    mountPath: /etc/nginx/nginx.conf
    subPath: nginx.conf

volumes:
  - name: config-volume
    configMap:
      name: my-configmap
  - name: nginx-conf
    configMap:
      name: my-configmap
```

### ✅ A few pointers:

* Keep your indentation clean (2 spaces under `data:`).
* You can have **as many keys** as you want under `data:`.
* If you mount multiple keys from one ConfigMap, you can either:

  * Mount the **whole directory** (it’ll drop all files inside it), or
  * Mount each file with `subPath` (as shown above).

If you’d like, I can help you merge your actual `mcpweb-env-settings.json` and nginx config into one clean ConfigMap snippet that’ll mount properly in your Deployment. Want me to do that?
