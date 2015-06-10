# Middleware Signature

A _Relay_ middleware callable must have the following signature:

```php
use Psr\Http\Message\ResponseInterface as Response;
use Psr\Http\Message\ServerRequestInterface as Request;

function (
    Request $request,   // the incoming request
    Response $response, // the outgoing response
    callable $next      // the next middleware
) {
    // ...
}
```

A _Relay_ middleware callable must return an implementation of _Psr\Http\Message\ResponseInterface_.

# Middleware Dispatching

Create a `$queue` array of middleware callables:

```php
$queue[] = function (Request $request, Response $response, callable $next) {
    // 1st middleware
};

$queue[] = function (Request $request, Response $response, callable $next) {
    // 2nd middleware
};

// ...

$queue[] = function (Request $request, Response $response, callable $next) {
    // Nth middleware
};
```

Create a _Relay_ with the `$queue`, and invoke it with a request and response.

```php
/**
 * @var \Psr\Http\Message\ServerRequestInterface $request
 * @var \Psr\Http\Message\ResponseInterface $response
 */

use Relay\Relay;

$dispatcher = new Relay($queue);
$dispatcher($request, $response);
```

That will execute each of the middlewares in first-in-first-out order.

# Middleware Logic

Your middleware logic should follow this pattern:

- Receive the incoming request and response objects from the previous middleware as parameters, along with the next middleware as a callable.

- Optionally modify the received request and response as desired.

- Optionally invoke the next middleware with the request and response, receiving a new response in return.

- Optionally modify the returned response as desired.

- Return the response to the previous middleware.

Here is a skeleton example; your own middleware may or may not perform the various optional processes:

```php
use Psr\Http\Message\ResponseInterface as Response;
use Psr\Http\Message\ServerRequestInterface as Request;

$queue[] = function (Request $request, Response $response, callable $next) {

    // optionally modify the incoming request
    $request = $request->...;

    // optionally skip the $next middleware and return early
    if (...) {
        return $response;
    }

    // optionally invoke the $next middleware and get back a new response
    $response = $next($request, $response);

    // optionally modify the Response if desired
    $response = $response->...;

    // NOT OPTIONAL: return the Response to the previous middleware
    return $response;
};
```

> N.b.: You should **always** return the response from your middleware logic.

Remember that the request and response are **immutable**. Implicit in that is the fact that changes to the request are always transmitted to the `$next` middleware but never to the previous one.

Note also that this logic chain means the request and response are subjected to two passes through each middleware:

- first on the way "in" through each middleware via the `$next` middleware invocation,

- then on the way "out" from each middleware via the `return` to the previous middleware.

For example, if the middleware queue looks like this:

```php
$queue[] = function (Request $request, Response $response, callable $next) {
    // "Foo"
};

$queue[] = function (Request $request, Response $response, callable $next) {
    // "Bar"
};

$queue[] = function (Request $request, Response $response, callable $next) {
    // "Baz"
};
```

... the request and response path through the middlewares will look like this:

```
Foo is 1st on the way in
    Bar is 2nd on the way in
        Baz is 3rd on the way in, and 1st on the way out
    Bar is 2nd on the way out
Foo is 3rd on the way out
```

You can use this dual-pass logic in clever and perhaps unintuitive ways. For example, middleware placed at the very start may do nothing with the request and call `$next` right away, but it is the middleware with the "real" last opportunity to modify the response.

# Resolvers

You may wish to use `$queue` entries other than anonymous functions. If so, you can pass a `$resolver` callable to the _Relay_ that will convert the `$queue` entry to a callable. Thus, using a `$resolver` allows you to pass in your own factory mechanism for `$queue` entries.

For example, this `$resolver` will naively convert `$queue` string entries to new class instances:

```php
$resolver = function ($class) {
    return new $class();
};
```

You can then add `$queue` entries as class names, and the _Relay_ will use the
`$resolver` to create the objects in turn.

```php
use Relay\Relay;

$queue[] = 'FooMiddleware';
$queue[] = 'BarMiddleware';
$queue[] = 'BazMiddleware';

$dispatcher = new Relay($queue, $resolver);
```

As long as the classes listed in the `$queue` implement `__invoke(Request $request, Response $response, callable $next)`, then the _Relay_ will work correctly.

# Queue Object and Relay Builder

Sometimes using an array for the `$queue` will not be suitable. You may wish to use an object to build retain the middleware queue.

If so, you can use the _RelayBuilder_ to create the _Relay_ queue from any object that extends _ArrayObject_ or that implements _Relay\GetArrayCopyInterface_. The _RelayBuilder_ will then get an array copy of that queue object for the _Relay_.

For example, if your `$queue` is an _ArrayObject_, first instantiate a _RelayBuilder_ with an optional `$resolver` ...

```php
use Relay\RelayBuilder;

$builder = new RelayBuilder($resolver);
```

... then instantiate a _Relay_ where `$queue` is an array, an _ArrayObject_, or a _Relay\GetArrayCopyInterface_ implementation:

```php
/**
 * var array|ArrayObject|Relay\GetArrayCopyInterface $queue
 */
$relay = $builder->newInstance($queue);
```

You can then use the `$relay` as described above.