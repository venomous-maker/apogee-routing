# apogee-routing

Routing for [Apogee](https://github.com/venomous-maker): routes, groups,
controllers with injection, resource routes, routes declared with
decorators, route model binding, URL generation and signed URLs,
middleware aliases, groups and parameters, rate limiting, and the HTTP
kernel that turns exceptions into responses and serves it all on the
standard library's HTTP server.

```grav
import "std"
import "apogee-routing"

let r = new Router(container)
r.get("/", fn(req: Request) -> Response { return Response.text("home") }).name("home")
r.get::<UserController>("/users/{id:int}", "show").name("users.show")
r.prefix("admin").name("admin.").middleware(["auth", "throttle:60,1"]).group(fn(r: Router) {
    r.resource::<PostController>("posts")      // index create store show edit update destroy
})
r.bind("post", fn(id: string, req: Request) -> any { return Post.find(id) })   // {post}: a Post, or 404
r.discover()                                    // routes declared with decorators on controllers

let kernel = new HttpKernel(r)
kernel.serve(8080)
```

## Routes

`get`, `post`, `put`, `patch`, `delete`, `options`, `any`,
`matching(["GET", "POST"], ...)`, `redirect(from, to)`, `fallback(h)`. A
handler is `fn(Request) -> Response`, or a controller action:
`get::<C>(uri, "method")`.

Patterns: `{id}`, `{id:int}` (shorthands `int`, `alpha`, `alnum`, `slug`,
`uuid`, `ulid`, `hex`, or a regular expression: `{code:[A-Z]{3}}`),
`{page?}` (optional, with `defaultValue`), `{path*}` (the rest of the path).
On a route: `.where(param, pattern)`, `whereNumber`, `whereAlpha`,
`whereUuid`, `whereIn(param, values)`, `.name()`, `.middleware()`,
`.withoutMiddleware()`, `.signed()`. `router.pattern(param, pattern)`
applies to every route.

A literal segment beats a parameter, a constrained parameter beats an open
one, and a dead end backs out to try the next branch. A path that exists for
other methods is a 405 with `Allow`; `OPTIONS` is answered from the routes;
`HEAD` runs the `GET` route; a trailing slash is ignored.

## Groups

```grav
r.prefix("api/v1").name("api.").middleware(["api"]).where("id", "int").group(fn(r: Router) { ... })
```

Groups nest: prefixes join, name prefixes join, middleware adds up, and
`withoutMiddleware` takes some back for the routes inside.

## Controllers

```grav
class PhotoController extends Controller {
    private photos: PhotoRepository
    constructor(photos: PhotoRepository) { self.photos = photos }          // injected
    fn show(req: Request, photo: string) -> Response { ... }               // services, Request, then route values
    fn update(req: Request, photo: string) -> any { return {"ok": true} }  // any result becomes a response
}
r.resource::<PhotoController>("photos")                  // only the actions it has methods for
r.apiResource::<PhotoController>("v1/photos", ["index", "show"])
```

A controller is built from the request's container scope each request, its
constructor injected; its action's parameters are filled by type (services,
the `Request`), then with route values (bound models where `bind` made one)
in order. A result that is not a `Response` becomes one: a string as HTML,
null as 204, anything else as JSON.

### Routes declared with decorators

```grav
@Prefix("/orders") @Name("orders.") @Middleware("auth")
class OrderController extends Controller {
    @Get("/{id}") @Name("show") @Where("id", "int")
    fn show(req: Request, id: string) -> Response { ... }
    @Route("PUT|PATCH", "/{id}") @Middleware("throttle:10,1")
    fn update(req: Request, id: string) -> Response { ... }
}
r.discover()      // every Controller subclass: they all have reflection through Controller's decorator
```

## URLs

```grav
r.url("users.show", {"id": 7, "tab": "posts"})        // "/users/7?tab=posts"
r.url("posts.show", {"post": post})                    // a model: its routeKey() (UrlRoutable) or id
r.signedUrl("unsubscribe", {"user": 7}, Time.now() + 3600)
r.hasValidSignature(req)                               // what .signed() routes check (403 otherwise)
```

Signatures are HMAC-SHA256 under `router.signingKey` (the app key),
compared in constant time, and may expire.

## Middleware

```grav
r.aliasMiddleware("throttle", fn(args: Array<string>) -> Middleware { return new ThrottleRequests(limits, args) })
r.aliasInstance("auth", new Authenticate(guard))
r.middlewareGroup("web", ["session", "csrf"])
r.use("request-id")                                   // on every route
```

A name may carry parameters (`"throttle:60,1"`), name a group, or name a
`Middleware` class bound in the container. Each runs once per route, where
it first appears. A route's middleware and handler are joined into one
function when routes are compiled: a request runs through plain calls.

## Rate limiting

`ThrottleRequests` counts hits per client in fixed windows, in memory:
`"throttle:60,1"` is 60 a minute per IP, and `"throttle:api"` uses a named
limiter, chosen per request:

```grav
limits.define("api", fn(req: Request) -> Limit {
    return req.user != null ? Limit.perMinute(600).by("user:${userId(req)}") : Limit.perMinute(60)
})
```

Responses carry `X-RateLimit-Limit` and `X-RateLimit-Remaining`; past the
limit, a 429 with `Retry-After`.

## The kernel

`HttpKernel` runs the router and turns exceptions into responses: a renderer
registered for the exception's kind, else a `Renderable` exception's own
response (`HttpException`, validation errors), else a 500 that is reported
first (`report(f)`) and shows the message and trace only with `debug` on.
`after(f)` runs once the response is made; `serve(port)` blocks and
`start(port)` serves in the background.

## Speed

Routes are compiled once: literal paths go into one hash table, the rest
into a segment tree, and middleware is joined per route. A request's
container scope is made only if something asks for it (a controller,
`req.make`), and ended as soon as the response is made. In-process,
`--release`: about 1.1 µs per request for a literal route, 3.2 µs with two
parameters, 4.7 µs for a controller action built with injection (the
response object included). On the standard library's server, a JSON route
answered about 61,000 requests a second (`ab -k -c 64`, the same 8-thread
laptop running both).

## Tests

```bash
grav test tests/routing.grav
```

MIT licensed.
