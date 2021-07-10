# Stretching body to full viewport height: the missing way

Suppose you're making a sticky footer or centering some content relative to the viewport. You want to stretch the `body` to the full height of the browser window, while also letting it stretch to its content. This task was surely solved a bazillion times, and it should be as easy as pie. Right? _Right?_

## The state-of-the-art way

Sure! Applying `min-height: 100vh` to the `body` should do the trick. `100vh` means that the initial `body` height will take 100% of the viewport height, whereas the use of `min-height` instead of `height` will let the `body` grow even more if necessary. This is exactly what we need!

Well... Almost. It turns out that, in a typical mobile browser, such a page will always be scrollable, and its bottom will disappear beneath the bottom UI panel of the browser. Even if the page content fits the screen perfectly!

The reason for this is fairly simple. UI elements in mobile browsers tend to shrink after the scroll, leaving more space for the actual content. `100vh` usually corresponds to the _maximum_ possible viewport height, and, since the initial viewport height is usually smaller, the `body` with a `min-height` of `100vh` may initially exceed the screen height regardless of its content.

This is true at least for [iOS Safari](https://bugs.webkit.org/show_bug.cgi?id=141832#c5) and [Android Chrome](https://developers.google.com/web/updates/2016/12/url-bar-resizing).

![Mobile browser scroll demo](./resources/100vh-scroll.png)

The known [fix](https://css-tricks.com/css-fix-for-100vh-in-mobile-webkit/) for this issue looks like this:

```css
html {
    height: -webkit-fill-available; /* We have to fix html height */
}

body {
    min-height: 100vh;
    min-height: -webkit-fill-available;
}
```

This solution has a minor glitch: there seems to be a bug in Chrome which makes the `body` height get out of sync with the viewport height when the browser height changes. Aside from that, this approach solves the issue.

However, we now have to fix the `html` height. If that's the case, shouldn't we use an older, more robust solution?

## The old-school way

Since we couldn't avoid fixing the `html` height, let's try the good old way that involves passing a 100% height from the `html`.

Let's apply `min-height: 100%` to the `body`, where 100% is the full height of its parent (namely, the `html` element). A percentage height on a child requires the parent to have a fixed height, so we have to apply `height: 100%` to the `html`, thereby fixing its height to the full viewport height.

Since the percentage height of an `html` element in mobile browsers is calculated relative to the _minimal_ viewport height, the above-mentioned scroll issue doesn't bug us anymore!

```css
html {
    height: 100%; /* We still have to fix html height */
}

body {
    min-height: 100%;
}
```

This solution is not as pretty as the `100vh` one, but it's been used since time immemorial, and it will work, that's for sure!

Well... Not quite. Apparently, the gradient applied to such a `body` will be cut at the `html` height (in other words, at the viewport height, or, to be more precise, at the _minimal_ viewport height).

It happens because of the fixed `html` height, and it doesn't matter whether it's `height: 100%` or `height: -webkit-fill-available`.

![Broken gradient demo](resources/gradient-clip.png)

Of course, this can be "fixed" by applying the gradient to the `body` content, but that's just not _right_. The page background _should_ be applied to the `body`, and the `html` _should_ stretch to its content. Can we achieve that?

## The missing way

I dare to suggest another way of stretching the `body` to the full viewport height that lacks the above-mentioned issues. The core idea is to use flexbox to pass the 100% `html` height, which saves us from having to fix it.

```css
html {
    min-height: 100%; /* Look, it's not fixed anymore! */

    display: flex;
    flex-direction: column;
}

body {
    flex-grow: 1;
}
```

Now the `html` can stretch to its content, and there are no issues with mobile browsers whatsoever. Neat!

## Notes

-   It should be obvious that the flexbox-based height passing works for any depth. It can easily be used in cases where the content is being rendered to some element inside the `body`, and not the `body` itself. It's a typical scenario with [React](https://medium.com/@dan_abramov/two-weird-tricks-that-fix-react-7cf9bbdef375) or [Vue](https://vuejs.org/v2/api/#el), for example.

-   The flexbox-based height passing doesn't work in IE. Not at all. But you don't support it anyway, do you?
