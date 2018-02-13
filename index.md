---
layout: site
---

<div style="font-size: 80%; text-align: center; color: gray;">For middleware to use with Relay, please review <a href="https://github.com/middlewares/psr15-middlewares">middlewares/psr15-middlewares</a>.</div>

_Relay 2_ is a [PSR-15 server request handler][RequestHandlerInterface] (aka "dispatcher") for a queue of [PSR-15 middleware][MiddlewareInterface] entries.

## Request Handling

First, create an array or [traversable](http://php.net/traversable) `$queue` of middleware entries:

```php
$queue[] = new FooMiddleware();
$queue[] = new BarMiddleware();
$queue[] = new BazMiddleware();
// ...
$queue[] = new ResponseFactoryMiddleware {
    return new Response();
};
```

Then create a _Relay_ with the `$queue` and call the `handle()` method with a server request.

```php
/**
 * @var \Psr\Http\Message\RequestInterface $request
 * @var \Psr\Http\Message\ResponseInterface $response
 */

use Relay\Relay;

$relay = new Relay($queue);
$response = $relay->handle($request);
```

## Queue Entry Resolvers

You may wish to use `$queue` entries other than already-instantiated objects. If so, you can pass a `$resolver` callable to _Relay_ that will convert the `$queue` entry to an instance. Thus, using a `$resolver` allows you to pass in your own factory mechanism for `$queue` entries.

For example, this `$resolver` will naively convert `$queue` string entries to new class instances:

```php
$resolver = function ($entry) {
    return new $entry();
};
```

You can then add `$queue` entries as class names, and _Relay_ will use the `$resolver` to create the objects in turn.

```php
$queue[] = 'FooMiddleware';
$queue[] = 'BarMiddleware';
$queue[] = 'BazMiddleware';
$queue[] = 'ResponseFactoryMiddleware';

$relay = new Relay($queue, $resolver);
```

## Callable Middleware

_Relay_ can also handle any middleware entries that are callables with the following signature:

```php
use Psr\Http\Message\ResponseInterface as Response;
use Psr\Http\Message\RequestInterface as Request;
use Psr\Http\Server\RequestHandlerInterface as Handler;

function (
    Request $request, // the request
    callable $next // the next middleware or handler
) : Response {
    // ...
}
```

Callable middleware may be intermingled with PSR-15 middleware.

[RequestHandlerInterface]: https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-15-request-handlers.md#21-psrhttpserverrequesthandlerinterface
[MiddlewareInterface]: https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-15-request-handlers.md#22-psrhttpservermiddlewareinterface

