# README

Taken from [this YT tutorial](https://www.youtube.com/watch?v=IaJqMcOMuDM)


## Service Workers

The most basic service worker has `self.addEventListener("<name>",
event => { ...});`

principally has three functions

  * install
  * fetch
  * activate

Using the cacheDB (keyword `cache`), we can store static files like
json replies or html files.

When we install, we call `event.waitUntil(...)` to wrap all the logic
inside it. This will be async (I think?).

Then we will open up the cache name (`cache.open("name")`), use the
returned promise to then add all the URLS.

```javascript
const self = this;

self.addEventListener("install", event => {
  event.waitUntil(
    caches.open(CACHE_NAME).then(cache => {
      console.log("opened cache");
      return cache.addAll(urlsToCache);
    })
  );
});
```

We use the `fetch` event to reply to every event.

```javascript
// Listen for requests
self.addEventListener("fetch", event => {
  event.respondWith(
    caches.match(event.request).then(() => {
      // try to fetch the resource, fallback otherwise
      return fetch(event.request).catch(() => {
        caches.match("offline.html");
      });
    })
  );
});
```

Finally, we use an activate event for ... something?

```javascript
// Activate the SW
self.addEventListener("activate", event => {
  const cacheWhiteList = [];
  cacheWhiteList.push(CACHE_NAME);
  event.waitUntil(
    caches.keys().then(cacheNames =>
      Promise.all(
        cacheNames.map(cacheName => {
          // whenever we update something, we'll delete all the previous versions and
          // just keep the white listed version
          if (!cacheWhiteList.includes(cacheName)) {
            return caches.delete(cacheName);
          }
        })
      )
    )
  );
});
```


## Manifest.json file

Use double quotes as its json!

```
{
    "short_name": <name>,
    "name": <name>,
    "icons": [   # most important part for installation on mobile devices
        {
            "src": <path-relative-to-public>,
            "type": "image/png",
            "sizes": "1024x1024"
        }
    ],
    "start_url": ".",
    "display": "standalone",  # basic thing to add
    "theme_color": "#000000", # black
    "background_color": "#ffffff",  # white
}
```


## Using lighthouse to audit PWAs

This is recommended.

It's included as an extension on Google Chrome. [See
here](https://developers.google.com/web/tools/lighthouse/). 

Note you'll need to deploy using Netlify or some other service in
order to fully be okay as localhost is insecure (uses HTTP).
