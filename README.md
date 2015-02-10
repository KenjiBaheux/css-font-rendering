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

While one could argue that any of these default behaviors are reasonable, it's unfortunately inconsistent accross browsers and, more importantly, none of these one-size-fits-all approaches is sufficient to cover the range of use cases required by modern UX+performance concscious applications.

The [Font Loading API](http://dev.w3.org/csswg/css-font-loading/) allows the developer to override some of the above behaviors, but that path requires JavaScript execution, a non trivial amount of developer effort, and ultimately does not provide the necessary hooks to deliver consistent first-paint experience - e.g. the API can be used to set custom timeouts, but there is no well defined "first-paint" event provided by the browser that could be used to ensure consistent rendering behavior. Also, if the web developer relies on an external library for his font loading needs, it introduces extra network RTTs into the critical rendering path and delays text rendering.

Design/performance conscious web developers have a good sense for the relative importance of a given web font for the intended user experience. We should give them the ability to control font timeout and rendering behavior. Developers should be able to:

* Define font rendering strategy when text is ready to be painted - e.g. block or paint with fallback.
* Define font rendering behavior once the desired font is available - e.g. rerender text or use fallback.
* Define custom timeout values for each font.
* Define custom render and timeout strategies on per selector basis.


## `font-rendering` parameter

`font-rendering: optional | swap | block | auto | [ [ block <timeout> ]? [ swap <timeout> ]? ]!`

* `optional`
  * user agent should not block text rendering.
  * if the desired font is available then it should be used, but if the font is not available then the next font fallback should be considered (and so on).
  * once rendered, the text is not re-rendered - i.e. first font render wins.

* `swap <timeout>?`
  * user agent should not block text rendering
  * if the desired font is available then it should be used, but if the font is not available then the next font fallback should be considered (and so on)
  * once the desired font becomes available, the text should be rerendered with desired font
    * if the specified timeout is reached, then the fallback remains active and the text is not rerendered
    * if the timeout value is omitted, the user agent shall use a reasonable default value.

* `block <timeout>?`
  * user agent should block text rendering until desired font is available.
    * if the specified timeout is reached, text is rendered with the fallback font.
    * if timeout value is omitted, the user agent shall use a reasonable default value.

If left unspecified, font-rendering must default to `swap`.

Finally, regarding the "reasonable default value" for when a keyword is missing an explicit timeout value, data analysis done in November 2014 lead Chrome to pick 3s as its default value. As network performance and technology improve, one can expect that what was deemed reasonable back then, becomes unacceptable. Therefore, we recommend that web developers specify the timeout value if they intend to rely on it.

### About the `[ [ block <timeout> ]? [ swap <timeout> ]? ]!` pattern

This pattern means that you have to specify [at least one of block or swap (with/without the optional timeout values) in that order](http://dev.w3.org/csswg/css-values/#combinator-multiplier-patterns)


## Optional `<timeout>` value

`<timeout>` is defined as `<timeout> = <time> | infinite` where `<time>` refers to [CSS3's time-value](http://dev.w3.org/csswg/css-values-3/#time-value)


## Alternative syntax
`font-rendering-block-timeout: <timeout>` and `font-rendering-swap-timeout: <timeout>` are alternatives to `font-rendering: block <timeout>` and `font-rendering: swap <timeout>` respectively.


### Examples
#### The intended font is optional

```css
font-rendering-block-timeout: 0s;
font-rendering-swap-timeout: 0s;

/* or for short */
font-rendering: block 0s swap 0s;

/* or slightly shorter */
font-rendering: swap 0s; /* omitting block is equivalent to block 0s */

/* or even shorter */
font-rendering: optional;
```

Use the desired font if it is available when the text is ready to be painted, and if the font resource is not yet ready (e.g. request is still in flight), then the fallback font should be used to render the text. Once text is rendered, it is not updated - i.e. first available font wins, and there are no delays in text rendering.


#### The intended font is progressive

```css
/* abandon progressive strategy after 500ms */
font-rendering-block-timeout: 0s; /* can be omitted (default value: block 0s) */
font-rendering-swap-timeout: 500ms;

/* or for short **/
font-rendering: block 0s swap 500ms;

/** or even shorter **/
font-rendering: swap 500ms; /* omitting block is equivalent to block 0s */
```

Alternatively, if you only care about the behavior but not the actual timeout (which may differ accross user agents):
```css
font-rendering: block 0s swap;

/* or for short **/
font-rendering: swap; /* omitting block is equivalent to block 0s */
```

If the desired font is not available when the text is ready to be painted, use the fallback font and rerender the text once the desired font is available. Abandon the progressive strategy and keep using the fallback font if the desired font exceeds the specified timeout.


#### The intended font is important to the user experience but not to the point of re-rendering after the timeout expires
```css
/* wait for the desired font, let the user agent try the next fallback font after the specified timeout expires */
font-rendering-block-timeout: 2s;
font-rendering-swap-timeout: 0s;

/* or for short **/
font-rendering: block 2s swap 0s;

/** Note: font-rendering: block 2s; is not equivalent as omitting swap would mean swap infinite **/
/** To be updated based on discussion at [Issue 11] **/
```

To be updated based on discussion at [Issue 11](https://github.com/KenjiBaheux/css-font-rendering/issues/11)
Alternatively, if you only care about the behavior but not the actual timeout  (which may differ accross user agents):
```css
font-rendering: block; /* wait for the desired font, let the user agent use a fallback after its default timeout expires) */
````

Block text rendering until desired font is available, or until the specified timeout has expired.


#### The intended font is mandatory

```css
 /* makes the desired font mandatory */
font-rendering-block-timeout: infinite;
font-rendering-swap-timeout: infinite;

font-rendering: block infinite swap infinite;

/* or shorter */
font-rendering: block infinite; /* omitting swap is equivalent to swap infinite */
````

Block text rendering until desired font is available, do not use any fallback (even if the font request failed).

Note: blocking indefinitely with no fallback can result in a very poor user experience and should therefore be reserved for cases where a fallback font doesn't make sense. (e.g. an icon font).


### Other combinations
 - 0s infinite: render immediately, allow swap indefinitely. Note: this can result in jarring user experience (after a certain point swapping will annoy users who are now in the middle of reading)
 - 1s+ 1s+: block for X seconds, allow swap for Y seconds; use fallback if X times out.
 - 1s+ infinite: block for X seconds, allow swap indefinitely; use fallback if X times out or req fails. Note: this can result in jarring user experience (after a certain point swapping will annoy users who are now in the middle of reading)
 - infinite 0s: block until desired font is available, use fallback if request failed. (TBC in issue 11)
 - infinite 1s+: block until desired font is available, use fallback if request failed (same as above; (TBC in issue 11)).

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
  /* block rendering until desired font is available (due to branding requirements).
     if branding font is not ready within 500ms, use fallback */
  font-rendering: block 500ms;
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

## Representation of `<timeout>` less keywords

## Contributors
With advices/contributions from: Tab Atkins, Ilya Grigorik, David Kuettel.
