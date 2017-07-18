# Polyfill that mimics Chrome's scroll restoration behavior

In single page apps that use the history API for navigation, the DOM is
typically not completely ready immediately when the `popstate` event is
fired, as opposed to traditional applications where after page load the
page is completely filled with content.

This means that the scroll position cannot be set until the AJAX requests
have finished loading, and the page is fully rendered. It seems that Chrome
implements this behavior by looking at AJAX requests made by the page,
and not restoring the scroll position until these have finished, unless
it is already possible earlier (e.g. if the page already has the correct
height).

See [Chrome's implementation](https://chromium.googlesource.com/chromium/blink/+/5da5b59/Source/core/loader/FrameLoader.cpp#1049).

This polyfill tries to mimic that behavior, though it is a rather limited
implementation because of browser differences and insufficient browser APIs.

Check out [Brigade.com](https://brigade.com) for an example:

![Demo on Brigade.com](https://raw.github.com/brigade/delayed-scroll-restoration-polyfill/master/demo.gif)

## Usage

### npm
```
npm install delayed-scroll-restoration-polyfill --save
```

### bower
```
bower install delayed-scroll-restoration-polyfill --save
```

### In your HTML
```html
<script src='node_modules/delayed-scroll-restoration-polyfill/index.js'></script>
```

## How it works

1. We overwrite `history.pushState` to record the current scroll position
   when navigating to a new page.
2. We also overwrite `history.replaceState` to avoid overwriting this scroll
   position.
3. Finally, we listen to the `popstate` event, and when it's fired we keep
   trying to restore the scroll position, which we only do if the page
   actually  has the correct width and height.
4. After a few seconds we time out and scroll as far as we can.

## Disabling native implementations

While this polyfill is designed to work alongside native implementations, if you
nevertheless experience compatibility problems, we suggest you disable native
scroll restoration in browsers that support it, like this:

```js
if ('scrollRestoration' in window.history) {
  window.history.scrollRestoration = 'manual';
}
```

## Differences from Chrome's native implementation

1. We use a timeout instead of checking which resources are loaded and such,
   because it's more convenient and it doesn't seem to matter much in practice.
   This does imply that if the user resizes the browser window during their
   session, causing pages to become less high (e.g. by making the window wider),
   then we might not restore the scroll position until we time out.

2. We don't abort restoring the scroll position when the user scrolls,
   because we cannot detect a user scroll from a browser scroll (e.g. because
   the page height changes), which the browser can do natively.

   Note that we cannot reliably detect if we are in a browser that already
   supports the delayed scroll restoration behavior, so this polyfill will
   break Chrome's behavior of aborting scroll restoration on user scroll.

3. We don't record the scroll position when clicking the back button, only
   when using `history.pushState`. This means that we don't get delayed scroll
   restoration when clicking the forward button in the browser. The reason for
   this is that we cannot reliably store the scroll position when the back
   button is clicked, due to spec / browser implementation. See:
   - https://github.com/rackt/react-router/issues/707#issuecomment-71552632
   - https://bugzilla.mozilla.org/show_bug.cgi?id=1155730#c1
   - https://bugzilla.mozilla.org/show_bug.cgi?id=679458#c16

   This problem can get solved when the Custom Scroll Restoration API gets
   widely implemented:
   - https://code.google.com/p/chromium/issues/detail?id=477353
   - http://majido.github.io/scroll-restoration-proposal

4. The polyfill assumes that on `popstate` the page is cleared, or at least
   that the height will only increase right after `popstate`. We do wait a tick
   to make sure that your code gets run before we attempt to scroll, but if the
   page height decreases after that then we might have already scrolled and we
   won't wait until the page height increases again.

* * *

Note that this polyfill just aims to mimic Chrome's behavior as closely
as possible, and does not attempt to keep scrolling down in cases when an
infinite list is not fully loaded. That case would also break in Chrome,
and so you need to provide sufficient caching yourself so that when
navigating back to a page it can render again relatively quickly.

## Changelog

If you're interested in seeing the changes and bug fixes between each version,
read the [Changelog](CHANGELOG.md).

## Code of conduct

This project adheres to the [Open Code of Conduct][code-of-conduct]. By
participating, you are expected to honor this code.

[code-of-conduct]: https://github.com/brigade/code-of-conduct

## License

This project is released under the [MIT license](MIT-LICENSE).
