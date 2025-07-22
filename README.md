# Cloudflare-Worker-Domain-Map
Cloudflare Worker 用于域名映射

```js
addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request));
});

async function handleRequest(request) {
  const DOMAIN_MAP = {
    'a.com': 'a.cn',
    'b.com': 'b.cn',
  };

  const EXCEPTIONS = [
    'et.a.com',
    'no.a.com',
    'et.b.com',
    'no.b.com',
  ];

  const url = new URL(request.url);
  const sourceHost = url.hostname;

  // 检查例外列表
  if (EXCEPTIONS.includes(sourceHost)) {
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
    redirect: 'follow'
  });
  modifiedRequest.headers.set('Host', targetHost);
  modifiedRequest.headers.delete('Referer');
  modifiedRequest.headers.set('Referer', `https://${targetHost}/`);

  try {
    const response = await fetch(modifiedRequest);
    const contentType = response.headers.get('Content-Type') || '';
    
    const textTypes = [
      'text/html',
      'text/css',
      'application/javascript',
      'application/json',
      'application/xml'
    ];
    const isText = textTypes.some(type => contentType.includes(type));

    if (!isText) {
      return new Response(response.body, {
        status: response.status,
        statusText: response.statusText,
        headers: response.headers
      });
    }

    let text = await response.text();
    const regex = new RegExp(
      `(https?:\\/\\/)(([a-zA-Z0-9_-]+\\.)*?)${escapeRegExp(targetDomain)}(:\\d+)?`,
      'gi'
    );
    text = text.replace(regex, `$1$2${rootDomain}$4`);
    
    const newHeaders = new Headers(response.headers);
    newHeaders.delete('Content-Length');

    return new Response(text, {
      status: response.status,
      statusText: response.statusText,
      headers: newHeaders
    });

  }
  catch (error) {
    return new Response('Proxy error: ' + error.message, { status: 502 });
  }
}

function escapeRegExp(string) {
  return string.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
}
```
