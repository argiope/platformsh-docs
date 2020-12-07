---
title: "HTTPS"
weight: 1
---

## Let's Encrypt

All environments on Platform.sh support both HTTP and HTTPS automatically.  Production TLS certificates are provided by [Let's Encrypt](https://letsencrypt.org/).  You may alternatively provide your own TLS certificate from a 3rd party issuer of your choice at no charge from us.

Let’s Encrypt TLS Certificates are valid for 90 days and Platform.sh will automatically renew them 28 days before expiration to avoid HTTPS interruptions.  If a renewal is available and needed, the environment will automatically redeploy to renew the certificate.  As no new build is required the process should take at most a few seconds.

If you are using a custom TLS certificate, seven days before it expires Platform.sh will issue a Let's Encrypt certificate and replace the custom certificate with it in order to no interruption in service.  If you wish to continue using the custom certificate, replace it with an updated certificate more than seven days before it expires.

{{< note >}}
TLS certificates are often still called SSL certificates.  TLS is a newer encryption system that has replaced SSL, but the name SSL is still widely recognized.  In practice, they mean the same thing today, but TLS is the more correct term.
{{</ note >}}

## Using HTTPS

Platform.sh recommends using HTTPS requests for all sites exclusively.  Doing so provides better security, access to certain features that web browsers only permit over HTTPS, and access to HTTP/2 connections on all sites which can greatly improve performance.

How HTTPS redirection is handled depends on the routes you have defined.  Platform.sh recommends specifying all HTTPS routes in your `routes.yaml` file.  That will result in all pages being served over SSL, and any requests for an HTTP URL will automatically be redirected to HTTPS.

```yaml
"https://{default}/":
    type: upstream
    upstream: "app:http"

"https://www.{default}/":
    type: redirect
    to: "https://{default}/"
```

Specifying only HTTP routes will result in duplicate HTTPS routes being created automatically, allowing the site to be served from both HTTP and HTTPS without redirects.

Although Platform.sh does not recommend it, you can also redirect HTTPS requests to HTTP explicitly to serve the site over HTTP only.  The use cases for this configuration are few.

```yaml
"http://{default}/":
    type: upstream
    upstream: "app:http"

"http://www.{default}/":
    type: redirect
    to: "http://{default}/"

"https://{default}/":
    type: redirect
    to: "http://{default}/"

"https://www.{default}/":
    type: redirect
    to: "http://{default}/"
```

Of course, more complex routing logic is possible if the situation calls for it. However, we recommend defining HTTPS routes exclusively.

{{< note >}}
Let's Encrypt has a limit of 100 TLS certificates per environment.  If you define both a `{default}` and `www.{default}` route for each domain you use, that will give you a limit of 50 domains.  Adding more than that will result in a warning on deploy and some domains will not be issued a TLS certificate.  If you need more than that, we recommend obtaining additional certificates or a wildcard certificate from another TLS provider.  Alternatively, consider splitting your project up into multiple discrete Platform.sh projects.
{{< /note >}}

## TLS configuration

Optionally, it's possible to further refine how secure TLS connections are handled on your cluster via the `tls` route property.

```yaml
https://{default}/:
    type: upstream
    upstream: app:http
    tls:
        # ...
```

### `min_version`

Setting a minimum version of TLS will cause the server to automatically reject any connections using an older version of TLS.  Rejecting older versions with known security vulnerabilities is necessary for some security compliance processes.

```yaml
tls:
    min_version: TLSv1.3
```

The above configuration will result in requests using older TLS versions to be rejected. Legal values are `TLSv1.2` and `TLSv1.3`.
TLS versions older than 1.2 are not supported by Platform.sh and will be rejected regardless of the setting here.

Note that if multiple routes for the same domain have different `min_version`s specified, the highest specified will be used for the whole domain.

### `strict_transport_security`

HTTP Strict Transport Security (HSTS) is a mechanism for telling browsers to use HTTPS exclusively with a particular website.  You can toggle it on for your site at the router level without having to touch your application, and configure it's behavior from `routes.yaml`.

```yaml
tls:
    strict_transport_security:
        enabled: true
        include_subdomains: true
        preload: true
```

There are three sub-properties for the `strict_transport_security` property:

* `enabled`: Can be `true`, `false`, or `null`.  Defaults to `null`.  If `false`, the other properties wil be ignored.
* `include_subdomains`: Can be `true` or `false`.  Defaults to `false`. If `true`, browsers will be instructed to apply HSTS restrictions to all subdomains as well.
* `preload`: Can be `true` or `false`.  Defaults to `false`.  If `true`, Google and others may add your site to a lookup reference of sites that should only ever be connected to over HTTPS.  Many although not all browsers will consult this list before connecting to a site over HTTP and switch to HTTPS if instructed.  Although not part of the HSTS specification it is supported by most browsers.

If enabled, the `Strict-Transport-Security` header will always be sent with a lifetime of 1 year.  The [Mozilla Developer Network](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Strict-Transport-Security) has more detailed information on HSTS.

Note: If multiple routes for the same domain specify different HSTS settings, the entire domain will still use a shared configuration.  Specifically, if any route on the domain has `strict_transport_security.enabled` set to `false`, HSTS will be disabled for the whole domain.  Otherwise, it will be enabled for the whole domain if at least one such route has `enabled` set to `true`.  As this logic may be tricky to configure correctly we strongly recommend picking a single configuration for the whole domain and adding it on only a single route.

### Client authenticated TLS

In some non-browser applications (such as mobile applications, IoT devices, or other restricted-client-list use cases), it is beneficial to restrict access to selected devices using TLS.  This process is known as client-authenticated TLS, and functions effectively as a more secure alternative to HTTP Basic Auth.

By default, any valid SSL cert issued by one of the common certificate issuing authorities will be accepted.  Alternatively, you can restrict access to SSL certs issued by just those certificate authorities you specify, including a custom authority.  (The latter is generally only applicable if you are building a mass-market IoT device or similar.)  To do so, set `client_authentication` required and then provide a list of the certificates of the certificate authorities you wish to allow.

```yaml
tls:
    client_authentication: "require"
    client_certificate_authorities:
        - !include
            type: string
            path: root-ca1.crt
        - !include
            type: string
            path: root-ca2.crt
```

In this case, the certificate files are resolved relative to the `.platform` directory.  Alternatively, the certificates can be specified inline in the file:

```yaml
tls:
    client_authentication: "require"
    client_certificate_authorities:
        - |
            -----BEGIN CERTIFICATE-----
            ### Several lines of random characters here ###
            -----END CERTIFICATE-----
        - |
            -----BEGIN CERTIFICATE-----
            ### Several lines of different random characters here ###
            -----END CERTIFICATE-----
```

## Let's Encrypt limits and branch names

You may encounter Let's Encrypt certificates failing to provision after the build hook has completed:

```bash
Provisioning certificates
  Validating 2 new domains
  E: Error provisioning the new certificate, will retry in the background.
  (Next refresh will be at 2020-02-13 14:29:22.860563+00:00.)
  Environment certificates
W: Missing certificate for domain www.<PLATFORM_ENVIRONMENT>-<PLATFORM_PROJECT>.<REGION>.platformsh.site
W: Missing certificate for domain <PLATFORM_ENVIRONMENT>-<PLATFORM_PROJECT>.<REGION>.platformsh.site
```

One reason that this can happen has to do with the limits of Let's Encrypt itself, which caps off at 64 characters for URLs. If your TLS certificates are not being provisioned, it's possible that the names of your branches are too long, and the environment's generated URL goes over that limit.

At this time, generated URLs have the following pattern:

```
<PLATFORM_ENVIRONMENT>-<PLATFORM_PROJECT>.<REGION>.platformsh.site
```

* `PLATFORM_ENVIRONMENT` = `PLATFORM_BRANCH` + 7 character hash
* `PLATFORM_PROJECT` = 13 characters
* `REGION` = 2-4 characters, depending on the region
* `platformsh.site` = 15 characters
* extra characters (`.` & `-`) = 4 characters

This breakdown leaves you with 21-23 characters to work with naming your branches (`PLATFORM_BRANCH`) without going over the 64 character limit, dependent on the region. Since this pattern for generated URLs will remain similar, but could change slightly over time, it's our recommendation to use branch names with a maximum length between 15 and 20 characters.
