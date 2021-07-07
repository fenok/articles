# My solution for react-router type safety

I like my code fully typed. Unfortunately, react-router has always been non-cooperative in providing type safety for route parameters.

It's possible to use [generatePath](https://reactrouter.com/web/api/generatePath) to build path with parameters and have some typing, but it's [not perfect](https://github.com/DefinitelyTyped/DefinitelyTyped/issues/52914). There is no built-in way to build query or hash, let alone type them, and there is no type safety for route state either.

Parsing is even worse. If some path parameter has multiple values, they will be parsed as a single string with dash-separated values instead of an array. There is no built-in way to parse query or hash, and all typing is done by casting, which is error-prone.

There are some libraries that allow type safety, but they are more or less incomplete and restrictive. The best I've seen so far is [typesafe-routes](https://www.npmjs.com/package/typesafe-routes), but it doesn't provide type safety for route state and hash, and it puts restrictions on what paths can be used. For instance, it doesn't support custom regexps for parameters.

Enter [react-router-typesafe-routes](https://www.npmjs.com/package/react-router-typesafe-routes). It tries to be as comprehensive, extensible and non-restrictive as possible.