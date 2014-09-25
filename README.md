# css-font-timeout

A proposal for CSS to let web developers control font timeout and rendering strategy.

## Background
Most web browsers have adopted some of form of webfont timeout:

Browser            | timeout      | fallback  | swap
------------------ | ------------ | --------- | --------
Chrome 35+         | 3 seconds    | yes       | yes
Firefox            | 3 seconds    | yes       | yes
Internet Explorer  | **0 seconds**| yes       | yes
Safari             | N/A          | N/A       | N/A
Opera              | *TODO*       | *TODO*    | *TODO*

* Chrome and Firefox have a 3 second **timeout** after which the text is shown with the **fallback** font. Eventually, a **swap** occurs: the text is re-rendered with the intended font once it becomes available.
* Internet Explorer has a 0 second timeout which results in immediate text rendering: if the requested font is not yet available, fallback is used, and text is rerendered later once the requested font becomes available.
* Safari has no timeout behavior (or at least nothing beyond a baseline network timeout of 60([?](http://www.stevesouders.com/blog/2009/10/13/font-face-and-performance/)) seconds)

While one could argue that any of these default behaviors are reasonable, it's unfortunately inconsistent accross browsers and, more importantly, none of these one-size-fits-all approaches is insufficient to cover the range of use cases required by modern UX+performance concscious applications.

The [Font Loading API](http://dev.w3.org/csswg/css-font-loading/) allows the developer to override some of the above behaviors, but that path requires JavaScript execution, a non trivial amount of developer effort, and ultimately does not provide the necessary hooks to deliver consistent first-paint experience - e.g. the API can be used ot set custom timeouts, but there is no well defined "first-paint" event provided by the browser that could be used to ensure consistent rendering behavior. Also, if the web developer relies on an external library for his font loading needs, it introduces extra network RTTs into the critical rendering path and delays text rendering.

Design/performance conscious web developers have a good sense for the relative importance of a given web font to the intended user experience. We should give them the ability to control font-timeout and rendering behavior. Developers should be able to:

* Define first-paint font rendering strategy
* Define font rerendering behavior if the desired font is not used on first paint
* Define custom font timeout values to trigger fallback rendering
* Define custom font rendering and timeout strategies on per selector basis


## `font-timeout` and `font-desirability` parameters

```css
@font-face {
  font-family: 'Open Sans';
  font-style: normal;
  font-weight: 400;
  src: local('Open Sans'), local('OpenSans'),
  url(//example.com/opensans/normal400.woff2) format('woff2');
  font-timeout: 2s; /* 2 second timeout */
  font-desirability: optional;
}
```

* ```font-timeout: <time>``` indicates to the user agent that it should wait for up to <time> for the requested font, after which point the text should be rendered with the fallback font.
* ```font-desirability: [optional | progressive | mandatory]``` indicates to the user agent the desired text rendering strategy:
 - `optional` - user agent should not block text rendering; if the desired font is available then it should be used, but if the font is not available then the fallback should be used; once rendered, the text is not re-rendered (i.e. first font render wins)
 - `progressive` - user agent should not block text rendering; if the desired font is available then it should be used, but if the font is not available then the fallback should be used; once the desired font becomes available, the text should be rerendered with desired font (i.e. progressive text rendering)
 - `mandatory` - user agent should block text rendering until the desired font is available, or until font-timeout is reached. In the latter case, fallback font should be used to render text.

The combination of `font-timeout` and `font-desirability` enables the developer to specify a consistent font rendering strategy: whenever the browser needs to render text content it can consult specified parameters to determine if fallback should be used, if rerendering is required in the future, or if text rendering should be blocked until desired font is available. 


### Examples
#### The intended font is optional

```css
font-desirability: optional;
```

Use the intended font if it is available when text is ready to be painted, and if the font resource is not yet ready (e.g. request is still in flight), then the fallback font should be used to render the text. Once text is rendered, it is not updated - i.e. first available font wins, and there are no delays in text rendering.


#### The intended font is progressive

```css
font-desirability: progressive;
font-timeout: 500ms; /* abandon progressive strategy after 0.5s */
```

If the desired font is not available when text is ready to be painted, use the fallback font and rerender the text once the desired font is available. Further, abandon the progressive strategy and keep using the fallback font if the desired font exceeds the specified timeout.


#### The intended font is critical

```css
font-desirability: mandatory;
font-timeout: 2s; /* render with fallback after 2s */
````

Hold text rendering until desired font is available, or until the specified timeout has expired. For instance, you wouldn't want to show the text with a fallback font if the intented font is an [icon font](http://fortawesome.github.io/Font-Awesome/icons/).


## Per-selector font rendering strategy

The text rendering strategy can vary based on type of content displayed on the page - e.g. headline, body text, notes, footer, etc. As such, it should be possible to define custom text rendering strategies for different document fragments via `font-timeout` and `font-desirability`. For instance: 

```css
@font-face {
  font-family: 'Open Sans';
  font-style: normal;
  font-weight: 400;
  src: url(//example.com/opensans/normal400.woff2) format('woff2');
  
  /* set global default, if not set then UA default (e.g. Chrome, Firefox: 3s) */ 
  font-timeout: 2s;
  
   /* set global default, if not set then UA default (e.g. Chrome, Firefox: mandatory) */
  font-desirability: progressive;
}

body { font-family: Open Sans; }

#header {
  /* block rendering until desired font is avaialble.. due to branding requirements */
  font-desirability: mandatory;
  /* if branding font is not ready within 500ms, use fallback */
  font-timeout: 500ms;
}

#headline {
  /* immediately render the headlines, if the desired font is available, great...
     and if not, use fallback and don't rerender to minimize reflow */
  font-desirability: optional;
}

#main-content {
  /* don't hold rendering, but rerender with desired font once available */
  font-desirability: progressive;
  /* give up on progressive strategy after 150ms due to UX requirements */
  font-timeout: 150ms;
}

#footer {  
  /* inherits font-desirability: progressive */
  /* inherits font-timeout: 2s timeout */
}
```

## Contributors
With advices/contributions from: Tab Atkins, Ilya Grigorik, David Kuettel.
