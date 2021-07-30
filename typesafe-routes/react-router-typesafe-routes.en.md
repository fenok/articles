# My solution for react-router type safety

I like my code fully typed. Unfortunately, type safety for route parameters has never been a strong suit of react-router.

If all you need is to build a path with parameters, the use of [generatePath](https://reactrouter.com/web/api/generatePath) will give you some typing, albeit [not perfect](https://github.com/DefinitelyTyped/DefinitelyTyped/issues/52914). However, there is no built-in way to build a query or a hash, let alone type them, and there is no type safety for a route state either.

Things get even worse when it comes to parsing. There is no built-in way to parse a query or a hash, and all typing is almost exclusively done by casting, which is error-prone.

There are some libraries for providing this type safety, but they are more or less incomplete and restrictive. The best I've seen so far is [typesafe-routes](https://www.npmjs.com/package/typesafe-routes), but it offers no type safety for a route state and a hash, and it puts restrictions on what paths can be used. For instance, it doesn't support custom regexps for parameters.

## The solution

Enter [react-router-typesafe-routes](https://www.npmjs.com/package/react-router-typesafe-routes). It tries to be as comprehensive, extensible, and non-restrictive as possible.

### Route definition

```typescript
import { route, path, query, hash } from "react-router-typesafe-routes";
import { state } from "./path/to/state";

const someRoute = route(path("/path/:id"), query(), hash(), state());
```

There are several helpers for processing different parts of a route:

-   `path` uses [generatePath](https://reactrouter.com/web/api/generatePath) to build a path string, making it possible to use any path template that's compatible with react-router. When it comes to parsing, react-router extracts params from a path string, and `path` performs various checks on these params to ensure that they belong to the specified path template. By default, `path` infers types of path params from a path template in the same way as [generatePath](https://reactrouter.com/web/api/generatePath) does.
-   `query` uses (configurable!) [query-string](https://www.npmjs.com/package/query-string) to build and parse a query string. By default, `query` uses the same types for query params as [query-string](https://www.npmjs.com/package/query-string) does.
-   `hash` just takes care of the `#` symbol while building and parsing a hash string. By default, `hash` uses the `string` type for a hash.
-   `state` is an ad-hoc helper written by the user. The library doesn't provide a generic helper for route state processing.

As expected, the types can be improved:

```typescript
import { route, path, query, hash, param } from "react-router-typesafe-routes";

const someRoute = route(
    path("/path/:id(\\d+)?", { id: param.number.optional }),
    query({ search: param.string.optional("") }), // Use "" as a fallback
    hash("about", "subscribe")
);
```

The `param` helper defines a set of transformers that transform values while building and parsing. The built-in transformers are `param.string`, `param.number`, `param.boolean`, `param.null`, `param.date`, `param.oneOf()`, and `param.arrayOf()`.

The `optional` modifier means that the corresponding value can be `undefined`. An unsuccessful parsing of an `optional` parameter will also result in `undefined`. It's possible to specify a fallback value that will be returned instead of `undefined`. This should be particularly useful for query params.

Note that query params are `optional` by their nature. React-router doesn't consider the query part on route matching, and the app shouldn't break in case of manual URL changes.

The transformers are very permissive. It's possible to (natively!) store arrays in a query and even in a path, and it's possible to write custom transformers for storing any serializable data.

### Route usage

Use `Route` components as usual:

```typescript jsx
import { Route } from "react-router";
import { someRoute } from "./path/to/routes";

<Route path={someRoute.path} />;
```

Use `Link` components as usual:

```typescript jsx
import { Link } from "react-router-dom";
import { someRoute } from "./path/to/routes";

// Everything is fully typed!
<Link to={someRoute.build({ id: 1 }, { search: "strawberries" }, "about")} />;
<Link to={someRoute.buildLocation({ state: "private" }, { id: 1 }, { search: "strawberries" }, "about")} />;
```

Parse params with usual hooks:

```typescript jsx
import { useParams, useLocation } from "react-router";
import { someRoute } from "./path/to/routes";

// You can use useRouteMatch() instead of useParams()
const { path, query, hash, state } = someRoute.parse(useParams(), useLocation());
```

Parse only what you need:

```typescript jsx
import { useParams, useLocation } from "react-router";
import { someRoute } from "./path/to/routes";

// Again, you can also use useRouteMatch()
const path = someRoute.parsePath(useParams());
const query = someRoute.parseQuery(useLocation());
const hash = someRoute.parseHash(useLocation());
const state = someRoute.parseState(useLocation());
```

## Notes

A more detailed description is available at the [project page](https://github.com/fenok/react-router-typesafe-routes#readme). The library requires battle-testing and has yet to reach version `1.0.0`.
