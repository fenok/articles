# Server-Side Data Fetching in React

This is my understanding of what methods are available for server-side data fetching in React today, what their pros and cons are, how they play with client-side data fetching methods, and how they relate to upcoming Suspense for Data Fetching.

## The methods

In React, there are the following fetching strategies:

-   **Fetch-on-Render**: fetching is a result of render. On server side, fetching is triggered at the render phase, because side effects are ignored. On client side, it's better to trigger fetching asynchronously _after_ the render phase via the `useEffect` hook. This way, fetching can be started without blocking rendering.
-   **Fetch-Then-Render**: trigger fetching, wait for its completion, and only then start rendering. This way, when a component renders, its data is already fetched. We can either delay the (initial) render of the whole app, or the render of its part, most likely a page content. In the latter case, the page component can actually use a different method, like **Fetch-on-Render**, to prepare data for its content.
-   **Render-as-You-Fetch**: basically the same as **Fetch-Then-Render**, but we start rendering while fetching is still in progress. This method is ideal for Suspense for Data Fetching, but, from my understanding, it can be used without Suspense as well. **Render-as-You-Fetch** is practically useless on server side without Suspense, because, as of React 17, the render phase is synchronous. Therefore, it's more efficient to wait for the data before rendering (in other words, use **Fetch-Then-Render**). However, on client side, **Render-as-You-Fetch** is better for UX, because we can show components in individual loading states instead of a plain preloader above them all.

It goes without saying that fetching strategies can differ between client and server environments, and even between different application parts. For instance, consider React Query in conjunction with Next.js. On server side, **Fetch-Then-Render** is used, because that's how Next.js operates (and React Query is unopinionated about SSR). On client side, **Fetch-on-Render** is used, because that's how React Query works. Additionally, on client side, React Query allows starting fetching early via [Prefetching](https://react-query.tanstack.com/guides/prefetching), which basically enables the **Render-as-You-Fetch** method even without Suspense. Finally, on client side, it's possible to move all page queries to the page component and render the page content only when all data arrives. This way, the page content will effectively use the **Fetch-Then-Render** method.

The following examples give a rough idea of what the **Fetch-on-Render** and **Fetch-Then-Render** methods look like both on server and client sides. **Render-as-You-Fetch** is not shown, because, again, it's just **Fetch-Then-Render**, but we render while fetching is still in progress.

### Fetch-on-Render

```typescript jsx
// Server-side part. Express middleware.
async function ssrMiddleware(_, res) {
    /** Request-specific store for our data. */
    const store = {};

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
            <script>window.STORE=${JSON.stringify(store)}</script>
            <script src="bundle.js"></script>
        </body`
    );
}

/**
 * Client-side part. Hydrate the recieved markup with the store from
 * SSR.
 */
hydrate(
    createElement(App, { store: window.STORE }),
    document.getElementById("root")
);

/** Isomorphic App component. */
const App = ({ store }) => {
    const user = useQuery(store, "user", fetchUser);

    return <div>{user ? user.name : "Loading..."}</div>;
};

/** Hook for fetching on both client and server sides. */
function useQuery(store, fieldName, fetchFn) {
    /**
     * Server-side only helper provided by the getDataFromTree
     * utility.
     */
    const ssrManager = useSsrManager();

    /** Helper for forcing component update. */
    const forceUpdate = useForceUpdate();

    /**
     * If no data on server side, fetch it and register the
     * promise.
     */
    if (ssrManager && !store[fieldName]) {
        ssrManager.add(
            fetchFn().then((data) => {
                store[fieldName] = data;
            })
        );
    }

    /**
     * If no data on client side, fetch it and force component update
     * to display it.
     */
    useEffect(() => {
        const cancelled = false;

        if (!store[fieldName]) {
            fetchFn().then((data) => {
                if (!cancelled) {
                    store[fieldName] = data;
                    forceUpdate();
                }
            });
        }

        return () => {
            cancelled = true;
        };
    });

    return store[fieldName];
}
```

### Fetch-Then-Render

```typescript jsx
/** Server-side part. Express middleware. */
async function ssrMiddleware(_, res) {
    /** Request-specific store for our data. */
    const store = {};

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
            <script>window.STORE=${JSON.stringify(store)}</script>
            <script src="bundle.js"></script>
        </body`
    );
}

/**
 * Client-side part. Hydrate the recieved markup with the store from
 * SSR, enriched by cleint-side prefetch.
 */
hydrate(
    createElement(App, { store: await App.prefetch(window.STORE) }),
    document.getElementById("root")
);

/** Isomorphic App component. */
const App = ({ store }) => {
    const user = useRefetchingQuery(store, "user", fetchUser);

    return <div>{user ? user.name : "Loading..."}</div>;
};

/** Function for initial fetching. */
App.prefetch = async (store) => {
    if (!store.user) {
        /** We explicitly prefetch some data. */
        store.user = await fetchUser();
    }

    return store;
};

/**
 * Hook for refetching by timer on client side.
 * Illustrates that we still might need to trigger fetching in response
 * to user actions or other events.
 */
function useRefetchingQuery(store, fieldName, fetchFn) {
    /** Helper for forcing component update. */
    const forceUpdate = useForceUpdate();

    useEffect(() => {
        const cancelled = false;

        const timeoutId = window.setTimeout(() => {
            fetchFn().then((data) => {
                if (!cancelled) {
                    store[fieldName] = data;
                    forceUpdate();
                }
            });
        }, 10000);

        return () => {
            window.clearTimeout(timeoutId);
            cancelled = true;
        };
    }, []);

    return store[fieldName];
}
```

## Fetch-on-Render vs Fetch-Then-Render

### Fetching start time

As you can see, **Fetch-Then-Render** allows fetching to be started earlier, because the request doesn't wait for render to kick it off.

### Fetching waterfalls

Fetching waterfalls are situations where requests are unintentionally made sequential, while they should have been parallelized.

**Fetch-on-Render** makes it easy to create such waterfalls, because the requests are decentralized. Some parent can fetch its data, then pass this data to its newly-rendered child, which itself can trigger a request that doesn't use the passed data at all, and viola: we got ourselves a waterfall.

**Fetch-Then-Render**, on the other side, forces the requests to be centralized (most likely on a per-page basis), thereby decreasing the risk of creating these waterfalls.

### Number of server-side renders

As of React 17, the app can only be rendered synchronously.

In case of **Fetch-Then-Render**, it's not a problem. Since the requests are centralized, we can simply wait for them all and then render the app only once.

**Fetch-on-Render**, however, forces us to render the app an unknown number of times. The idea is to render the app, wait for started requests and repeat until there are no more promises to wait for. If it sounds inefficient and non-production-ready, don't you worry: it's exactly what Apollo [does](https://github.com/apollographql/apollo-client/blob/da4e9b95dcf11328cc568a5518151fb80de8f8df/src/react/ssr/getDataFromTree.ts#L52).

With React 18 Suspense for Data Fetching either approach will result in single render. **Fetch-Then-Render** will be transformed into **Render-as-You-Fetch**, meaning that we won't wait for the fetching to finish before rendering. **Fetch-on-Render** will likely be a perfectly valid option [too](https://github.com/reactwg/react-18/discussions/35#discussioncomment-823980).

### Fetching logic encapsulation

**Fetch-on-Render** allows to encapsulate both client- and server-side code in a single hook.

**Fetch-Then-Render**, however, forces us to split the fetching logic. There is initial fetching before render and outside of React, which can happen both server- and client-side, and there is a client-side-only fetching in response to user actions (or other events). In the latter case, the fetching is still happening before render, but it most likely resides within React.

### Access to React-specific data

In case of **Fetch-on-Render**, everything is happening inside React. It means that the fetching code has access to props (we most likely care about URL params), and we can guarantee that we're fetching the data for the right page.

**Fetch-Then-Render** is a bit more complicated. The initial fetching happens outside of React. It means that we have to do some extra work to determine which page we're on and what the URL params are.

The events-driven fetching, however, most likely resides within React and have access to props and everything else.

## To summarize

### Render methods comparison

|                                                  | Fetch-Then-Render / Render-as-You-Fetch                                                                                                                                                                                        | Fetch-on-Render                                                                                                |
| ------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------- |
| Fetching start time                              | ✔️ Fetching is started as soon as possible.                                                                                                                                                                                    | ❌ Fetching is delayed until render.                                                                           |
| Fetching waterfalls                              | ✔️ Waterfalls can only be created explicitly.                                                                                                                                                                                  | ❌ It's possible to accidentally create a waterfall.                                                           |
| Number of server-side renders (without Suspense) | ✔️ Single render (**Render-as-You-Fetch** is useless in this case and shouldn't be used).                                                                                                                                      | ❌ Multiple renders required to guarantee that all data is fetched.                                            |
| Compatibility with Suspense for Data Fetching    | ✔️The recommended approach is **Render-as-You-Fetch** (for client and server sides). **Fetch-Then-Render** shouldn't be used.                                                                                                  | ✔️Will likely be supported [too](https://github.com/reactwg/react-18/discussions/35#discussioncomment-823980). |
| Fetching logic encapsulation                     | ❌ Fetching is split in two pieces of code: initial fetching before render, outside of React (server-side and _maybe_ client-side), and fetching in response to user actions or other events, inside React (client-side only). | ✔️ It's easy to encapsulate both client- and server-side code in a single hook.                                |
| Access to React-specific data                    | ❌ Initial fetching is done outside of React. There is no access to props, so there has to be a separate URL parser, for example.                                                                                              | ✔️ Since fetching is always done during render, there is access to props and everything else.                  |

### What methods libraries use

|        | Fetch-Then-Render                                                                                                                                                                 | Render-as-You-Fetch                                                                                                                                                                                             | Fetch-on-Render                                |
| ------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------- |
| Client | Apollo (code organization), React Query (code organization). It's possible to move all page queries to the page component and render the page content only when all data arrives. | Apollo (optional [Prefetching](https://www.apollographql.com/docs/react/performance/performance/#prefetching-data)), React Query (optional [Prefetching](https://react-query.tanstack.com/guides/prefetching)). | Apollo (by default), React Query (by default). |
| Server | Next.js, Apollo (if you choose to manually execute queries during SSR).                                                                                                           | **Fetch-Then-Render** yields the same result more efficiently.                                                                                                                                                  | Apollo (`getDataFromTree`).                    |
