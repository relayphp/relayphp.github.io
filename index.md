---
layout: site
---
NOTE: Relay recently received some backwards-compatiblilty breaking changes. You may wish to lock your composer.json entry to [0.2.0](https://github.com/relayphp/Relay.Relay/releases/tag/0.2.0), then read the most-recent [CHANGES](https://github.com/relayphp/Relay.Relay/blob/1.x/CHANGES.md) and/or the new documentation below, before upgrading to the 1.x development version.

# Middleware Signature

A _Relay_ middleware callable must have the following signature:

{% highlight php %}
<?php
use Psr\Http\Message\ResponseInterface as Response;
use Psr\Http\Message\RequestInterface as Request;

function (
    Request $request,   // the request
    Response $response, // the response
    callable $next      // the next middleware
) {
    // ...
}
?>
{% endhighlight %}

A _Relay_ middleware callable must return an implementation of _Psr\Http\Message\ResponseInterface_.

This signature makes _Relay_ appropriate for both server-related and client-related use cases. That is, it can receive an incoming _ServerRequestInterface_ and generate an outgoing _ResponseInterface_ (acting as a server), or it can build an outgoing _RequestInterface_ and return the resulting _ResponseInterface_ (acting as a client).

> N.b.: Psr\Http\Message\ServerRequestInterface extends RequestInterface, so typehinting to RequestInterface covers both use cases.

# Middleware Dispatching

Create a `$queue` array of middleware callables:

{% highlight php %}
<?php
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
?>
{% endhighlight %}

Use the _RelayBuilder_ to create a _Relay_ with the `$queue`, and invoke the _Relay_ with a request and response.

{% highlight php %}
<?php
/**
 * @var \Psr\Http\Message\RequestInterface $request
 * @var \Psr\Http\Message\ResponseInterface $response
 */

use Relay\RelayBuilder;

$relayBuilder = new RelayBuilder();
$relay = $relayBuilder->newInstance($queue);
$response = $relay($request, $response);
?>
{% endhighlight %}

That will execute each of the middlewares in first-in-first-out order.

# Middleware Logic

Your middleware logic should follow this pattern:

- Receive the request and response objects from the previous middleware as parameters, along with the next middleware as a callable.

- Optionally modify the received request and response as desired.

- Optionally invoke the next middleware with the request and response, receiving a new response in return.

- Optionally modify the returned response as desired.

- Return the response to the previous middleware.

Here is a skeleton example; your own middleware may or may not perform the various optional processes:

{% highlight php %}
<?php
use Psr\Http\Message\ResponseInterface as Response;
use Psr\Http\Message\RequestInterface as Request;

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
?>
{% endhighlight %}

> N.b.: You MUST return the response from your middleware logic.

Remember that the request and response are **immutable**. Implicit in that is the fact that changes to the request are always transmitted to the `$next` middleware but never to the previous one.

Note also that this logic chain means the request and response are subjected to two passes through each middleware:

- first on the way "in" through each middleware via the `$next` middleware invocation,

- then on the way "out" from each middleware via the `return` to the previous middleware.

For example, if the middleware queue looks like this:

{% highlight php %}
<?php
$queue[] = function (Request $request, Response $response, callable $next) {
    // "Foo"
};

$queue[] = function (Request $request, Response $response, callable $next) {
    // "Bar"
};

$queue[] = function (Request $request, Response $response, callable $next) {
    // "Baz"
};
?>
{% endhighlight %}

... the request and response path through the middlewares will look like this:

    Foo is 1st on the way in
        Bar is 2nd on the way in
            Baz is 3rd on the way in, and 1st on the way out
        Bar is 2nd on the way out
    Foo is 3rd on the way out

You can use this dual-pass logic in clever and perhaps unintuitive ways. For example, middleware placed at the very start may do nothing with the request and call `$next` right away, but it is the middleware with the "real" last opportunity to modify the response.

# Resolvers

You may wish to use `$queue` entries other than anonymous functions or already-instantiated objects. If so, you can pass a `$resolver` callable to the _Relay_ that will convert the `$queue` entry to a callable. Thus, using a `$resolver` allows you to pass in your own factory mechanism for `$queue` entries.

For example, this `$resolver` will naively convert `$queue` string entries to new class instances:

{% highlight php %}
<?php
$resolver = function ($class) {
    return new $class();
};
?>
{% endhighlight %}

You can then add `$queue` entries as class names, and the _Relay_ will use the
`$resolver` to create the objects in turn.

{% highlight php %}
<?php
use Relay\RelayBuilder;

$queue[] = 'FooMiddleware';
$queue[] = 'BarMiddleware';
$queue[] = 'BazMiddleware';

$relayBuilder = new RelayBuilder($resolver);
$relay = $relayBuilder->newInstance($queue);
?>
{% endhighlight %}

As long as the classes listed in the `$queue` implement `__invoke(Request $request, Response $response, callable $next)`, then the _Relay_ will work correctly.

# Queue Object

Sometimes using an array for the `$queue` will not be suitable. You may wish to use an object to build retain the middleware queue instead.

In these cases, you can use the _RelayBuilder_ to create the _Relay_ queue from any object that extends _ArrayObject_ or that implements _Relay\GetArrayCopyInterface_. The _RelayBuilder_ will then get an array copy of that queue object for the _Relay_.

For example, if your `$queue` is an _ArrayObject_, first instantiate a _RelayBuilder_ with an optional `$resolver` ...

{% highlight php %}
<?php
use Relay\RelayBuilder;

$relayBuilder = new RelayBuilder($resolver);
?>
{% endhighlight %}

... then instantiate a _Relay_ where `$queue` is an array, an _ArrayObject_, or a _Relay\GetArrayCopyInterface_ implementation:

{% highlight php %}
<?php
/**
 * var array|ArrayObject|Relay\GetArrayCopyInterface $queue
 */
$relay = $relayBuilder->newInstance($queue);
?>
{% endhighlight %}

You can then use the `$relay` as described above.

# Resuable Relays

If you wish, you can reuse the same _Relay_ object multiple times. The same middleware queue will be used each time you invoke that _Relay_. For example, if you are making multiple client requests:

{% highlight php %}
/**
 * @var Psr\Http\Message\ResponseInterface $response1
 * @var Psr\Http\Message\ResponseInterface $response2
 * @var Psr\Http\Message\ResponseInterface $responseN
 */
$request1 = ...;
$response1 = $relay($request1, $response1);

$request2 = ...;
$response2 = $relay($request2, $response2);

// ...

$requestN = ...;
$responseN = $relay($requestN, $responseN);
{% endhighlight %}

If you are certain that you will never reuse the _Relay_ instance, you can instantiate a single-use _Runner_ to avoid some minor overhead.

{% highlight php %}
/**
 * @var array $queue The middleware queue.
 * @var callable|null $resolver An optional queue entry resolver.
 * @var Psr\Http\Message\RequestInterface $request The HTTP request.
 * @var Psr\Http\Message\ResponseInterface $response The HTTP response.
 */
use Relay\Runner;

$runner = new Runner($queue, $resolver);
$response = $runner($request, $response);
{% endhighlight %}
