{:title "Learn You A Kee-Frame, Part 4"
 :layout :post
 :tags  ["clojure" "lyakf"]}

Originally I just wanted to take a look at re-frame/kee-frame.
But I've been thinking about a program aware training log
that runs on my phone for a while now.  So I had been
thinking about React Native and that possibility was in
the back of my mind.

I was  not really aware of "Progressive Web Apps",
I only had a vague notion that webapps these days can
run offline too.  But I did read up on it and figured this
might be a viable option.  And I would 
not have to deal with app stores.

## Making a Progressive Web App

TEDIOUS BUT WORTH IT.

Mistake #1, I started with MDN which usually works out well for me.
But apparently Mozilla is not as hot about PWAs as they used to be,
and lets just say that I found much better docs elsewhere.

Mistake #2, I use firefox - I would not say I trust Mozilla,
but I trust them a lot more than Google.  But did I mention that Mozilla is not
really into PWAs anymore?   They removed the install feature
from Firefox, which explains why I could not make it work.

Mistake #3, I did not use [Workbox][workbox].  I figured it was too much
effort to bring javascript tooling into the build.  So I spent
a full day pressing reload and trying to guess why it was not working.
Spending the time to integrate existing tooling would have been
faster.

That being said, you will need some kind of tooling.
Service workers don't work well with hot reloading, so
you want to turn this off for development.  And
the exact path of deployment is very important, unless
that is the same for release and development you will
have to modify some files.

I am using `babaska` and `selmer` for this.

### HTML Header

```html
<head>
    <meta charset="utf-8" />
    <title>Learn You A Kee-Frame</title>
    <meta description="Example PWA using re-frame/kee-frame" />
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <link rel="stylesheet" href="css/bulma.min.css">
    <script src="js/main.js" defer></script>
    {% if not debug? %}
    <!-- google uses manifest.json, will this fix offline mode? -->
    <link rel="manifest" href="{{ base-path }}/manifest.json">
    <script src="js/register_sw.js" defer></script>
    {% endif %}
</head>
```

The journey starts with the html header.  You absolutely need
that `viewport` line or the app will look horrible on a phone.
Next, `<link rel="manifest">` - the header points to the
application manifest.   In some docs it is called
`foo.webmanifest` instead (newer), both work.  The important
bit is `{{ base-path }}`:  service workers are all about
scope, and you MUST bring the deployment base path into
your tooling - relative paths will not work (actually, some
docs show this one relative but I just bit the bullet
and used the full path everywhere).

`js/register_sw.js` is the script that will register the
service worker.  You could do this from the main app (you
might even have to if you want to influence the installation
popups), but it's just a couple of javascript API calls,
I don't see the point in using clojurescript for that.


### Manifest.json

```json
{
    "name": "Learn You A Kee-Frame",
    "short_name": "lyakf",
    "description": "Example PWA using re-frame/kee-frame",
    "start_url": "{{ base-path }}/",
    "id": "{{ base-path }}/",
    "prefer_related_applications": false,
    "display": "standalone",
    "icons": [
        {
            "src": "icons/android-chrome-192x192.png",
            "sizes": "192x192",
            "type": "image/png"
        },
        {
            "src": "icons/android-chrome-512x512.png",
            "sizes": "512x512",
            "type": "image/png"
        }
    ],
    "theme_color": "#ffffff",
    "background_color": "#ffffff"
}
```

Everything except the last 2 lines is essential in this file.
You need those icons, without them the app is not eligible
for installation.  I used [Favicon Generator][favicon-generator]
because it generates icons for me, my graphicals skills are
sorely lacking.

`start_url` and `id` are especially important:  you have to
use the base path here AND NOT `index.html`.  Kee-frame wants
`/` for the app root, if you append `index.html` it will choke.
But your installed app will be called with `start_url`, 
which results in a never ending loading spinner.

Oh, I almost forgot: if you deploy into a URL with a sub-directory
(*cough* *cough* github pages), you will need my fork
of kee-frame for the moment: regular kee-frame cant deal
with non-root locations.   There is a [pull request][kf_110]

### Registering the Service Worker

```javascript
const basePath = '{{ base-path }}'

const registerServiceWorker = async () => {
    if ('serviceWorker' in navigator) {
        try {
            const registration = await navigator.serviceWorker.register(basePath + '/sw.js', {})
            // leaving out some stuff that helps with updating the app
        } catch (error) {
            console.error(`Registration failed with ${error}`)
        }
    }
}

registerServiceWorker()
```

The crucial bit here is that `basePath + '/sw.js'` in the try block.
Service workers have scopes - if you get this wrong, the worker
will be installed and active, but it will not be actually used
to fetch your pages, which means the app may be installed but it will
not work offline.

All your assets must be inside `basePath`, and the service worker
must be at the root of it.  I made the mistake of trying to have
all javascript files in the `js` subdirectory.  But that meant
that the worker could only cache the javascript files, breaking
the offline experience.


### The Service Worker

```javascript
const basePath = '{{ base-path }}'
const cacheName = '{{ base-path }}' // multiple sws from same origin!!! 
const appShellFiles = [
    '/', // we never load /index.html - always /
    '/css/bulma.min.css',
    '/js/main.js',
    '/js/register_sw.js',
    '/config.json',
    '/manifest.json',
    // no favicons!
]
const cacheContent = appShellFiles.map(v => basePath + v)

self.addEventListener('install', (e) => {
    console.log('[Service Worker] Install')
    e.waitUntil((async () => {
        const cache = await caches.open(cacheName)
        console.log('[Service Worker] Caching critical resources')
        await cache.addAll(cacheContent)
    })())
})

addEventListener('activate', e => {
    self.clients.claim()
})

// stale while revalidate from https://web.dev/learn/pwa/serving/
// for ease of update
self.addEventListener("fetch", event => {
    event.respondWith(
        caches.match(event.request).then(cachedResponse => {
            const networkFetch = fetch(event.request).then(response => {
                // update the cache with a clone of the network response
                caches.open(cacheName).then(cache => {
                    cache.put(event.request, response.clone());
                });
            });
            // prioritize cached response over network
            return cachedResponse || networkFetch;
        }
        )
    )
});
```

Again, that base path is important.  Also for these example deployments, there will be
one service worker for every part I publish.  The caches need to be different,
that's why I use the base path as cache name as well.

Note that I am using `stale-while-revalidate` here, this allows me to get a new
version by reloading twice.  On a phone, your possibilites are limited,
I've had to resort to deleting the site data (or all of Chrome's data) multiple times
when I tried to use `cache first`.

I am not showing the tooling, but the templates are handled by this script:
[generate_sw.clj][generate_sw.clj]

### Conclusion

I am very pleased with how this turned out.  The web app works well on my phone, it works
offline, and it can even be installed.  If installed, startup is very fast and I can start it while offline.

In the futre, I might replace `register_sw.js` with equivalent clojure code in the main application.
In particular, when online there could be periodic checks for new
versions and a popup or button to install that version.  That would enable us
to go back to a `cache first` strategy which was a bit faster.


### Tips and Tricks

* Use Chrome for debugging service workers
* The best docs I found for PWAs are here: [Learn PWA][learn_pwa]
* Use the `Lighthouse` tool in the developer console.  It will
  also audit PWAs, and several times this is what let me find
  the current problem.
* `SHIFT APPLE R` or `SHIFT F5` is your friend.  Once a service worker is active,
  you will get a lot of blank pages.  `SHIFT + reload` bypasses the service worker,
  at least once, and hopefully your fixed worker will be re-installed.
* If that fails, in desktop browser deveopment tools there usally is a
  "Application" section, and it might have a button to upgrade or remove
  the current service worker.
* On the phone, if the app is installed, you can long-press the icon
  for an option to clear the site settings.  This is another way
  of getting rid of a bad version and/or service worker.
* In Android App settings for chrome, you can delete all of chrome's data.
* If you never see a blank page, your service worker is probably not active,
  probably because it is not in scope

## Release Build on GH Pages

* [Demo: Learn you a Kee-Frame, Part 4][lyakf_part4]
* [Source code][lyakf_part4_source]

216 KB compressed. I was worried about the size of the generated
javascript, but I found no easy wins except using `preact` instead of `react`.
This saves 35 KB compressed.


[workbox]: https://web.dev/learn/pwa/workbox/
[favicon-generator]: https://favicon.io/favicon-generator/
[kf_110]: https://github.com/ingesolvoll/kee-frame/pull/110
[generate_sw.clj]: https://github.com/grmble/learn-you-a-keeframe/blob/part4/scripts/generate_sw.clj
[learn_pwa]: https://web.dev/learn/pwa/
[lyakf_part4]: https://grmble.github.io/learn-you-a-keeframe/part4/
[lyakf_part4_source]: https://github.com/grmble/learn-you-a-keeframe/tree/part4
