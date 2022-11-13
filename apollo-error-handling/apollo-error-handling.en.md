![Cover](resources/cover-alt2.png)

# Handling Apollo Errors in React

## Introduction

Let me start with my golden principles of error handling:

- If anything goes wrong, the user should be notified. No exceptions (pun intended).
- We should fall into [The Pit of Success](https://blog.codinghorror.com/falling-into-the-pit-of-success/). That is, error handling should _just work_.
- We should be able to customize global error handling up to downright disabling it and implementing something completely custom.

Throughout the article, we will apply these principles to handling Apollo errors (both GraphQL and network) in React. We assume the use of Apollo Server and Typescript. SSR is also covered.

## What we want

The way I see things, the best way to adhere to these principles is to implement the following:

- A notification system that allows showing all kinds of messages, including errors.
- A global error handler that shows error messages via the notification system. It can also log errors, send them to your error tracker, redirect to the login page in case of an authentication error, etc.
- A way for us to communicate with the global error handler in order to alter its behavior (e.g. prevent it from showing an error message).

## How to pass GraphQL errors to the client

A quick recap of what GraphQL errors look like (shamelessly borrowed from [GraphQL Spec](http://spec.graphql.org/draft/#example-8b658)):

```json
{
  "errors": [
    {
      "message": "Name for character with ID 1002 could not be fetched.",
      "locations": [{ "line": 6, "column": 7 }],
      "path": ["hero", "heroFriends", 1, "name"],
      "extensions": {
        "code": "CAN_NOT_FETCH_BY_ID",
        "timestamp": "Fri Feb 9 14:33:09 UTC 2018"
      }
    }
  ]
}
```

- There can be multiple `errors`.
- The `message` field is a [description of the error intended for the developer](http://spec.graphql.org/draft/#sel-GAPHRPDCAZCCyD5lH) (please don't show it to the user).
- The `locations` and `path` fields point to the associated points in the request and the response field respectively. That is, errors are detached from where they occurred, and we need to implement some custom logic to parse the `path` field if we want to know what field caused the error. Since the `null` result [will bubble up](http://spec.graphql.org/draft/#sel-GAPHRPTCAACExBpsd) to the nearest nullable field, it may be tricky.
- The `extensions` field is a map of arbitrary data associated with the error. We specifically care about its `code` field which is set by Apollo Server (see [built-in error codes](https://www.apollographql.com/docs/apollo-server/data/errors/#error-codes)). It should be used to differentiate errors on the client side.
- Errors are not a part of the GraphQL Schema. Standard fields like `message`, `locations`, `path`, and (Apollo-specific) `extensions.code` can (somewhat) be typed, but the rest of the `extensions` field is a wild west.

### Consider treating errors as data

There is an approach to handling errors where they are added to the schema ([by using unions and interfaces](https://blog.logrocket.com/handling-graphql-errors-like-a-champ-with-unions-and-interfaces/) or by providing an additional field). It basically says "Let's add errors to GraphQL Schema to have typed errors that are co-located to where they occurred".

This approach is great, but not always viable. Some errors can happen _anywhere_, and there is no point in polluting your schema with them. One such example is [`UNAUTHENTICATED`](https://www.apollographql.com/docs/apollo-server/data/errors/#unauthenticated) error in a private application. And sure enough, there is always a possibility of an unexpected error.

That is, with this approach, we have both actual errors and errors in data to handle. For the latter, we need to constantly repeat the error handling logic (which doesn't really conform to the "Pit of Success" principle) or create a custom [link](https://www.apollographql.com/docs/react/api/link/introduction/) to handle them, because Apollo doesn’t know that they are, in fact, errors. In contrast, there is a built-in [`onError`](https://www.apollographql.com/docs/react/api/link/apollo-link-error/) link for handling actual errors.

Personally, I think that it's best to use actual errors by default. We should switch to "errors as data" only in specific cases when we _do_ need to know anything else other than `code`. We likely don’t want to use global error handling for them anyway, and there's a good chance that we can avoid them altogether.

### Use null instead of NOT_FOUND in queries

There is [a long-standing issue](https://github.com/apollographql/apollo-client/issues/3897) with error caching (its absence, that is), which leads to errors being rendered as a loading state on the server side. Therefore, if you're doing SSR, you might want to design your schema in such a way that there are no intentional errors in queries whatsoever. To do so, we can use `null` values instead of `NOT_FOUND` errors (see how we essentially treated errors as data here?). Note how nothing stops us from setting the response code to `404` in case of a `null` value, should we want so.

In fact, even if you don't do SSR, you might still benefit from discarding the `NOT_FOUND` errors in queries. There is no [`previousData`](https://www.apollographql.com/docs/react/data/queries/#previousdata) analog for errors, so if you want to render a "not found" page state, it's easier to rely on `data`/`previousData` with a value of `null` (and, honestly, the absence of something in case of a read operation feels more data-like than error-like to me).

Most likely, all other errors can be treated as unintentional and related to the whole query (that is, we don't need to parse the `path` field because we just don't care). There's either a bug in our code (e.g. [`INTERNAL_SERVER_ERROR`](https://www.apollographql.com/docs/apollo-server/data/errors/#internal_server_error)) or the user is doing something malicious (e.g. [`FORBIDDEN`](https://www.apollographql.com/docs/apollo-server/data/errors/#forbidden)). Either way, it's more or less fine to SSR them as a loading state with an appropriate response code and then show a notification (or something custom) on the client side. It's not the most elegant solution, but it's the best we can do given the circumstances.

There is also a special error: [`UNAUTHENTICATED`](https://www.apollographql.com/docs/apollo-server/data/errors/#unauthenticated). Fortunately, we most likely don't need to actually render it, because it should be rendered on the server side as a redirect.

### Rely on client-side validation only

Validation should be completely static. That is, we should be able to validate the user input beforehand. Therefore, any validation errors should be treated as bugs in the frontend validation logic.

It means that validation errors can and should provide `extensions` with detailed validation results, but we shouldn't base our logic on them. We should only apply the global handler to such errors, anticipating that these errors should never happen.

It’s worth noting that updating validation rules is technically a breaking change, so it’s perfectly fine to update client-side code in response to these changes.

### Make error codes specific

In order to avoid non-standard `extensions`, we can make error `code`s more specific. For instance, don't use the `NOT_FOUND` code. Make it `USER_NOT_FOUND`, `JOB_NOT_FOUND`, `MEANING_NOT_FOUND`, and so on.

Specific error codes also aid in determining the origin of the error without parsing the `path` field. Especially for mutations, which are usually much more granular than queries.

For the sake of clarity: unlike queries, it's perfectly normal to use `SOMETHING_NOT_FOUND` errors in mutations.

### Provide enum with error codes

In order to type `extensions.code`, we need to provide an enum with possible error codes. The easiest way to do so is to add it to the GraphQL Schema. Tools like [GraphQL Code Generator](https://www.graphql-code-generator.com/) can then generate the actual type that we can use.

Sadly, there is no guarantee that the enum will ever be complete. There should always be a fallback to an unknown error.

There is also no way to tell what errors the given operation may produce. We can only document them using comments.

### Use standard errors

Use the [built-in error codes](https://www.apollographql.com/docs/apollo-server/data/errors/#error-codes) where appropriate. Use [`ApolloError`](https://www.apollographql.com/docs/apollo-server/data/errors/#custom-errors) to create custom errors. Use standard error code format. Be consistent. Don't reinvent the wheel.

## The notification system

The notification system is highly application-dependent. We can create something custom or use a library like [notistack](https://www.npmjs.com/package/notistack). For the sake of simplicity, we’ll assume the use of notistack from now on, but the actual implementation doesn’t really matter.

With notistack, we wrap the whole application in its provider:

```tsx
import { SnackbarProvider } from "notistack";
import { RestOfTheApp } from "./RestOfTheApp";

const App = () => (
  <SnackbarProvider>
    <RestOfTheApp />
  </SnackbarProvider>
);
```

And then queue notifications like this:

```tsx
import { useSnackbar } from "notistack";

const RestOfTheApp = () => {
  const { enqueueSnackbar } = useSnackbar();

  const handleClick = () => {
    enqueueSnackbar("Button clicked!");
  };

  return <button onClick={handleClick}>Show snackbar</button>;
};
```

As you can see, notistack can only be accessed from within React. Let’s see how we can connect an Apollo client instance to it.

### Creating Apollo client instance within React

The obvious approach is to create an Apollo client instance within React:

```tsx
import { useSnackbar } from "notistack";
import { ApolloProvider } from "@apollo/client";
import { useMemoOne } from "use-memo-one";
import { createApolloClient } from "./createApolloClient";
import { ActualApp } from "./ActualApp";

const RestOfTheApp = () => {
  const { enqueueSnackbar } = useSnackbar();
  const client = useMemoOne(() => createApolloClient({ enqueueSnackbar }), [enqueueSnackbar]);

  return (
    <ApolloProvider client={client}>
      <ActualApp />
    </ApolloProvider>
  );
};
```

We create an Apollo client instance with our own `createApolloClient` function and memoize it with [`useMemoOne`](https://www.npmjs.com/package/use-memo-one), ensuring a semantic guarantee of memoization.

This seems suboptimal, because:

- This will break if `enqueueSnackbar` changes because we will create a new Apollo client instance. This can be fixed with our own `stableEnqueueSnackbar` with a stable identity, which can call the latest `enqueueSnackbar` under the hood, but it will make the code even more complex. And if `RestOfTheApp` itself remounts for some reason, the client instance will be recreated anyway.
- In the case of SSR, we can’t create an Apollo client instance within React, because the app is rendered multiple times during the data fetching process on the server side.

### Creating Apollo client instance outside of React

Passing Apollo client instance in props is definitely nicer:

```tsx
import { ApolloProvider, ApolloClient, NormalizedCacheObject } from "@apollo/client";
import { SnackbarProvider } from "notistack";
import { ActualApp } from "./ActualApp";

interface AppProps {
  apolloClient: ApolloClient<NormalizedCacheObject>;
}

const App = ({ apolloClient }: AppProps) => {
  return (
    <ApolloProvider client={apolloClient}>
      <SnackbarProvider>
        <ActualApp />
      </SnackbarProvider>
    </ApolloProvider>
  );
};
```

However, we now have to connect it to the `enqueueSnackbar` function. This can be done with a mediator. Let’s create a generic one:

```tsx
class Dispatcher<TAction> {
  // We only need a single subscriber, which is just a callback accepting an action
  private subscriber: ((value: TAction) => void) | undefined;

  // If there is no subscriber, we store actions to dispatch them later
  private pendingActions: TAction[] = [];

  public dispatch(action: TAction): void {
    if (this.subscriber) {
      // Upon dispatch, we simply call the subscriber...
      this.subscriber(action);
    } else {
      // ... or store the action if there is no subscriber
      this.pendingActions.push(action);
    }
  }

  public subscribe(subscriber: (value: TAction) => void): () => void {
    this.subscriber = subscriber;

    // Upon subscription, we dispatch all pending actions
    this.pendingActions.forEach((action) => {
      this.subscriber?.(action);
    });

    this.pendingActions = [];

    // We return a callback to enable unsubscribing
    return () => {
      this.subscriber = undefined;
    };
  }
}
```

Then we can create a dispatcher for `enqueueSnackbar`:

```tsx
import { ProviderContext } from "notistack";
import { Dispatcher } from "./Dispatcher";

const snackbarDispatcher =
  // We don't want global stateful objects on the server side
  typeof window !== "undefined" ? new Dispatcher<Parameters<ProviderContext["enqueueSnackbar"]>>() : undefined;
```

And here is how we can connect it to React:

```tsx
import { useSnackbar } from "notistack";
import { snackbarDispatcher } from "./snackbarDispatcher";

// This hook should be used once as early as possible, though below SnackbarProvider
function useSnackbarDispatcher(): void {
  const { enqueueSnackbar } = useSnackbar();

  useEffect(() => {
    const unsubscribe = snackbarDispatcher?.subscribe((args) => enqueueSnackbar(...args));

    return () => {
      unsubscribe?.();
    };
  }, [enqueueSnackbar]);
}
```

The idea is to dispatch `enqueueSnackbar` arguments from an Apollo client instance and then pass them to `enqueueSnackbar` inside React. That is, `snackbarDispatcher.dispatch` is essentially equivalent to `enqueueSnackbar`.

Note that `snackbarDispatcher` is a global stateful object, which is a bad thing for server-side code. Using it on the server side wouldn’t make much sense anyway, but we explicitly create it only on the client side just to be safe.

The `useSnackbarDispatcher` hook should be used as early as possible, though below `SnackbarProvider`. It might be a good idea to create a custom wrapper for `SnackbarProvider`, which can conceal the use of `useSnackbarDispatcher` within itself.

## The global error handler

As mentioned earlier, there is an intended way to handle errors globally: the [`onError`](https://www.apollographql.com/docs/react/api/link/apollo-link-error/) link. With it, we can create a global error handler:

```tsx
import { ErrorResponse } from "@apollo/client/link/error";
import { getNotifications } from "./getNotifications";
import { snackbarDispatcher } from "./snackbarDispatcher";

function globalErrorHandler(error: ErrorResponse) {
  if (snackbarDispatcher) {
    getNotifications(error).forEach((notification) => snackbarDispatcher.dispatch(notification));
  }
}
```

And use it like this:

```tsx
import { ApolloClient, from } from "@apollo/client";
import { onError } from "@apollo/client/link/error";
import { globalErrorHandler } from "./globalErrorHandler";

function createApolloClient() {
  return new ApolloClient({
    // Other options
    link: from([
      onError(globalErrorHandler),
      // Other links
    ]),
  });
}
```

Let’s see how we can implement the `getNotifications` helper.

### Parsing `ErrorResponse`/`ApolloError`

We need helpers to determine if the `ErrorResponse` object from the global error handler is a network error, and, if not, what are its GraphQL errors.

We also want these helpers to be usable for `ApolloError` objects which are returned from e.g. the [`useQuery`](https://www.apollographql.com/docs/react/api/react/hooks/#usequery) hook. Fortunately, we only need the `graphQLErrors` and `networkError` fields, which are essentially the same between `ErrorResponse` and `ApolloError`.

`ApolloError` objects can also be thrown, so we type the `error` arguments as `unknown`.

Finally, we need to address an Apollo [bug](https://github.com/apollographql/apollo-client/issues/9870) that causes GraphQL errors to appear in the network error.

```tsx
import { ServerError, ServerParseError, ApolloError } from "@apollo/client";
import { GraphQLError } from "graphql";

function isNetworkError(error: unknown): error is { networkError: Error | ServerError | ServerParseError } {
  return hasNetworkError(error) && !hasGqlErrors(error);
}

function getGqlErrors(error: unknown): GraphQLError[] {
  const result: GraphQLError[] = [];

  if (hasPopulatedGqlErrors(error)) {
    result.push(...error.graphQLErrors);
  }

  if (hasUnpopulatedGqlErrors(error)) {
    result.push(...error.networkError.result.errors);
  }

  return result;
}

function hasGqlErrors(error: unknown): boolean {
  return hasPopulatedGqlErrors(error) || hasUnpopulatedGqlErrors(error);
}

function hasPopulatedGqlErrors(error: unknown): error is { graphQLErrors: ReadonlyArray<GraphQLError> } {
  return Boolean(
    error &&
      typeof error === "object" &&
      Array.isArray((error as Partial<ApolloError>).graphQLErrors) &&
      (error as ApolloError).graphQLErrors.length
  );
}

function hasUnpopulatedGqlErrors(error: unknown): error is {
  networkError: { result: { errors: ReadonlyArray<GraphQLError> } };
} {
  return Boolean(
    hasNetworkError(error) &&
      "result" in error.networkError &&
      Array.isArray(error.networkError.result.errors) &&
      error.networkError.result.errors.length
  );
}

function hasNetworkError(error: unknown): error is { networkError: Error | ServerError | ServerParseError } {
  return Boolean(error && typeof error === "object" && (error as ApolloError).networkError);
}

export { getGqlErrors, isNetworkError };
```

And then we can implement the `getNotifications` helper:

```tsx
import { ProviderContext } from "notistack";
import { ErrorResponse } from "@apollo/client/link/error";
import { GqlError } from "./graphql-codegen-result";
import { getGqlErrors, isNetworkError } from "./parsing-helpers";

const NOTIFICATION_MAP: Record<GqlError, Parameters<ProviderContext["enqueueSnackbar"]>> = {
  [GqlError.InternalServerError]: ["Something went wrong :(", { variant: "error" }],
};

function getNotifications(error: ErrorResponse) {
  if (isNetworkError(error)) {
    return ["Check your network connection", { variant: "error" }];
  }

  return getGqlErrors(error).map(
    ({ extensions }) =>
      NOTIFICATION_MAP[extensions?.code ?? GqlError.InternalServerError] ??
      NOTIFICATION_MAP[GqlError.InternalServerError]
  );
}
```

As you can see, there can be either a network error or an array of GraphQL errors. In the latter case, we only rely on the `code` field. If it's absent or unknown, we fall back to `INTERNAL_SERVER_ERROR`.

`GqlError` is an enum with possible error codes generated from the GraphQL Schema (as a reminder, its completeness cannot be guaranteed).

### Setting SSR response code

If there was an error during SSR, we shouldn’t return the result with a `200` status code. Therefore, we need a way to tell the renderer that there was an error. This can be done with a custom context. We can create this context on a per-request basis and pass it to `globalErrorHandler` via `createApolloClient` helper:

```tsx
import { ApolloClient, from } from "@apollo/client";
import { ErrorResponse, onError } from "@apollo/client/link/error";
import { getNotifications } from "./getNotifications";
import { snackbarDispatcher } from "./snackbarDispatcher";

interface ApolloContext {
  statusCode?: number;
}

interface CreateApolloClientOptions {
  context?: ApolloContext;
}

interface GlobalErrorHandlerOptions {
  context?: ApolloContext;
}

function createApolloClient({ context }: CreateApolloClientOptions = {}) {
  return new ApolloClient({
    // Other options
    link: from([
      onError(globalErrorHandler({ context })),
      // Other links
    ]),
  });
}

function globalErrorHandler({ context }: GlobalErrorHandlerOptions = {}) {
  return (error: ErrorResponse) => {
    if (snackbarDispatcher) {
      getNotifications(error).forEach((notification) => snackbarDispatcher.dispatch(notification));
    }

    if (context) {
      // We can also set the status code depending on the error code
      context.statusCode = 500;
    }
  };
}
```

Note that context is only needed in the case of SSR, so it’s optional.

### Handling [`UNAUTHENTICATED`](https://www.apollographql.com/docs/apollo-server/data/errors/#unauthenticated) errors

[`UNAUTHENTICATED`](https://www.apollographql.com/docs/apollo-server/data/errors/#unauthenticated) errors can be handled in a similar way, so I leave it as an exercise for the reader. Things to keep in mind:

- Instead of showing a notification, we should redirect to the login page.
- This requires a helper to determine if there is at least one [`UNAUTHENTICATED`](https://www.apollographql.com/docs/apollo-server/data/errors/#unauthenticated) error and a mediator to connect the global error handler to the app-specific login redirect logic. Sure enough, we can use our generic parsing helpers and `Dispatcher` for this.
- We also have to add a `url` field to the context to enable server-side redirects.
- We most likely want to redirect to the original page after successful login. To form a proper login URL on the server side, we need to pass the current location to the global error handler alongside the context.

## Overriding the global error handler

Conveniently, there is a built-in way to associate data (and even functionality) with GraphQL operations: operation context (not to be confused with our custom `ApolloContext` object). We can put there anything we want:

```tsx
import { useQuery } from "@apollo/client";
import { MyDocument } from "./MyDocument";
import { context } from "./global-error-handler";

const { data } = useQuery(MyDocument, {
  context: context({ disableErrorNotification: true }),
});
```

And we can adjust the global error handler accordingly:

```tsx
import { ErrorResponse } from "@apollo/client/link/error";
import { getNotifications } from "./getNotifications";
import { snackbarDispatcher } from "./snackbarDispatcher";

interface OperationContext {
  disableErrorNotification?: boolean;
}

function globalErrorHandler(error: ErrorResponse) {
  if (snackbarDispatcher) {
    const { disableErrorNotification } = error.operation.getContext() as OperationContext;

    if (!disableErrorNotification) {
      getNotifications(error).forEach((notification) => snackbarDispatcher.dispatch(notification));
    }
  }
}

// This identity function is used for context typing
function context(context: OperationContext): OperationContext {
  return context;
}
```

More fine-grained control can be achieved by:

- using a callback accepting the `ErrorResponse` object and returning a `boolean`;
- using something like `{network?: boolean, gql?: Record<GqlError, boolean>}` instead of `boolean` (we can pass it to `getNotifications` and filter out notifications that should be ignored).

Unfortunately, there doesn’t seem to be a way to properly type operation context. Usually, it can be done via [Declaration Merging](https://www.typescriptlang.org/docs/handbook/declaration-merging.html), but Apollo types are written in such a way that it’s not possible. One way to mitigate that is to use an identity function for context creation.

## Conclusion

Proper error handling in Apollo is a relatively complex task, which requires effort from both backend and frontend teams. Some aspects are open for debate, especially the “where to put errors” problem. Apollo also has some bugs and implementation quirks that make the situation even worse. Even the GraphQL specification itself doesn’t make it easy.

The provided solution is not by any means definitive, but it should be general enough to cover a large number of cases.
