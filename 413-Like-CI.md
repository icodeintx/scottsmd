Yep—that Ingress annotation fixes it **only if** the *smallest* limit on the path is the Ingress. In your case the 413 is coming from the backend pod (nginx → Kestrel), so you need the **Ingress** limit **and** the **pod’s nginx/Kestrel limits** higher than (or equal to) 100 MB.

Here’s the “make it all agree” setup using **100 MB** to match your old app:

## 1) Ingress (you already have this)

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "100m"
```

## 2) Pod’s nginx (reverse proxy inside your API container/pod)

Set `client_max_body_size 100m;` (global `http {}` or inside your `server`/`location`).

**nginx.conf**

```nginx
http {
  client_max_body_size 100m;
  proxy_read_timeout 600s;
  proxy_send_timeout 600s;
  # proxy_request_buffering off; # (optional) stream directly to Kestrel
  ...
}
```

(If you manage this via Helm, put it in a ConfigMap and mount over `/etc/nginx/nginx.conf`.)

## 3) ASP.NET Core (Kestrel + multipart/form)

Kestrel default is ~30 MB; multipart default ~128 MB. Bump both:

**Program.cs**

```csharp
using Microsoft.AspNetCore.Http.Features;

var builder = WebApplication.CreateBuilder(args);

builder.WebHost.ConfigureKestrel(o =>
{
    o.Limits.MaxRequestBodySize = 100 * 1024 * 1024; // 100 MB
});

builder.Services.Configure<FormOptions>(o =>
{
    o.MultipartBodyLengthLimit = 100L * 1024 * 1024; // 100 MB
});

var app = builder.Build();
app.MapControllers();
app.Run();
```

**(Optional config/env instead of code)**
*appsettings.json (.NET 8+)*:

```json
{
  "Kestrel": { "Limits": { "MaxRequestBodySize": 104857600 } },
  "FormOptions": { "MultipartBodyLengthLimit": 104857600 }
}
```

*Program.cs*:

```csharp
builder.Services.Configure<FormOptions>(builder.Configuration.GetSection("FormOptions"));
```

*Helm env*:

```yaml
env:
  - name: ASPNETCORE_Kestrel__Limits__MaxRequestBodySize
    value: "104857600"
  - name: FormOptions__MultipartBodyLengthLimit
    value: "104857600"
```

## 4) Quick verify checklist

* Hit the upload and check the response header `server:`. If it says `nginx` from your pod, that’s the pod’s nginx.
* Check nginx error log for the classic line:
  `client intended to send too large body`
* Make sure any **outer** proxy (Ingress NGINX or App Gateway) isn’t set **lower** than 100m.

If you want, tell me how you’re packaging nginx (sidecar vs baked into the same image), and I’ll drop in exact Helm `values.yaml` + `ConfigMap` and `Deployment` snippets wired to your chart style.
