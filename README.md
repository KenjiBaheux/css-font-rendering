# css-font-rendering

A proposal to let web developers control font-rendering via CSS.

## Background

Most web browsers have adopted some of form of webfont timeout:

Browser            | timeout      | fallback  | swap
------------------ | ------------ | --------- | --------
Chrome 35+         | 3 seconds    | yes       | yes
Opera              | 3 seconds    | yes       | yes
Firefox            | 3 seconds    | yes       | yes
Internet Explorer  | **0 seconds**| yes       | yes
Safari             | N/A          | N/A       | N/A

* Chrome and Firefox have a 3 second **timeout** after which the text is shown with the **fallback** font. Eventually, a **swap** occurs: the text is re-rendered with the intended font once it becomes available.
* Internet Explorer has a 0 second timeout which results in immediate text rendering: if the requested font is not yet available, fallback is used, and text is rerendered later once the requested font becomes available.
* Safari has no timeout behavior (or at least nothing beyond a baseline network timeout of 60([?](http://www.stevesouders.com/blog/2009/10/13/font-face-and-performance/)) seconds)

While one could argue that any of these default behaviors are reasonable, it's unfortunately inconsistent accross browsers and, more importantly, none of these one-size-fits-all approaches is insufficient to cover the range of use cases required by modern UX+performance concscious applications.

The [Font Loading API](http://dev.w3.org/csswg/css-font-loading/) allows the developer to override some of the above behaviors, but that path requires JavaScript execution, a non trivial amount of developer effort, and ultimately does not provide the necessary hooks to deliver consistent first-paint experience - e.g. the API can be used ot set custom timeouts, but there is no well defined "first-paint" event provided by the browser that could be used to ensure consistent rendering behavior. Also, if the web developer relies on an external library for his font loading needs, it introduces extra network RTTs into the critical rendering path and delays text rendering.

Design/performance conscious web developers have a good sense for the relative importance of a given web font for the intended user experience. We should give them the ability to control font timeout and rendering behavior. Developers should be able to:

* Define font rendering strategy when text is ready to be painted - e.g. block or paint with fallback.
* Define font rendering behavior once the desired font is available - e.g. rerender text or use fallback.
* Define custom timeout values for each font.
* Define custom render and timeout strategies on per selector basis.


## `font-rendering` parameter

`font-rendering: optional | swap <time>? | mandatory <time>?`

* `optional`
  * user agent should not block text rendering
  * if the desired font is available then it should be used, but if the font is not available then the fallback should be used
  * once rendered, the text is not re-rendered - i.e. first font render wins.

* `swap <time>?`
  * user agent should not block text rendering
  * if the desired font is available then it should be used, but if the font is not available then the fallback should be used
  * once the desired font becomes available, the text should be rerendered with desired font
    * if the specified timeout is reached, then fallback remains active and text is not rerendered
    * if timeout value is omitted, default value of 3s is used.
  
* `mandatory <time>?`
  * user agent should block text rendering until desired font is available
    * if the specified timeout is reached, text is rendered with the fallback font
    * if timeout value is omitted, default value of 3s is used.

If left unspecified, font-rendering must default to `mandatory 3s` - i.e. current FF, Opera, and Chrome behavior.


### Examples
#### The intended font is optional

```css
font-rendering: optional;
```

Use the desired font if it is available when text is ready to be painted, and if the font resource is not yet ready (e.g. request is still in flight), then the fallback font should be used to render the text. Once text is rendered, it is not updated - i.e. first available font wins, and there are no delays in text rendering.


#### The intended font is progressive

```css
font-rendering: swap 500ms; /* abandon progressive strategy after 0.5s */
```

If the desired font is not available when text is ready to be painted, use the fallback font and rerender the text once the desired font is available. Abandon the progressive strategy and keep using the fallback font if the desired font exceeds the specified timeout.


#### The intended font is mandatory

```css
font-rendering: mandatory; /* wait for desired font, use fallback after 3s (default timeout) */
````

Block text rendering until desired font is available, or until the specified timeout has expired. For instance, you wouldn't want to show the text with a fallback font if the intented font is an [icon font](http://fortawesome.github.io/Font-Awesome/icons/).


## Per-selector font rendering strategy

The text rendering strategy can vary based on type of content displayed on the page - e.g. headline, body text, notes, footer, etc. As such, it should be possible to define custom text rendering strategies for different document fragments via `font-rendering`. For instance: 

```css
@font-face {
  font-family: 'Open Sans';
  font-style: normal;
  font-weight: 400;
  src: url(//example.com/opensans/normal400.woff2) format('woff2');

  /* 
    set default font-rendering strategy 
    - don't block text rendering on this font
    - abandon progressive rendering after 2s
  */ 
  font-rendering: swap 2s;
}

body { font-family: Open Sans; }

#header {
  /* block rendering until desired font is avaialble.. due to branding requirements.
     if branding font is not ready within 500ms, use fallback */
  font-rendering: mandatory 500ms; 
}

#headline {
  /* immediately render the headlines, if the desired font is available, great...
     and if not, use fallback and don't rerender to minimize reflow */
  font-rendering: optional;
}

#main-content {
  /* don't hold rendering, but rerender with desired font once available */
  /* give up on progressive strategy after 150ms due to UX+perf requirements */
  font-rendering: swap 150ms;
}

#footer {  
  /* inherits font-rendering: swap */
}
```

## Contributors
With advices/contributions from: Tab Atkins, Ilya Grigorik, David Kuettel.
