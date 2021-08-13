![Cover](resources/cover.png)

# Client- and Server-Side Data Fetching in React

This is an overview of client- and server-side data fetching approaches in React 17, their pros and cons, and the way upcoming [Suspense for Data Fetching](https://reactjs.org/docs/concurrent-mode-suspense.html) will change them.

## So how do we fetch?

React supports the following fetching approaches:

-   **Fetch-on-Render**: fetching is triggered by render.
-   **Fetch-Then-Render**: we start fetching as early as possible and render only when the data is ready.
-   **Render-as-You-Fetch**: we start fetching as early as possible and then start rendering _immediately_, without waiting for the data to be ready. In a sense, **Fetch-Then-Render** is a special case of **Render-as-You-Fetch**.

It goes without saying that the fetching approaches can differ between client and server environments, and even between different parts of an application. For instance, consider how [Apollo](https://www.apollographql.com/) works.

On the server side, if we use [`getDataFromTree`](https://www.apollographql.com/docs/react/api/react/ssr/#getdatafromtree), we implement **Fetch-on-Render**, because we render the app to trigger fetching. Or, we can use [Prefetching](https://www.apollographql.com/docs/react/performance/performance/#prefetching-data) instead and get either **Fetch-Then-Render** or **Render-as-You-Fetch**, depending on when we start rendering.

On the client side, **Fetch-on-Render** is the default approach, because that's how the [`useQuery`](https://www.apollographql.com/docs/react/api/react/hooks/#usequery) hook works. We can also use [Prefetching](https://www.apollographql.com/docs/react/performance/performance/#prefetching-data) and essentially get **Render-as-You-Fetch**.

Finally, on the client side, we can delay the initial render until [Prefetching](https://www.apollographql.com/docs/react/performance/performance/#prefetching-data) is complete to implement **Fetch-Then-Render**, but it's likely not a very good idea.

In fact, we can mix the fetching approaches. For instance, on the client side, we can move all page queries to the page component and render its content only when all data arrives. This way, the page content will effectively use the **Fetch-Then-Render** approach, though the page component itself will use either **Fetch-on-Render** or **Render-as-You-Fetch**.

Throughout the article, we will focus on "pure" forms of the fetching approaches.

## Show me the code!

The following examples give a rough idea of what the fetching approaches look like on both the server and the client sides (as of React 17).

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

/** A hook for all fetching logic. */
function useQuery(store, fieldName, fetchFn) {
    /** Server-side-only helper from the getDataFromTree utility. */
    const ssrManager = useSsrManager();

    /**
     * If no data on the server side, fetch it and register the
     * promise.
     * We do it at the render phase, because side effects are
     * ignored on the server side.
     */
    if (ssrManager && !store.has(fieldName)) {
        ssrManager.add(
            fetchFn().then((data) => store.set(fieldName, data))
        );
    }

    /**
     * If no data on the client side, fetch it.
     * We do it in a passive effect, so render isn't blocked.
     */
    useEffect(() => {
        if (!store.has(fieldName)) {
            fetchFn().then((data) => store.set(fieldName, data));
        }
    });

    /** Subscribe to a store part. */
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

    const app = createElement(App, { store });

    /**
     * Fill the store with data.
     * Server-side fetching can be disabled.
     */
    if (process.env.PREFETCH) {
        await App.prefetch(store);
    }

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

/** A function for initial fetching. */
App.prefetch = async (store) => {
    if (!store.has("user")) {
        /** We explicitly prefetch some data. */
        store.set("user", await fetchUser());
    }

    return store;
};

/** A hook for fetching in response to a user action. */
function useQuery(store, fieldName, fetchFn) {
    /** Subscribe to a store part. */
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

    const app = createElement(App, { store });

    /**
     * Fill the store with data.
     * Server-side fetching can be disabled.
     */
    if (process.env.PREFETCH) {
        const prefetchPromise = App.prefetch(store);

        /** We "render-as-we-fetch", but it's completely useless. */
        renderToString(app);

        await prefetchPromise;
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

/** A function for initial fetching. */
App.prefetch = async (store) => {
    if (!store.has("user")) {
        /** We explicitly prefetch some data. */
        store.set("user", await fetchUser());
    }

    return store;
};

/** A hook for fetching in response to a user action. */
function useQuery(store, fieldName, fetchFn) {
    /** Subscribe to a store part. */
    const data = useStoreValue(store, fieldName);

    const refetch = () =>
        fetchFn().then((data) => store.set(fieldName, data));

    return [data, refetch];
}
```

## Fetch-on-Render vs Fetch-Then-Render vs Render-as-You-Fetch

### Fetching start time

As you can see, **Fetch-Then-Render** and **Render-as-You-Fetch** make it possible to start fetching earlier, because the requests don't wait for the render to kick them off.

### Rendering without data

**Fetch-Then-Render** is simple: a component will never be rendered without its data.

With **Fetch-on-Render** or **Render-as-You-Fetch**, however, the data can arrive after the render, so the component has to be able to display some "no-data" state.

### Fetching waterfalls

Fetching waterfalls are situations where requests that should have been parallelized are unintentionally made sequential.

**Fetch-on-Render** makes it easy to create such waterfalls, because the requests are decentralized. Some parent can fetch its data, then pass this data to its newly-rendered child, which itself can trigger a request that doesn't use the passed data at all. That's a clear waterfall.

**Fetch-Then-Render**, on the other hand, forces the requests to be centralized (most likely on a per-page basis), thereby eliminating the risk of creating these waterfalls. However, now that we've grouped all the requests into a single promise, we therefore have to wait for all of them to complete before we can render, which is not ideal.

**Render-as-You-Fetch** also forces the requests to be centralized, but, since render is not delayed, we can show pieces of data as they arrive.

### Number of server-side renders

As of React 17, we can't wait for data during render.

For **Fetch-Then-Render**, it's not a problem. Since the requests are centralized, we can simply wait for them all and then render the app only once.

**Fetch-on-Render**, however, forces us to render the app _at least_ two times. The idea is to render the app, wait for all the initiated requests to complete, and then repeat the process until there are no more requests to wait for. If it seems inefficient and not ready for production, don't you worry: this approach has long been [used](https://github.com/apollographql/apollo-client/blob/da4e9b95dcf11328cc568a5518151fb80de8f8df/src/react/ssr/getDataFromTree.ts#L52) by Apollo.

**Render-as-You-Fetch** is very similar to **Fetch-Then-Render**, but slightly less efficient (it requires two renders, one of which is useless). In fact, it shouldn't be used on the server side at all.

### Encapsulation of the fetching logic

With **Fetch-on-Render**, it's easy to encapsulate both client- and server-side code in a single hook.

In contrast, **Fetch-Then-Render** and **Render-as-You-Fetch** force us to split the fetching logic. On one hand, there is the initial fetching. It occurs before render (outside of React), and it can happen on both the server and the client sides. On the other hand, there is the client-side-only fetching in response to user actions (or other events), which still happens before render, but most likely resides within React.

### Access to React-specific data

In case of **Fetch-on-Render**, everything happens inside React. It means that the fetching code has access to props (we most likely care about URL params), and we are guaranteed to always fetch the data for the right page.

**Fetch-Then-Render** and **Render-as-You-Fetch** are a bit more complicated. The initial fetching happens outside of React. Hence, we have to do some extra work to determine which page we're on and what the URL params are.

The event-driven fetching, however, usually resides within React and has access to props and everything else.

## What will change in React 18?

React 18 will support [Suspense for Data Fetching](https://reactjs.org/docs/concurrent-mode-suspense.html).

With the [recommended API](https://github.com/reactwg/react-18/discussions/22#discussion-3385743), either fetching approach will result in a single render on the server side (in the sense that we won't discard previously rendered parts).

With Suspense in general, we will render a component only if its data is ready, because otherwise the component will suspend, and we will try again when the data is ready.

All other mentioned pros and cons will remain the same.

As you can see, **Render-as-You-Fetch** will work equally well on both the server and the client sides, and it will completely replace **Fetch-Then-Render**, because the latter just won't have any advantages left.

**Fetch-on-Render** will remain [available](https://github.com/reactwg/react-18/discussions/35#discussioncomment-823980) as a more convenient (though less efficient) alternative.

## Summary

|                                             | Fetch-on-Render                                        | Fetch-Then-Render                                         | Render-as-You-Fetch                                         |
| ------------------------------------------- | ------------------------------------------------------ | --------------------------------------------------------- | ----------------------------------------------------------- |
| Fetching start time                         | ❌ Fetching is delayed until render                    | ✔️ Fetching is started as soon as possible                | ✔️ Fetching is started as soon as possible                  |
| Rendering without data (no Suspense)        | ❌ Always                                              | ✔️ Never                                                  | ❌ Sometimes                                                |
| Rendering without data (Suspense)           | ✔️ Never                                               | ⚠️ It's completely replaced by **Render-as-You-Fetch**    | ✔️ Never                                                    |
| Fetching waterfalls                         | ❌ Implicit waterfalls, but we show data independently | ❌ Only explicit waterfalls, but we show "all or nothing" | ✔️ Only explicit waterfalls, and we show data independently |
| Number of server-side renders (no Suspense) | ❌ At least two renders                                | ✔️ A single render                                        | ❌ Two renders, one of which is useless                     |
| Number of server-side renders (Suspense)    | ✔️ A single render                                     | ⚠️ It's completely replaced by **Render-as-You-Fetch**    | ✔️ A single render                                          |
| Encapsulation of the fetching logic         | ✔️ Yes                                                 | ❌ No                                                     | ❌ No                                                       |
| Access to React-specific data               | ✔️ Yes                                                 | ❌ The initial fetching is done outside of React          | ❌ The initial fetching is done outside of React            |
| Usage with Suspense for Data Fetching       | ✔️ It's less efficient but more convenient             | ⚠️ It's completely replaced by **Render-as-You-Fetch**    | ✔️It's the recommended approach                             |
