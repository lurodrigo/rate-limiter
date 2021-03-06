
[clojars-badge]: https://img.shields.io/clojars/v/lurodrigo/ratelimiter.svg
[clojars]: http://clojars.org/lurodrigo/ratelimiter
[github-issues]: https://github.com/lurodrigo/ratelimiter/issues
[license-badge]: https://img.shields.io/badge/license-MIT-blue.svg
[license]: ./LICENSE
[resilience4clj]: https://github.com/resilience4clj
[upstream-docs]: https://resilience4j.readme.io/docs/ratelimiter
[cache-effect]: https://github.com/resilience4clj/resilience4clj-cache#using-as-an-effect

# Rate Limiter

This library allows you to decorate a function with a rate limiter. This project's structure borrows heavily from
[Resilicience4clj][resilience4clj]. Hopefully it will be merged there soon :)

[![Clojars][clojars-badge]][clojars]

## Getting Started

Add `lurodrigo/ratelimiter` as a dependency to your
`deps.edn` file:

``` clojure
lurodrigo/ratelimiter {:mvn/version "0.1.3"}
```

If you are using `lein` instead, add it as a dependency to your
`project.clj` file:

``` clojure
[lurodrigo/ratelimiter "0.1.3"]
```

Require the library:

``` clojure
(require '[ratelimiter.core :as r])
```

Then create a rate limiter calling the function `create`:

``` clojure
(def rate-limiter (r/create "my-ratelimiter" {:limit-for-period     3
                                            :limit-refresh-period 3000
                                            :timeout-duration     4000}))
```

Now you can decorate any function you have with the rate limiter you just
defined.

```clojure
(defn my-print
  [msg]
  (println (str "At " (java.time.LocalDateTime/now) ": " msg "\n")))

(def dprint (r/decorate my-print rate-limiter))
```

According to the [upstream docs][upstream-docs]:

> Resilience4j provides a RateLimiter which splits all nanoseconds from the
> start of epoch into cycles. Each cycle has a duration configured by 
> `RateLimiterConfig.limitRefreshPeriod`. At the start of each cycle, 
> the RateLimiter sets the number of active permissions to `RateLimiterConfig.limitForPeriod`.
>

We can test that behavior with the following code:

```clojure
(doseq [x ["A" "B" "C" "D" "E" "F" "G" "H" "I"]]
  (dprint x))
```

The first three letters should be printed at almost the same time, immediately. 
The next three should be printed between 0 and 3 seconds after that. 
Then, after 3 more seconds, the final three letters will be printed. Let's look at the output:

```
At 2020-07-28T21:16:52.475560: A

At 2020-07-28T21:16:52.476211: B

At 2020-07-28T21:16:52.476757: C

At 2020-07-28T21:16:54.661593: D

At 2020-07-28T21:16:54.662235: E

At 2020-07-28T21:16:54.662693: F

At 2020-07-28T21:16:57.661574: G

At 2020-07-28T21:16:57.662249: H

At 2020-07-28T21:16:57.662785: I
```

Just as expected! Now, let's try to print all nine letters at the same time:

```clojure
(import io.github.resilience4j.ratelimiter.RequestNotPermitted)

(doseq [x ["A" "B" "C" "D" "E" "F" "G" "H" "I"]]
  (future
    (try
      (dprint x)
      (catch RequestNotPermitted e
      (my-print (.getMessage e))))))
```

What is the expected behavior? Well, some three letters will be printed immediately. Then, after some time between
0 and 3 seconds, another three letters will be printed. Finally, ~4 seconds after the code was sent for evaluation,
`RequestNotPermitted` will be sent for the last three letters. Let's check:

```
At 2020-07-28T21:26:19.024148: A

At 2020-07-28T21:26:19.024150: C

At 2020-07-28T21:26:19.024156: B

At 2020-07-28T21:26:21.661503: D
At 2020-07-28T21:26:21.661503: E

At 2020-07-28T21:26:21.661503: F


At 2020-07-28T21:26:23.025852: RateLimiter 'my-ratelimiter' does not permit further calls

At 2020-07-28T21:26:23.026161: RateLimiter 'my-ratelimiter' does not permit further calls

At 2020-07-28T21:26:23.026509: RateLimiter 'my-ratelimiter' does not permit further calls
```


## Rate Limiter Settings

When creating a rate limiter, you can fine tune three of its settings:

1. `:limit-for-period` - The number of permissions available during one limit refresh period. Default is 50.
2. `:limit-refresh-period` - The period of a limit refresh. After each period the rate limiter sets its permissions
 count back to the limit-for-period value. Default is 0.0005 ms (500 ns).
3. `:timeout-duration` - The default wait time a thread waits for a permission. Default value is 5000 (ms). 

These three options can be sent to `create` as a map. In the following
example, any function decorated with `rate-limiter` will be attempted for 10
times with in 300ms intervals.

``` clojure
(def rate-limiter (r/create "my-rate-limiter" {:limit-for-period 10
                                               :limit-refresh-period 10000}))
```

The function `config` returns the configuration of a rate-limiter in case
you need to inspect it. Example:

``` clojure
(r/config rate-limiter)
=> {:limit-refresh-period #object[java.time.Duration 0x5fbf7654 "PT10S"],
    :limit-for-period 10,
    :timeout-duration #object[java.time.Duration 0x6e0113dc "PT5S"]}
```

Using `change`, it's possible to change some of the rate limiter's configs after its
creation. Currently, it's possible to change `limit-for-period` and `timeout-duration`.

```clojure
(r/change rate-limiter {:limit-for-period 20})
```

## Fallback Strategies

When decorating your function with a rate limiter you can opt to have a fallback function. This function 
will be called instead of an exception being thrown when the call would fail (its traditional throw). 
This feature can be seen as an obfuscation of a try/catch to consumers.

This is particularly useful if you want to obfuscate from consumers that the retry and/or that the external dependency failed. Example:

```clojure
(def rate-limiter (r/create "hello-service-rate-limiter"))

(defn hello [person]
  ;; hypothetical flaky, external HTTP request
  (str "Hello " person))

(def limited-hello
  (r/decorate hello rate-limiter
              {:fallback (fn [e person]
                           (str "Hello from fallback to " person))}))
```

The signature of the fallback function is the same as the original function plus an exception as the
 first argument (e on the example above). This exception is an ExceptionInfo wrapping around the real cause of the error. You can inspect the :cause node of this exception to learn about the inner exception:

When considering fallback strategies there are usually three major strategies:

1. **Failure**: the default way for Resilience4clj - just let the exception flow - is called a "Fail Fast" approach (the call will fail fast once there are no permits left). Another approach is "Fail Silently". In this approach the fallback function would simply hide the exception from the consumer (something that can also be done conditionally).

2. **Content Fallback**: some examples of content fallback are returning "static content" (where a failure would always yield the same static content), "stubbed content" (where a failure would yield some kind of related content based on the paramaters of the call), or "cached" (where a cached copy of a previous call with the same parameters could be sent back).

3. **Advanced**: multiple strategies can also be combined in order to create even better fallback strategies.

## Effects

A common issue for some fallback strategies is to rely on a cache or other content source (see Content Fallback above). 
In these cases, it is good practice to persist the successful output of the function call as a side-effect of the call itself.

Resilience4clj retry supports this behavior in the following way:

The signature of the effect function is the same as the original function plus a "return" argument as the first argument (ret on the example above). This argument is the successful return of the encapsulated function.

The effect function is called on a separate thread so it is non-blocking.

```clojure
(def rate-limiter (r/create "hello-service-rate-limiter"))

(defn hello [person]
  ;; hypothetical flaky, external HTTP request
  (str "Hello " person))

(def limited-hello
  (r/decorate hello rate-limiter
              {:effect (fn [ret person]
                          ;; ret will have the successful return from `hello`
                          ;; you can save it on a memory cache, disk, etc
                          )}))
```

You can see an example of how to use effects for caching purposes at using [Resilience4clj cache as an effect][cache-effect].

## Metrics

The function `metrics` returns a map with the rate limiter's metrics:

``` clojure
(r/metrics rate-limiter)

=> {:available-permissions 10, 
    :number-of-waiting-threads 0}
```

The nodes should be self-explanatory.

## Events

TODO

## Bugs

If you find a bug, submit a [Github issue][github-issues].

## License

Copyright © 2020 Luiz Rodrigo de Souza

Distributed under the MIT License.
