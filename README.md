# Cloudflare-Worker-Domain-Map
Cloudflare Worker 用于域名映射

```ts
addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request, event));
});

async function handleRequest(request, event) {
  const DOMAIN_MAP = {
    'a.com': 'a.cn',
    'b.com': 'b.cn',
  };

  const EXCLUDED_HOSTS = [
    'et.a.com',
    'no.a.com',
    'et.b.com',
    'no.b.com',
  ];

  const url = new URL(request.url);
  const sourceHost = url.hostname;

  if (EXCLUDED_HOSTS.includes(sourceHost)) {
    return fetch(request);
  }

  const rootDomain = Object.keys(DOMAIN_MAP).find(domain => 
    sourceHost === domain || sourceHost.endsWith(`.${domain}`)
  );
  if (!rootDomain) return new Response('Domain not mapped', { status: 404 });

  const targetDomain = DOMAIN_MAP[rootDomain];
  const targetHost = sourceHost.replace(rootDomain, targetDomain);
  const targetUrl = `https://${targetHost}${url.pathname}${url.search}`;

  const modifiedRequest = new Request(targetUrl, {
    headers: new Headers(request.headers),
    method: request.method,
    redirect: 'manual'
  });
  modifiedRequest.headers.set('Host', targetHost);
  modifiedRequest.headers.delete('Referer');
  modifiedRequest.headers.set('Referer', `https://${targetHost}/`);

  try {
    // -------- 方法二：先检查缓存 --------
    const cache = caches.default;
    let cachedResponse = await cache.match(request);
    if (cachedResponse) {
      return cachedResponse;
    }
    // -----------------------------------

    const response = await fetch(modifiedRequest);
    if (response.status >= 300 && response.status < 400) {
      let location = response.headers.get('Location');
      if (!location) {
        return new Response('Redirect without location', { status: 502 });
      }

      let redirectedUrl = new URL(location, targetUrl); // 相对跳转也能解析

      if (redirectedUrl.hostname.endsWith(targetDomain)) {

        const newHost = redirectedUrl.hostname.replace(targetDomain, rootDomain);
        redirectedUrl.hostname = newHost;

        return Response.redirect(redirectedUrl.toString(), 302);
      } else {
        return Response.redirect(redirectedUrl.toString(), 302);
      }
    }
    const contentType = response.headers.get('Content-Type') || '';
    
    // 白名单兜底
    const textTypes = [
      'text/html',
      'application/xhtml+xml',
      'text/css',
      'application/javascript',
      'application/x-javascript',
      'text/javascript',
      'application/json',
      'application/ld+json',
      'application/manifest+json',
      'application/xml',
      'text/xml',
      'application/rss+xml',
      'application/atom+xml',
      'application/xml+rss',
      'application/xml-dtd',
      'application/opensearchdescription+xml',
      'application/xslt+xml',
      'application/mathml+xml',
      'application/svg+xml',
      'text/plain',
      'text/markdown',
      'text/csv',
      'application/csv',
      'application/x-csv',
      'text/tab-separated-values'
    ];
    // 自动判断是否是文本类型
    const isText = (
      contentType.startsWith('text/') ||
      contentType.includes('+xml') ||
      contentType.includes('+json') ||
      textTypes.some(type => contentType.includes(type))
    );

    if (!isText) {
      // -------- 方法一：加缓存头 --------
      const newHeaders = new Headers(response.headers);
      newHeaders.set('Cache-Control', 'public, max-age=2592000');
      const newResp = new Response(response.body, {
        status: response.status,
        statusText: response.statusText,
        headers: newHeaders
      });
      // -------- 方法二：写入缓存 --------
      event.waitUntil(cache.put(request, newResp.clone()));
      return newResp;
    }

    let text = await response.text();
    const urlRegex = new RegExp(
      `(https?:\\/\\/)(([a-zA-Z0-9_-]+\\.)*?)${escapeRegExp(targetDomain)}(:\\d+)?`,
      'gi'
    );
    text = text.replace(urlRegex, `$1$2${rootDomain}$4`);
    const domainRegex = new RegExp(escapeRegExp(targetDomain), 'gi');
    text = text.replace(domainRegex, rootDomain);

    const newHeaders = new Headers(response.headers);
    newHeaders.delete('Content-Length');
    // -------- 方法一：加缓存头 --------
    newHeaders.set('Cache-Control', 'public, max-age=2592000');

    const newResp = new Response(text, {
      status: response.status,
      statusText: response.statusText,
      headers: newHeaders
    });

    // -------- 方法二：写入缓存 --------
    event.waitUntil(cache.put(request, newResp.clone()));

    return newResp;

  }
  catch (error) {
    return new Response('Proxy error: ' + error.message, { status: 502 });
  }
}

function escapeRegExp(string) {
  return string.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
}
```
