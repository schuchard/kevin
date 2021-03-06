---
templateKey: blog-post
title: 3 ways to use Angular HTTP Interceptors
date: 2019-05-08T00:00:00.000Z
description: HTTP Interceptors provide a flexible mechanism to control your application when dealing with network-related resources. They’re similar to middleware in other frameworks and allow network logic to be abstracted and reused.
tags:
  - angular
  - http
---

## Resources

- [🚀 Demo app](https://ng-interceptors.netlify.com)
- [🛠 Github](https://github.com/schuchard/ng-interceptors)

## Why use HTTP Interceptors

Interceptors provide a convenient location to apply functionality across all or some network requests and responses. As an application grows, re-implementing the same logic can become tedious, error-prone, and potentially leads to inconsistent functionality. For example, setting up an Authorization Header across multiple network requests can quickly lead to duplicate code at the service or component level. By utilizing an interceptor it can be configured once and applied on all existing and future HTTP requests. By abstracting global network logic to a single responsibility class, we make it easier to test and build reliable applications quickly.

## Intercepting Requests

Requests can be controlled with the `HttpHandler` that is passed to interceptor methods. In it's simplest usage, if you don't want to modify the request, you return the `handle` method. Typically used like this:

```ts
intercept(req: HttpRequest<any>, next: HttpHandler) {
  return next.handle(req)
}
```

This is where you can modify [Auth headers](https://angular.io/guide/http#set-default-headers) or anything else related to the request.

```ts
intercept(req: HttpRequest<any>, next: HttpHandler) {
  // Get the auth token from the service.
  const authToken = this.auth.getAuthorizationToken();

  // Clone the request and replace the original headers with
  // cloned headers, updated with the authorization.
  const authReq = req.clone({
    headers: req.headers.set('Authorization', authToken)
  });

  // send cloned request with header to the next handler.
  return next.handle(authReq);
}
```

## Intercepting Responses

To interact with the response, you'll `pipe` off the `handle` method. From there you can interact with other services such as adding to a cache like you'll see below or modifying the response in the XML to JSON example.

Be aware that there may be other events besides the response so it's a good idea to check for an `HttpResponse` before acting on the response event.

```ts
return next.handle(req).pipe(
  // There may be other events besides the response.
  filter(event => event instanceof HttpResponse),
  tap((event: HttpResponse<any>) => {
    cache.set(req.urlWithParams, {
      key: req.urlWithParams,
      body: event.body,
      dateAdded: Date.now(),
    });
  })
);
```

## Examples

### Caching

Caching, specifically [cache invalidation](https://martinfowler.com/bliki/TwoHardThings.html), can be quite difficult so this article will skip the specifics on that. Instead, we'll look at a simple example that returns something from the cache or allows the request to go through. For a given endpoint you can cache the results in a number of ways and return the cache based on time, existence, or another factor.

The following example checks for the existence in the cache and returns it:

```ts
export class CachingInterceptor implements HttpInterceptor {
  constructor(private cache: CacheService) {}

  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    // continue request if not cacheable.
    if (!this.canCache(req)) {
      return next.handle(req);
    }

    const cachedResponse = this.cache.get(req.urlWithParams);
    // return HttpResponse so that HttpClient methods return a value
    return cachedResponse
      ? of(new HttpResponse({ body: cachedResponse.body }))
      : sendRequest(req, next, this.cache);
  }

  canCache(req: HttpRequest<any>): boolean {
    // only cache `todo` routes
    return req.url.includes('todos');
  }
}
```

If the request is not in the cache, make the request as normal with `sendRequest`.

```ts
function sendRequest(
  req: HttpRequest<any>,
  next: HttpHandler,
  cache: CacheService
): Observable<HttpEvent<any>> {
  return next.handle(req).pipe(
    // There may be other events besides the response.
    filter(event => event instanceof HttpResponse),
    tap((event: HttpResponse<any>) => {
      cache.set(req.urlWithParams, {
        key: req.urlWithParams,
        body: event.body,
        dateAdded: Date.now(),
      });
    })
  );
}
```

Here we `pipe` off the `handle` method to interact with the response and set it in the cache. Now if the same request is made it will be returned immediately.

```ts
const cachedResponse = this.cache.get(req.urlWithParams);

return cachedResponse
  ? of(new HttpResponse({ body: cachedResponse.body }))
  : sendRequest(req, next, this.cache);
 ```

### XML to JSON

Often developers don’t have complete control over how they get data. For example, if you have one API that returns XML but the rest of your app works with JSON it might make sense to convert the XML response to JSON for consistency. With an interceptor, the XML conditional logic can be abstracted from consumers and applied to all network requests.

```ts
export class XmlInterceptor implements HttpInterceptor {
  constructor(@Inject(XmlParser) private xml: XMLParser) {}

  intercept(req: HttpRequest<any>, next: HttpHandler) {
    // extend server response observable with logging
    return next.handle(req).pipe(
      // proceed when there is a response; ignore other events
      filter(event => event instanceof HttpResponse),
      map(
        (event: HttpResponse<any>) => {
          if (this.xml.validate(event.body) !== true) {
            // only parse xml response, pass all other responses to other interceptors
            return event;
          }

          // {responseType: text} expects a string response
          return event.clone({ body: JSON.stringify(this.xml.parse(event.body)) });
        },
        // Operation failed; error is an HttpErrorResponse
        error => event
      )
    );
  }
}
```

In order to handle the response, this logic is `pipe`'ed off of the `handle` method. Then if the response is valid XML, we return the cloned response `event` and parse the XML into JSON.

### Redirect based on scopes

When an application needs to restrict access to certain routes, an interceptor can provide that functionality in one place across many routes. Since interceptors are run for every request you can organize your route guard logic in one place. Assuming you’ve set up what routes are guarded and can access the user's available scopes, the interceptor can determine whether to allow the request or redirect.

```ts
export class ScopesInterceptor implements HttpInterceptor {
  constructor(private scopesService: ScopesService, private router: Router) {}

  intercept(req: HttpRequest<any>, next: HttpHandler) {
    // not protected, pass request through
    if (!this.scopesService.protectedRoutes(req.urlWithParams)) {
      return next.handle(req);
    }

    // route is protected, only allow admins
    if (this.scopesService.isAdmin) {
      return next.handle(req);
    } else {
      // not admin, redirect and cancel request
      this.router.navigate(['404']);
      return of(undefined);
    }
  }
}
```

In this example, if the route isn't protected we don't take any action. If it is the request is allowed if the user is an admin otherwise we redirect and cancel the request.

## Immutability

It may not be obvious but the `HttpRequest` and `HttpResponse` instances are immutable. From the [Angular docs](https://angular.io/guide/http#immutability)
> interceptors are capable of mutating requests and responses, the HttpRequest and HttpResponse instance properties are read-only, rendering them largely immutable.

This is done because the app may retry a request several times and if the interceptor could modify the original request then each subsequent request could be different.

What this means for developers is that you must `clone` the request and modify that instance:

```ts
// clone request and replace 'http://' with 'https://' at the same time
const secureReq = req.clone({
  url: req.url.replace('http://', 'https://')
});
// send the cloned, "secure" request to the next handler.
return next.handle(secureReq);
```

## Interceptor order

Interceptors pass requests through in the order that they're [provided](https://angular.io/guide/http#provide-the-interceptor), `[A,B,C]`. The following "barrel" example gathers interceptors to later be provided in their declared order.

> requests will flow in A->B->C and responses will flow out C->B->A.

```ts
export const httpInterceptorProviders = [
  { provide: HTTP_INTERCEPTORS, useClass: NoopInterceptor, multi: true },
];
```

then in app.module.ts

```ts
providers: [
  httpInterceptorProviders
],
```

## Summary

As we’ve seen in these examples, interceptors provide a straightforward mechanism to interact with HTTP requests and responses. This makes it easy to add layers of control and provide more functionality throughout an application without duplicating logic.

## Helpful links

- [Angular docs](https://angular.io/guide/http#intercepting-requests-and-responses)
- [Demo app](https://ng-interceptors.netlify.com)
- [Github](https://github.com/schuchard/ng-interceptors)