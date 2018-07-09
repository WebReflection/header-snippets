# Header Snippets
A collection of snippets to put in your header.

### Background

In my [A Bloatless Web post](https://itnext.io/a-bloatless-web-d4f811c7991b) I have shown tiny runtime features detection to avoid bloating code, or making network requests, in modern browsers.

I haven't wasted much time explaining those features detection, but they can be handy, and I want to collect them somewhere in here.

Please note these snippets should be used in your pages only if you need those features, and it's not wise to just trash everything in.

## Features Detection: Origami Style

Forget linters and formatting, use the least amount of maintainable code to reliably inject a patch when necessary.

<sup><sub>Semicolons might be necessary to avoid IDE highlight issues.</sub></sup>

### The most basic detection

```html
<script>this.Reflect||document.write(
  '<script src="https://unpkg.com/core-js-bundle"><\x2fscript>'
);</script>
```

Above snippet does the following things:

  * is there a `Reflect` object in the global `this` context ?
  * if yes, nothing else will be executed, happy browsing 🎉
  * if not, the _OR_ `||` will pass through the code on the right, executing it
  * the code on the right will be executed only on browsers that do not have [Reflect](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Reflect#Browser_compatibility) which are only legacy browsers
  * the `document.write` in those browsers will always work without a single warning or issue
  * `document.write` will grants, in those browsers, the next script on the header/page will be executed **after** the injected one has been parsed
  * the operation is blocking for legacy, implicitly inviting them to update their browsers if they think the Web is a bit slow (it's their fault, after all), and fully irrelevant for modern browsers

If you are wondering why the final `</script>` is written as `<\x2fscript>`, it's because otherwise the browser would close the current script at that position, resulting in a partially broken page.

Theoretically I could omit that final `<\x2fscript>` completely, and let the browser dial with the never closed `<script>` since the current one executing the operation will close, but that's too mind blowing to use or explain so I'll kept thing simple for documentation sake.

### The conditional comment

```html
<!--[if IE 8]>
  <script src="https://unpkg.com/ie8"></script>
<![endif]-->
```

Once upon a time, until IE 10 was published, unfortunately before the evergreen Edge was born, it was possible to reach specific versions of IE through [conditional comments](https://www.sitepoint.com/internet-explorer-conditional-comments/) via JS or HTML.

The benefit of this technique is that not a single browser different from the one you are targeting will ever even consider the content within those funny looking comments.

The script can be inline too.

```html
<!--[if lte IE 9]>
  <script>(function(f){window.setTimeout=f(window.setTimeout);window.setInterval=f(window.setInterval)})(function(f){return function(c,t){var a=[].slice.call(arguments,2);return f(function(){c.apply(this,a)},t)}});</script>
<![endif]-->
```

Above snippet, as example, will patch `setTimeout` and `setInterval` for IE9 and lower, providing extra arguments functionality.

### The multiple inline detection

```html
<script>(document.head&&document.head.after)||document.write(
  '<script src="https://unpkg.com/dom4"><\x2fscript>'
);</script>
```

Using a combination of the _AND_ `&&` operator, it is possible check multiple things or, if not satisfied, fallback through an _OR_ `||`.


### The inline try/catch

```html
<script>try{new EventTarget}catch(e){document.write(
  '<script src="https://unpkg.com/event-target"><\x2fscript>'
)}</script>
```

Above snippet will check in one single `try` the existence of the `EventTarget` and its ability to be used as constructor, falling back to a polyfill.

Don't worry about global scope pollution, the `catch` argument will never leak outside its `catch` block. That `e` won't exist anywhere else.

### The IIFE

```html
<script>(function(g,U){
  try {
    if (new g[U]('q=%2B').get('q')!='+' || new g[U]({q:1})!='q=1')
      throw{};
  } catch(e) {
    g[U]=void 0;
    document.write('<script src="https://unpkg.com/url-search-params"><\x2fscript>');
  }
}(this,'URLSearchParams'));</script>
```

Immediately Invoked Function Expressions are a way to create a temporary private scope, avoiding in this case pollution of the global scope or simply short-cutting some code through arguments.

Above snippet performs the following operations:

  * pass the global context as `g` to the IIFE
  * also pass the global identifier of the constructor we want to test
  * verify the constructor exists, it's usable as such, and it produces the expected result
  * if any of the previous cases fail, throw an error, in case it's not automatically thrown
  * delete the badly implemented constructor from the global context
  * bring in the polyfill for such constructor

Combining all techniques described above we can bring in selectively anything we need for our application.

## Shortcuts

There are various ways to simplify features detection, or reduce repeated check.

Nothing in here is strictly needed, but it might become handy if you need many detections.

### LEGACY

```html
<script>var LEGACY=!this.Reflect;</script>
```

Above snippet, placed at the very top of the page, can be used to both distinguish between ES2015 and ES5 targeted browsers, or inject, on demand, polyfills as we go.

The advantage of polluting the global scope upfront with a `LEGACY` boolean is to be able to find out, even after patching `Reflect`, if the browser needed such patch or not.

### write(test, url)

```html
<script>function write(o,O) {
  try{if(typeof o!='string'||!window[o])o()}
  catch(o){window['docum\x65nt']['writ\x65']('<script src="'+O+'"><\x2fscript>')}
}</script>
```

Above snippet would perform the following actions:

  * is the first argument a string?
  * if yes, and such string is not in the global context, throw an error by executing such string.
  * if not, execute the function instead.
  * if there is an error or the global name is unknown, bring in the polyfill.
  * use obfuscated `window['docum\x65nt']['writ\x65']` to hopefully avoid modern Chrome sniffing and deoptimizing the page anyhow

```html
<script>
// will bring in the polyfill if not found
write('customElements', 'https://unpkg.com/document-register-element');
</script>
<script>
// simplified EventTarget test
write(function(){new EventTarget}, 'https://unpkg.com/event-target');
</script>
```

Please note using `document.write` must be performed inline on the page otherwise `write(...)` will destroy your document. It is also important to understand that in order to have incremental patching, each `write(...)` might need its own script a part, otherwise the next `write(...)` wont consider the potentially already fixed behavior the previous write pulled in.

_TL;DR_ `write(...)` here is not usable as generic JS loader and it's better used one `<script>` per time.

## To document.write or not

In this [post from Phil Walton](https://philipwalton.com/articles/deploying-es2015-code-in-production-today/) there are great info about addressing the exact same concerns I have addressed in here.

However, that post is rising the `LEGACY` bar to ES2017+, using `<script type=module>` as elegant, and native, way to load modern code.

I believe in 2018 the `this.Reflect` check is more suitable for current browsers versions, making ES2015 a reliable target for any transpiled code.

I also believe in 2020 the `<script type=module>` will be the best way to distinguish, but the incremental fallback through `this.Reflect` and `ES5` might still be desired in some, hopefully rare, case.

These techniques should never conflict with each other, and [browsers interfering with legacy are shooting their own feet](https://developers.google.com/web/updates/2016/08/removing-document-write) (which is why I had to obfuscate `document.write`).

## Universal Bundle Loader

There is also another way to bring in bundlers, it's called [Universal Bundle Loader](https://github.com/WebReflection/ubl#ubl), or _ubl_ for short.
