---
layout: post
title:  "How Does Social Sharing JS Link Work?"
date:   2017-01-20 19:59:03 -0800
---
![]({{ site.url }}/assets/social_share.png)
One day I wondered how we produce the social sharing link easily. After some searching, I found an open source project called [share.js](https://github.com/overtrue/share.js). I am a JS newbee, and I learned some useful stuff from this relatively simple repo. In this blog I will talk about my learings of *share.js*.

## How To Use
Here is a simple use case - inside the HTML body, user configures the title and description in a div of 'share-component' class:

```html
  <div class="share-component" data-disabled="facebook" data-description="Share.js - content to share!"></div>
```

This simple setting will enable the creation of the sharing buttons!

## Repo Structure
The repo has several pieces:
1. src: the sorurce coding containing JS, css and fonts.
2. gulpfile.js: The build file for generating the minified css/js files. The files are stored in the dist directory.
3. dist: js/css scripts location after minifying. you can directly use them for building the sharing buttons.
4. npm/bower files for easily installing the library.

## Anatomy of The Source Code
This repo uses the [Imediately invoked function expression (IIFE)](https://toddmotto.com/what-function-window-document-undefined-iife-really-means/),
in the format of (function (window, document, undefined) {})(window, document).
This funciton is executed immediately after being created. This pattern is often used when trying to avoid polluting the global namespace, because all the variables used inside the IIFE (like in any other normal function) are not visible outside its scope. IIFE also makes it easier for minifying.

Inside the IIFE, there is an [alReady method](https://github.com/jed/alReady.js). It provides a corss-browser way for implementinng domcontentloaded. The core method 'socialShare' is triggered through this method. Basically, right after dom conetent is loaded, socialShare is called to find element with id 'social-share' or 'share-component', and creates the sharing buttons inside that element. 

```javascript
window.socialShare = function (elem, options) {
    // querySelectorAlls is a generic method supporting document.querySelectorAll, jquery and zepto.
    elem = typeof elem === 'string' ? querySelectorAlls(elem) : elem;

    if (elem.length === undefined) {
        elem = [elem];
    }

    // Go over all the divs that we want to put sharing buttons on.
    each(elem, function (el) {
        if (!el.initialized) {
            // Create a list of sharing buttons.
            share(el, options);
        }
    });
};.
```

If there are such elements, it calls 'share' method, which implemtns the core features of this JS library:
 
```javascript
    // elem is a dom element with class  share-component or social-share.
    function share(elem, options) {
        var data = mixin({}, defaults, options || {}, dataset(elem));

        if (data.imageSelector) {
            data.image = querySelectorAlls(data.imageSelector).map(function(item) {
                return item.src;
            }).join('||');
        }

        // Attach class label share-component and social-share to elem if elem // does not have it.
        addClass(elem, 'share-component social-share');
        // For sites that are not diabled (set through data-disabled), create an icon element and set a URL it.
        createIcons(elem, data);
        // Wechat involves some QR code and need some special handling.
        createWechat(elem, data);
 
        elem.initialized = true;
    }
```

createIcons goes over all the sites we want to create a sharing button, and calls makeUrl to generate a sharing link. This makeUrl method is an interesting one.

```javascript
var defaults = {
    url: location.href,
    origin: location.origin,
    source: site,
    title: title,
    ...
};

var templates = {
    ...
    linkedin: 'http://www.linkedin.com/shareArticle?mini=true&ro=true&title={{TITLE}}&url={{URL}}&summary={{SUMMARY}}&source={{SOURCE}}&armin=armin',
    facebook: 'https://www.facebook.com/sharer/sharer.php?u={{URL}}',
    twitter: 'https://twitter.com/intent/tweet?text={{TITLE}}&url={{URL}}&via={{ORIGIN}}',
    google: 'https://plus.google.com/share?url={{URL}}'
};

function makeUrl(name, data) {
    data['summary'] = data['description'];

    return templates[name].replace(/\{\{(\w)(\w*)\}\}/g, function (m, fix, key) {
        var nameKey = name + fix + key.toLowerCase();
        key = (fix + key).toLowerCase();

        return encodeURIComponent((data[nameKey] === undefined ? data[key] : data[nameKey]) || '');
    });
}
```

Assume a user configures the title and description in a div of 'share-component' class:

```html
  <div class="share-component" data-description="Share.js - content to share!"></div>
```

In makeUrl method, data argument is the combination of data-* fields in the 'share-component' div and the defaults dictionary. So the sharing link description is "Share.js - content to share!", and the url is location.href. replace method is called on each sharing site template. This essentially replaces contents inside '{{' and '}}' and replaces them with contents from data argument.

As a side note, gulp is used for building and minifying the contents.
It combines and minifies qrcode.js and social-share.js into a file called social-share.min.js, and style files under css are processed with a name social-share.min.css.

## Some Learnings
First time reading open-source JS repository! It is a reasonable example about how to orgnize the code.