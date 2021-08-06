# Server-Side Data Fetching in React

This is my understanding of what methods are available for server-side data fetching in React today, what their pros and cons are, how they play with client-side data fetching methods, and how they relate to upcoming Suspense for Data Fetching.

## The methods

In React, there are the following fetching strategies:

-   **Fetch-on-Render**: we start render, which starts fetching at some point.
-   **Fetch-Then-Render**: we start fetching before render, wait for its completion and start render.
-   **Render-as-You-Fetch**: we start fetching and then, _not necessarily after its completion_, we start render. In a sense, **Fetch-Then-Render** is a special case of **Render-as-You-Fetch**.

It goes without saying that fetching strategies can differ between client and server environments, and even between different application parts. For instance, consider how [Apollo](https://www.apollographql.com/) works.

On server side, if we use [`getDataFromTree`](https://www.apollographql.com/docs/react/api/react/ssr/#getdatafromtree), we get **Fetch-on-Render**, because we render the app to trigger fetching. Or we can use [Prefetching](https://www.apollographql.com/docs/react/performance/performance/#prefetching-data) instead and get either **Fetch-Then-Render** or **Render-as-You-Fetch**, depending on when we start render.

On client side, we get **Fetch-on-Render** by default, because that's how the [`useQuery`](https://www.apollographql.com/docs/react/api/react/hooks/#usequery) hook works. We can also use [Prefetching](https://www.apollographql.com/docs/react/performance/performance/#prefetching-data) and essentially get **Render-as-You-Fetch**.

Finally, on client side, it's possible to move all page queries to the page component and render the page content only when all data arrives. This way, the page content will effectively use the **Fetch-Then-Render** method (though the page component itself will use either **Fetch-on-Render** or **Render-as-You-Fetch**). Sure enough, you can also delay the initial app render and get pure **Fetch-Then-Render**, if you feel creative.

## Examples

The following examples give a rough idea of what the fetching methods look like both on server and client sides (as of React 17).

### Fetch-on-Render

```typescript jsx
/** Server-side part. Express middleware. */
async function ssrMiddleware(_, res) {
    /** Request-specific store for our data. */
    const store = createStore();

    const app = createElement(App, { store });

    /**
     * Render the app (possibly multiple times) and wait for
     * registered promises.
     * Server-side fetching can be disabled.
     */
    if (process.env.PREFETCH) {
        await getDataFromTree(app);
    }

    /**
     * Render the final variant of the app and send it alongside the
     * store.
     */
    res.send(
        `<!doctype html>
        <body>
            <div id="root">${renderToString(app)}</div>
            <script>window.STORE=${JSON.stringify(
                store.extract()
            )}</script>
            <script src="bundle.js"></script>
        </body`
    );
}

/**
 * Client-side part. Hydrate the received markup with the store from
 * SSR.
 */
hydrate(
    createElement(App, { store: createStore(window.STORE) }),
    document.getElementById("root")
);

/** Isomorphic App component. */
const App = ({ store }) => {
    const [user, refetch] = useQuery(store, "user", fetchUser);

    return (
        <div>
            {user ? user.name : "Loading..."}
            <button onClick={refetch}>Refetch</button>
        </div>
    );
};

/** Hook for all fetching logic. */
function useQuery(store, fieldName, fetchFn) {
    /** Server-side only helper from the getDataFromTree utility. */
    const ssrManager = useSsrManager();

    /**
     * If no data on server side, fetch it and register the
     * promise.
     * We do it at the render phase, because side effects are
     * ignored on server side.
     */
    if (ssrManager && !store.has(fieldName)) {
        ssrManager.add(
            fetchFn().then((data) => store.set(fieldName, data))
        );
    }

    /**
     * If no data on client side, fetch it.
     * We do it in a passive effect, so render isn't blocked.
     */
    useEffect(() => {
        if (!store.has(fieldName)) {
            fetchFn().then((data) => store.set(fieldName, data));
        }
    });

    /** Subscribe to the store part. */
    const data = useStoreValue(store, fieldName);

    const refetch = () =>
        fetchFn().then((data) => store.set(fieldName, data));

    return [data, refetch];
}
```

### Fetch-Then-Render

```typescript jsx
/** Server-side part. Express middleware. */
async function ssrMiddleware(_, res) {
    /** Request-specific store for our data. */
    const store = createStore();

    /**
     * Fill the store with data.
     * Server-side fetching can be disabled.
     */
    if (process.env.PREFETCH) {
        await App.prefetch(store);
    }

    const app = createElement(App, { store });

    /**
     * Render the first and final variant of the app and send it
     * alongside the store.
     */
    res.send(
        `<!doctype html>
        <body>
            <div id="root">${renderToString(app)}</div>
            <script>window.STORE=${JSON.stringify(
                store.extract()
            )}</script>
            <script src="bundle.js"></script>
        </body`
    );
}

/**
 * Client-side part. Hydrate the received markup with the store from
 * SSR, enriched by cleint-side initial fetching.
 */
hydrate(
    createElement(App, {
        store: await App.prefetch(createStore(window.STORE)),
    }),
    document.getElementById("root")
);

/** Isomorphic App component. */
const App = ({ store }) => {
    const [user, refetch] = useQuery(store, "user", fetchUser);

    return (
        <div>
            {user ? user.name : "Loading..."}
            <button onClick={refetch}>Refetch</button>
        </div>
    );
};

/** Function for initial fetching. */
App.prefetch = async (store) => {
    if (!store.has("user")) {
        /** We explicitly prefetch some data. */
        store.set("user", await fetchUser());
    }

    return store;
};

/** Hook for fetching in response to user action. */
function useQuery(store, fieldName, fetchFn) {
    /** Subscribe to the store part. */
    const data = useStoreValue(store, fieldName);

    const refetch = () =>
        fetchFn().then((data) => store.set(fieldName, data));

    return [data, refetch];
}
```

### Render-as-You-Fetch

```typescript jsx
/** Server-side part. Express middleware. */
async function ssrMiddleware(_, res) {
    /** Request-specific store for our data. */
    const store = createStore();

    /**
     * Fill the store with data.
     * Server-side fetching can be disabled.
     */
    if (process.env.PREFETCH) {
        const prefetchPromise = App.prefetch(store);

        /** We 'render-as-we-fetch', but it's completely useless */
        renderToString(app);

        await prefetchPromise;
    }

    const app = createElement(App, { store });

    /**
     * Render the final variant of the app and send it alongside the
     * store.
     */
    res.send(
        `<!doctype html>
        <body>
            <div id="root">${renderToString(app)}</div>
            <script>window.STORE=${JSON.stringify(
                store.extract()
            )}</script>
            <script src="bundle.js"></script>
        </body`
    );
}

/**
 * Client-side part. Start client-side initial fetching and immediately
 * hydrate the received markup with the store from SSR.
 */
const store = createStore(window.STORE);
App.prefetch(store);
hydrate(createElement(App, { store }), document.getElementById("root"));

/** Isomorphic App component. */
const App = ({ store }) => {
    const [user, refetch] = useQuery(store, "user", fetchUser);

    return (
        <div>
            {user ? user.name : "Loading..."}
            <button onClick={refetch}>Refetch</button>
        </div>
    );
};

/** Function for initial fetching. */
App.prefetch = async (store) => {
    if (!store.has("user")) {
        /** We explicitly prefetch some data. */
        store.set("user", await fetchUser());
    }

    return store;
};

/** Hook for fetching in response to user action. */
function useQuery(store, fieldName, fetchFn) {
    /** Subscribe to the store part. */
    const data = useStoreValue(store, fieldName);

    const refetch = () =>
        fetchFn().then((data) => store.set(fieldName, data));

    return [data, refetch];
}
```

## Fetch-on-Render vs Fetch-Then-Render vs Render-as-You-Fetch

### Fetching start time

As you can see, **Fetch-Then-Render** and **Render-as-You-Fetch** allow fetching to be started earlier, because the request doesn't wait for render to kick it off.

### Fetching waterfalls

Fetching waterfalls are situations where requests are unintentionally made sequential, while they should have been parallelized.

**Fetch-on-Render** makes it easy to create such waterfalls, because the requests are decentralized. Some parent can fetch its data, then pass this data to its newly-rendered child, which itself can trigger a request that doesn't use the passed data at all, and viola: we got ourselves a waterfall.

**Fetch-Then-Render**, on the other side, forces the requests to be centralized (most likely on a per-page basis), thereby decreasing the risk of creating these waterfalls. However, if we group all requests into a single promise, we still wait for all requests to complete before we can start render, which is not ideal.

**Render-as-You-Fetch** also forces the requests to be centralized, but, since render is not delayed, we can show pieces of data as they arrive.

### Number of server-side renders

As of React 17, the app can only be rendered synchronously.

In case of **Fetch-Then-Render**, it's not a problem. Since the requests are centralized, we can simply wait for them all and then render the app only once.

**Fetch-on-Render**, however, forces us to render the app _at least_ two times. The idea is to render the app, wait for started requests and repeat until there are no more promises to wait for. If it sounds inefficient and non-production-ready, don't you worry: it's exactly what Apollo [does](https://github.com/apollographql/apollo-client/blob/da4e9b95dcf11328cc568a5518151fb80de8f8df/src/react/ssr/getDataFromTree.ts#L52).

**Render-as-You-Fetch** is the same as **Fetch-Then-Render**, but less efficient (it requires two renders, one of which is useless).

### Fetching logic encapsulation

**Fetch-on-Render** allows to encapsulate both client- and server-side code in a single hook.

**Fetch-Then-Render**, however, as well as **Render-as-You-Fetch**, forces us to split the fetching logic. There is initial fetching before render and outside of React, which can happen both server- and client-side, and there is a client-side-only fetching in response to user actions (or other events). In the latter case, the fetching is still happening before render, but it most likely resides within React.

### Access to React-specific data

In case of **Fetch-on-Render**, everything is happening inside React. It means that the fetching code has access to props (we most likely care about URL params), and we can guarantee that we're fetching the data for the right page.

**Fetch-Then-Render**, as well as **Render-as-You-Fetch**, is a bit more complicated. The initial fetching happens outside of React. It means that we have to do some extra work to determine which page we're on and what the URL params are.

The events-driven fetching, however, most likely resides within React and have access to props and everything else.

## Suspense for Data Fetching

As of React 17, the render phase is synchronous. React 18 will support Suspense for Data Fetching, which will be based on _asynchronous_ rendering.

With Suspense, **Fetch-Then-Render** will be completely replaced by **Render-as-You-Fetch**, which will work equally well on both server and client sides.

**Fetch-on-Render** will also be [available](https://github.com/reactwg/react-18/discussions/35#discussioncomment-823980) as a more convenient (though less efficient) alternative.

Thanks to asynchronous rendering, either method will result in a single render in any environment (we're not counting intermediate rerenders of suspended components).

All other mentioned pros and cons will remain the same.

## To summarize

|                                                  | Fetch-on-Render                                                  | Fetch-Then-Render                                                                                           | Render-as-You-Fetch                                                                  |
| ------------------------------------------------ | ---------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------ |
| Fetching start time                              | ❌ Fetching is delayed until render.                             | ✔️ Fetching is started as soon as possible.                                                                 | ✔️ Fetching is started as soon as possible                                           |
| Fetching waterfalls                              | ❌ It's possible to accidentally create a waterfall.             | ✔️❌ No waterfalls between components, but we likely won't show any data before all requests are completed. | ✔️ Waterfalls can only be created explicitly.                                        |
| Number of server-side renders (without Suspense) | ❌ At least two renders.                                         | ✔️ Single render.                                                                                           | ❌ Two renders, one of which is useless.                                             |
| Compatibility with Suspense for Data Fetching    | ✔️Less efficient, but more convenient and perfectly valid.       | ❌ It's completely replaced by **Render-as-You-Fetch**.                                                     | ✔️It's the recommended approach.                                                     |
| Fetching logic encapsulation                     | ✔️ It's easy to encapsulate all fetching logic in a single hook. | ❌ Fetching logic is split into initial fetching and fetching in response to events.                        | ❌ Fetching logic is split into initial fetching and fetching in response to events. |
| Access to React-specific data                    | ✔️ Fetching is always done inside React.                         | ❌ Initial fetching is done outside of React.                                                               | ❌ Initial fetching is done outside of React.                                        |
