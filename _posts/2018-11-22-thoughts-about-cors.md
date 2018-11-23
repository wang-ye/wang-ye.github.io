I came across a [CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS) error and spent long time debugging it. This article shares my learnings during the debugging process.

## Why CORS?

The same origin policy imposes constraints to API calls in different domains. However, there are situations we still want to call external APIs. To tackle this problem, CORS is designed to soften the restriction without introducing extra loopholes.

## A Common CORS Error
Sometimes, you will see errors like "Access to fetch at WEBSITE from origin YOUR_ORIGIN has been blocked by CORS policy: No 'Access-Control-Allow-Origin' header is present on the requested resource.". Clearly this is a CORS error. What is more interesting is, although you can still make the request and even get the response, you cannot really use it!

How can we fix this issue? Simple case first. If you have full control to the remote server, you can set in response the Access-Control-Allow-Origin header value to '*', and the CORS issue will be resolved.

However, if you  you do not have control on remote server, what can you do?

## Solving CORS Error With Proxy
There is a generic approach for solving CORS errors. [CORS Everywhere](https://github.com/Rob--W/cors-anywhere/) provides a proxy-based solution. The high level idea is adding the necessary CORS headers in the response, so the client calling external APIs can parse the content correctly. Here I took the code snippet to add the CORS headers.

```js
function withCORS(headers, request) {
  headers['access-control-allow-origin'] = '*';
  var corsMaxAge = request.corsAnywhereRequestState.corsMaxAge;
  if (corsMaxAge) {
    headers['access-control-max-age'] = corsMaxAge;
  }
  if (request.headers['access-control-request-method']) {
    headers['access-control-allow-methods'] = request.headers['access-control-request-method'];
    delete request.headers['access-control-request-method'];
  }
  if (request.headers['access-control-request-headers']) {
    headers['access-control-allow-headers'] = request.headers['access-control-request-headers'];
    delete request.headers['access-control-request-headers'];
  }

  headers['access-control-expose-headers'] = Object.keys(headers).join(',');

  return headers;
}
```
