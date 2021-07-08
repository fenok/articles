# My solution for react-router type safety

I like my code fully typed. Unfortunately, react-router has always been non-cooperative in providing type safety for route parameters.

It's possible to use [generatePath](https://reactrouter.com/web/api/generatePath) to build path with parameters and have some typing, but it's [not perfect](https://github.com/DefinitelyTyped/DefinitelyTyped/issues/52914). There is no built-in way to build query or hash, let alone type them, and there is no type safety for route state either.

Parsing is even worse. There is no built-in way to parse query or hash, and almost all typing is done by casting, which is error-prone.

There are some libraries that allow type safety, but they are more or less incomplete and restrictive. The best I've seen so far is [typesafe-routes](https://www.npmjs.com/package/typesafe-routes), but it doesn't provide type safety for route state and hash, and it puts restrictions on what paths can be used. For instance, it doesn't support custom regexps for parameters.

## The solution

Enter [react-router-typesafe-routes](https://www.npmjs.com/package/react-router-typesafe-routes). It tries to be as comprehensive, extensible and non-restrictive as possible.

### Route definition

```typescript
const someRoute = route(path("/path/:id"), query(), hash(), state());
```

There are several helpers that process different route parts:

-   `path` uses [generatePath](https://reactrouter.com/web/api/generatePath) to build parametrized path, which allows usage of any path string that's compatible with react-router. On parse, it performs various checks on the given params object to ensure that it belongs to the given route. By default, it infers path params types from the path string in the same way as [generatePath](https://reactrouter.com/web/api/generatePath) does.
-   `query` uses [query-string](https://www.npmjs.com/package/query-string), which can be configured, to build and parse query string. By default, it uses the same types for query params as [query-string](https://www.npmjs.com/package/query-string) does.
-   `hash` just takes care of the `#` symbol on building and parsing hash string. By default, it uses the `string` type.
-   `state` is some ad-hoc helper written by the user. The library doesn't provide a generic helper for route state processing.

As expected, the types can be improved:

```typescript
const someRoute = route(
    path("/path/:id(\\d+)?", { id: param.number.optional }),
    query({ search: param.string.optional("") }), // Use "" as a fallback
    hash("about", "subscribe")
);
```

The `param` helper defines a set of transformers that transform values on building and parsing. The `.optional` modifier means that the value can be `undefined`, and an unsuccessful parsing will use it as a fallback. It's also possible to specify the fallback, which should be particularly useful for query params.

Note that query params have to be `.optional` because of their nature. React-router doesn't consider query on route matching, and the app shouldn't break on manual URL changes.

The transformers are very permissive. It's possible to store arrays in a query and even in a path, and it's possible to write custom transformers.

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

## Final words

More detailed description is available at the [project page](https://github.com/fenok/react-router-typesafe-routes#readme).
