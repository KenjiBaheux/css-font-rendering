css-font-timeout
================
A proposal for CSS to let web developers control font timeout.

#Background#
Most web browsers have adopted some of form of web font timeout:

Browser            | timeout      | fallback  | swap
------------------ | ------------ | --------- | --------
Chrome 35+         | 3 seconds    | yes       | yes
Firefox            | 3 seconds    | yes       | yes
Internet Explorer  | **0 seconds**| yes       | yes
Safari             | N/A          | N/A       | N/A
Opera              | *TODO*       | *TODO*    | *TODO*


For instance Chrome and Firefox have a 3 seconds **timeout** after which the text is shown with the **fallback** font. Eventually, a **swap** occurs: the text is re-rendered with the intended font as it becomes available.

On the other hand, Internet Explorer has a 0 seconds timeout which results in rendering the text with the fallback font from the get go. Eventually, the swap occurs as soon as the intended font becomes available.

Safari has no timeout behavior (or at least nothing beyond a baseline network timeout of 60([?](http://www.stevesouders.com/blog/2009/10/13/font-face-and-performance/)) seconds)

While one could argue that any of these default behaviors are reasonable, it's unfortunately inconsistent accross browsers and, more importantly, any of those one-size-fits-all approaches is unlikely to satisfy all the needs.

Sure, one can use the [Font Loading API](http://dev.w3.org/csswg/css-font-loading/) to carve the behavior they want. But it requires JavaScript and a non trivial amount of effort. Also, if the web developer relies on an external library for his font loading needs, it introduces one extra RTT.

Design/performance conscious web developers most likely have a good sense about the relative importance of a given web font to the intended user experience. So, we should give them the ability to control the font timeout behavior.


#Use cases#
##Overriding the timeout behavior for a specific font##

```
@font-face {
  font-family: 'Open Sans';
  font-style: normal;
  font-weight: 400;
  src: local('Open Sans'), local('OpenSans'),
  url(http://example.com/opensans/normal400.woff2) format('woff2');
  font-timeout: 2s;
  font-desirability: optional;
}
```

Where ```font-timeout: <time>``` indicates to the user agent that it can wait after the font for up to <time> after which point the text should be rendered with the fallback font.

Where ```font-desirability: [optional | mandatory]``` indicates to the user agent what should happen when the font becomes available **after the timeout**:
 - *optional* means that the user agent should *not swap* with the intended font as it becomes available
 - *mandatory* means that the user agent should *swap*  with the intended font as it becomes available


###Examples###
####The intended font is relatively unimportant####
Only use the intended font if it’s "readily available" , otherwise just let the user agent revert to the fallback.
```
timeout: 50ms;
font-desirability: optional;
````

####The intended font is critical####
For instance, you wouldn't want to show the text with a fallback font if the intented font is an [icon font](http://fortawesome.github.io/Font-Awesome/icons/).
```
timeout: 5000ms;
font-desirability: mandatory;
````

####The intended font is important but up to a point####
Wait for the intended font for a sufficiently long time. When timeout kicks in, the user agent should revert to the fallback font until the intended font becomes available at which point the user agent should swap back to the intended font.

```
timeout: 1000ms;
font-desirability: mandatory;
```

##Overriding the timeout behavior for specific CSS style definitions##
With the [Font Loading API](http://dev.w3.org/csswg/css-font-loading/), one can customize the behavior of individual document fragments. The same should be achievable in CSS. In order to do that, font-timeout and font-desirability should be usable in regular style definitions. For instance:

```
@font-face {
  font-family: 'Open Sans';
  font-style: normal;
  font-weight: 400;
  src: url(http://example.com/opensans/normal400.woff2) format('woff2');
  
  /* global default, if not set then UA default (e.g. Chrome, Firefox: 3s) */ 
  font-timeout: 2s;
  
   /* global default, if not set then UA default (e.g. Chrome, Firefox: mandatory) */
  font-desirability: optional;
}


body { font-family: Open Sans; }

#header {
  /* header in the right font is important for branding but up to a point */
  font-timeout: 500ms;
  
   /* we do want to show the header in the right font eventually for branding */
  font-desirability: mandatory;
}

#main-content {
  /* set aggressive timeout on main content, gotta get the content visible ASAP */
  font-timeout: 100ms;
  
  /* inherits font-desirability: optional. */
}

# footer {  
  /* inherits font-timeout: 2s timeout */
  font-desirability: mandatory;
}
```


#Contributors#
With advices/contributions from: Tab Atkins, Ilya Grigorik, David Kuettel.
