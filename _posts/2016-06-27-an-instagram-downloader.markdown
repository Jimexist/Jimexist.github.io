---
layout: post
title:  "An Instagram Downloader"
categories: Instagram chrome extension
---

Recently I find myself often wanting to save great Instagram photos from others.

Although mostly accessed via its app, Instagram does have a [website version](https://instagram.com). It's a good starting point. However it's quite troublesome to write a crawler as the website only lazy-loads the urls and images when you scroll down, and those image urls are initially loaded via JavaScript/JSON, not via HTML.

Since the moment I realize I want to save certain photos, there are already there, rendered into HTML tags, so I did some digging:

```js
// Hoping this jQuery query does not break too often.
// Other class and id based approaches are not quite feasible
// due to the fact that the site is built using React and they are
// hash-based, i.e., can change quite frequently.
$('article img').each(function(i, img) {
  console.log(i, img.src);
});
```

So it worked as expected when pasted into the Chrome DevTools: the site has shielding set up for the photos so you cannot just right click and save the photo, but you can always get those `src` attributes by jQuery-ing `img`s.

So I went on and create [a content-script based Chrome extension](https://github.com/Jimexist/instagram-downloader), because this is a great way to evaluate a `js` code in the current tab's document as context. I did it in ES6 just to try things out, but to be honest that was an overkill. The extension uses [this great `fileSaver` package](https://github.com/eligrey/FileSaver.js/) that I've used before, and everything else is quite self-explanatory.

The extension only downloads a url list as a text file because I think [aria2c](https://aria2.github.io/) is a better tool for this step as it handles retries, url parsing, etc. nicely.
