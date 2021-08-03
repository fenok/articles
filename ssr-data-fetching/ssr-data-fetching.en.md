# Server-Side Data Fetching in React

## TL;DR

|                                                      | Fetch-Then-Render                                                                                                             | Fetch-on-Render                                                                                                                                                                                   |
| ---------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Fetching start time                                  | ✔️ Fetching is started as soon as possible.                                                                                   | ❌ Fetching is delayed until render.                                                                                                                                                              |
| Fetching waterfalls                                  | ✔️ No fetching waterfalls.                                                                                                    | ❌ Waterfalls are very possible.                                                                                                                                                                  |
| Number of server-side renders (without Suspense)     | ✔️ Only one render on the server side.                                                                                        | ❌ Multiple renders required to guarantee that all data is fetched.                                                                                                                               |
| Compatibility with React 18 Suspense (single render) | ✔️It's the recommended approach.                                                                                              | ✔️Will likely be supported too, but there have to be a [way](https://github.com/reactwg/react-18/discussions/35#discussioncomment-823980) to preserve requests between suspense-caused rerenders. |
| Isomorphism                                          | ❌ Server-side fetching logic is separate from the client-side one.                                                           | ✔️ The fetching code is isomorphic, there is no SSR-specific code.                                                                                                                                |
| Access to React-specific data                        | ❌ SSR fetching is done outside of React. There is no access to props, so there has to be a separate URL parser, for example. | ✔️ Since the fetching is done during render, there is access to props and everything else.                                                                                                        |
| Used by                                              | Next.js.                                                                                                                      | Apollo.                                                                                                                                                                                           |

## Introduction

The main problem during SSR is to know when all the data is fetched. In React, the data can be fetched either during the render phase, or prior to it. Fetching after the render phase is impossible (and makes little sense), since the server only cares about rendering and not about committing, executing side effects, etc.

### Fetch-on-Render (oversimplified) example

```typescript jsx
// Server-side, express middleware
async function ssrMiddleware(_, res) {
    // Request-specific store for our data
    const store = {};

    const app = createElement(App, { store });

    // We can disable data prefetch to illustrate client-side fetching
    if (process.env.PREFETCH) {
        // This helper renders the app over and over and waits for returned promises.
        // Resolves when no more promises returned.
        // Which means that all requests were made and all data is in store.
        await getDataFromTree(app);
    }

    // Store is sent with the HTML
    res.send(
        `<!doctype html>
        <body>
            <div id="root">${renderToString(app)}</div>
            <script>window.STORE=${JSON.stringify(store)}</script>
            <script src="bundle.js"></script>
        </body`
    );
}

// Client-side, pickup store from SSR
hydrate(createElement(App, { store: window.STORE }), document.getElementById("root"));

// Isomorphic App
const App = ({ store }) => {
    const user = useQuery(store, "user", fetchUser);

    return <div>{user ? user.name : "Loading..."}</div>;
};

// Helper for fetching data, both client- and server-side
function useQuery(store, fieldName, fetchFn) {
    // Provided by the getDataFromTree() helper via React Context
    // Non-existent on the client side
    const ssrManager = useSsrManager();

    // Helper for forcing component update.
    const forceUpdate = useForceUpdate();

    if (ssrManager && !store[fieldName]) {
        // If server-side and no data, fetch it and add the promise to the manager.
        // This way, the getDataFromTree() can wait for the request to complete.
        ssrManager.add(
            fetchFn.then((data) => {
                store[fieldName] = data;
            })
        );
    }

    // Client-side only.
    // If no data, fetch it and force component update to display it.
    // There will be no data, if the getDataFromTree() helper wasn't used.
    useEffect(() => {
        if (!store[fieldName]) {
            fetchFn.then((data) => {
                store[fieldName] = data;
                forceUpdate();
            });
        }
    });

    return store[fieldName];
}
```

### Fetch-Then-Render (oversimplified) example

```typescript jsx
// Server-side, express middleware
async function ssrMiddleware(_, res) {
    // We can disable data prefetch to illustrate client-side fetching
    const store = process.env.PREFETCH ? await App.prefetch() : {};

    const app = createElement(App, { store });

    // Store is sent with the HTML
    res.send(
        `<!doctype html>
        <body>
            <div id="root">${renderToString(app)}</div>
            <script>window.STORE=${JSON.stringify(store)}</script>
            <script src="bundle.js"></script>
        </body`
    );
}

// Client-side, pickup store from SSR
hydrate(createElement(App, { store: window.STORE }), document.getElementById("root"));

// Isomorphic App
const App = ({ store }) => {
    const user = useQuery(store, "user", fetchUser);

    return <div>{user ? user.name : "Loading..."}</div>;
};

App.prefetch = async () => {
    // Request-specific store for our data
    const store = {};

    // We explicitly prefetch some data.
    store.user = await fetchUser();

    return store;
};

// Helper for fetching data, client-side only
function useQuery(store, fieldName, fetchFn) {
    // Helper for forcing component update.
    const forceUpdate = useForceUpdate();

    // Client-side only.
    // If no data, fetch it and force component update to display it.
    useEffect(() => {
        if (!store[fieldName]) {
            fetchFn.then((data) => {
                store[fieldName] = data;
                forceUpdate();
            });
        }
    });

    return store[fieldName];
}
```

## Fetching start time

As you can see, Fetch-Then-Render allows fetching to be started earlier, because the request doesn't wait for render to kick it off (hence the name).

## Fetching waterfalls

Fetching waterfalls are situations where requests are unintentionally made sequential, while they should have been parallelized.

Fetch-on-Render makes it easy to create such waterfalls, because the requests are decentralized. Some parent can fetch its data, then pass this data to its newly-rendered child, which itself can trigger a request that doesn't use the passed data at all, and viola: we got ourselves a waterfall.

Fetch-Then-Render, on the other side, forces the requests to be centralized (most likely on a per-page basis), thereby decreasing the risk of creating these waterfalls.

## Number of server-side renders

As of React 17, the app can only be rendered synchronously.

In case of Fetch-Then-Render, it's not a problem. Since the requests are centralized, we can simply wait for them all and then render the app only once.

Fetch-on-Render, however, forces us to render the app an unknown number of times. The idea is to render the app, wait for started requests and repeat until there are no more promises to wait for. If it sounds inefficient and non-production-ready, don't you worry: it's exactly what Apollo [does](https://github.com/apollographql/apollo-client/blob/da4e9b95dcf11328cc568a5518151fb80de8f8df/src/react/ssr/getDataFromTree.ts#L52).

## Isomorphism

It's easy to see that Fetch-on-Render is isomorphic: the `useQuery` hook hides all SSR-specific logic, and the `App` is
