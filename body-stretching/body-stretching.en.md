# Stretching body to full viewport height: the missing way

So, you're making a sticky footer or centering some content relative to the viewport. You want to stretch the `body` to full browser window height, at the same time allowing it to stretch to its content. No doubt, this task was solved a bazillion of times, and it should be as easy as pie. Right? _Right?_

## The state-of-the-art way

Sure! Just set `min-height: 100vh` to the `body`. `100vh` is 100% of the viewport height, and using `min-height` instead of `height` does not fix the `body` height, allowing it to grow as necessary. Exactly what we need!

Well... Almost. Apparently, in a typical mobile browser such a page will always have a scroll, and the bottom of the page will disappear beneath the browser bottom UI panel. Even if the content fits the screen perfectly!

The case is that on mobile devices `100vh` is the maximum possible viewport height, but the initial viewport height is usually _smaller_. This happens due to browser UI elements that initially take up some space, but then shrink or disappear on the page scroll.

![Mobile browser scroll demo](./resources/100vh-scroll.png)

It's possible to google a [fix](https://css-tricks.com/css-fix-for-100vh-in-mobile-webkit/) for that issue that looks like this:

```css
html {
    height: -webkit-fill-available; /* We have to fix html height */
}

body {
    min-height: 100vh;
    min-height: -webkit-fill-available;
}
```

It seems that Chrome has a bug where the `body` height can get out of sync with the viewport height on the browser height change. Aside from that, the issue is really solved, but now we have to fix the `html` height. If that's the case, why shouldn't we use an older, more robust solution?

## The old-school way

We couldn't get rid of fixing the `html` height, so let's try the good old way that involves passing a 100% height from the `html`.

Let's set `min-height: 100%` to the `body`, where 100% is the full height of the parent (the `html` element). Percentage height requires the parent to have a fixed height, so we have to set `height: 100%` to the `html`, fixing its height to full viewport height.

By the way, in case of mobile browsers, percentage height for an `html` is calculated relative to the _minimal_ viewport height, so the scroll issue mentioned above is not present anymore!

```css
html {
    height: 100%; /* We still have to fix html height */
}

body {
    min-height: 100%;
}
```

This solution is not as pretty as the `100vh` one, but it's been used since time immemorial, and it will work, that's for sure!

Well... Not quite. Apparently, the gradient set to such a `body` will be cut at the `html` height (in other words, at the viewport height, or, even more precise, at the _minimal_ viewport height).

![Broken gradient demo](resources/gradient-clip.png)

Of course, this can be "fixed" by setting the gradient to the `body` content, but that's just not _right_. The page background _should_ be set to the `body`, and the `html` _should_ stretch to its content. Can we achieve that?

## The missing way

I dare to suggest another way of stretching the `body` to full viewport height that lacks the issues mentioned above. The core idea is that we pass the 100% `html` height via the flexbox, and therefore we are not forced to fix the `html` height.


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

* It should be obvious that the flexbox-based height passing works for any depth. It can easily be used in cases where the content is being rendered to some element inside the `body`, and not the `body` itself. It's a typical scenario with [React](https://medium.com/@dan_abramov/two-weird-tricks-that-fix-react-7cf9bbdef375) or [Vue](https://vuejs.org/v2/api/#el), for example.

* The flexbox-based height passing doesn't work in IE. Not at all. But you don't support it anyway, do you?