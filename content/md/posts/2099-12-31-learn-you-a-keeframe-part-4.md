{:title "Learn You A Kee-Frame, Part 4"
 :layout :post
 :tags  ["clojure" "lyakf"]}

The original plan was to make a web app,
then convert it to React Native to see how bad
that switch is.

But as I read up about the options I realized that
"Progressive Web Apps" are pretty powerful these
days, so I thought I would give that a spin first.

## Making a Progressive Web App

In one word: TEDIOUS.

Mistake #1, I started with MDN which usually works out well for me.
But apparently Mozilla is not as hot about PWAs as they used to be,
and lets just say that I found much better docs elsewhere.

Mistake #2, I use firefox - I would not say I trust Mozilla,
but I trust them a lot more than Google, no way I would store
my passwords in Chrome.  But did I mention that Mozilla is not
really into PWAs anymore?   They removed the install feature
from Firefox, which explains why I could not make it work.

Mistake #3, I did not use [Workbox][workbox].  I figured it was too much
effort to bring javascript tooling into the build.  So I spent
all day pressing reload and trying to guess why it was not working.
Spending that time to integrate existing tooling would have been
much faster.

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

Also this is only for release builds.  You generally have to
reload twice to get changed files through your service worker,
this does not play nice with hot reloading :)

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





* Use Google Chrome
* LIGHTHOUSE!
* SHIFT APPLE R is your friend - once sw actually active, you will get a lot
of blank pages.  SHIFT + reload bypasses the service worker, at least once,
and hopefully a fixed one will get re-installed
* Learn PWA 




[workbox]: https://web.dev/learn/pwa/workbox/
[favicon-generator]: https://favicon.io/favicon-generator/
[kf_110]: https://github.com/ingesolvoll/kee-frame/pull/110
