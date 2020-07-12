# README

Taken from [this YT
tutorial](https://www.youtube.com/watch?v=IaJqMcOMuDM)

More Links:

  * [The Offline
    Cookbook](https://developers.google.com/web/fundamentals/instant-and-offline/offline-cookbook)

  * [MDN docs on Caching
    API](https://developer.mozilla.org/en-US/docs/Web/API/Cache) 


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


## Caching Strategies

#### Network-only

Useful when you can't perform it offline.

Pretty much all non-GET requests can be covered by this.

```javascript
self.addEventListener('fetch', function(event) {
  event.respondWith(fetch(event.request));
});
```


#### Cache falling back to network

Best for offline-first applications.

You'll use exceptions based on the incoming request

```javascript
self.addEventListener('fetch', function(event) {
  event.respondWith(
    caches.match(event.request).then(function(response) {
      // this is key -- response can be null
      return response || fetch(event.request);
    })
  );
});
```

#### Network falling back to the cache

Useful for when resources are updated frequently and not part of a
"version" of the website applications.

This includes articles, avatars, social media timelines.

Users will get the most up-to-date when possible, othewise an older
cached version.

```javascript
self.addEventListener('fetch', function(event) {
  event.respondWith(
    fetch(event.request).catch(function() {
      return caches.match(event.request);
    })
  );
});
```

#### Cache then network

This approach will respond first with the cache, and then update with
the fresher content once that arrives.

```javascript
var networkDataReceived = false;

startSpinner();

// fetch fresh data
var networkUpdate = fetch('/data.json').then(function(response) {
  return response.json();
}).then(function(data) {
  networkDataReceived = true;
  updatePage(data);
});

// fetch cached data
caches.match('/data.json').then(function(response) {
  if (!response) throw Error("No data");
  return response.json();
}).then(function(data) {
  // don't overwrite newer network data
  if (!networkDataReceived) {
    updatePage(data);
  }
}).catch(function() {
  // we didn't get cached data, the network is our last hope:
  return networkUpdate;
}).catch(showErrorMessage).then(stopSpinner());
```



```javascript
// service worker
self.addEventListener('fetch', function(event) {
  event.respondWith(
    caches.open('mysite-dynamic').then(function(cache) {
      return fetch(event.request).then(function(response) {
        cache.put(event.request, response.clone());
        return response;
      });
    })
  );
});
```

#### Generic FAllback when failures

We can use this for generic fallbacks such as mising avatars, failed
POST requests, "Unavailable while offline" type pages. 

```javascript
self.addEventListener('fetch', function(event) {
  event.respondWith(
    // Try the cache
    caches.match(event.request).then(function(response) {
      // Fall back to network
      return response || fetch(event.request);
    }).catch(function() {
      // If both fail, show a generic fallback:
      return caches.match('/offline.html');
      // However, in reality you'd have many different
      // fallbacks, depending on URL & headers.
      // Eg, a fallback silhouette image for avatars.
    })
  );
});
```

Alternatively in case of network failure

```javascript
self.addEventListener('fetch', function(event) {
  event.respondWith(
    // Try the cache
    caches.match(event.request).then(function(response) {
      if (response) {
        return response;
      }
      return fetch(event.request).then(function(response) {
        if (response.status === 404) {
          return caches.match('pages/404.html');
        }
        return response
      });
    }).catch(function() {
      // If both fail, show a generic fallback:
      return caches.match('/offline.html');
    })
  );
});
```


#### Removing outdates caches

You'll want to use the activate event in order to remove older caches
once we have different versions of the service worker (i.e. app) in
place.

```javascript
self.addEventListener('activate', function(event) {
  event.waitUntil(
    caches.keys().then(function(cacheNames) {
      return Promise.all(
        cacheNames.filter(function(cacheName) {
          // Return true if you want to remove this cache,
          // but remember that caches are shared across
          // the whole origin
        }).map(function(cacheName) {
          return caches.delete(cacheName);
        })
      );
    })
  );
});
```

### Cache API

Check if browser supports: ``'caches' in window``

Creating the cache: ``caches.open(cacheName)``

Creating data:

```javascript
// first we open the cache object, which returns the cache as a promise
caches.open(cacheName).then(cache => {
  // then we add a URL
  // the cache retrieves it and adds the resulting object to the given cache
  cache.add("example-file.html");

  // alternatively, you can do this if you want to modify the responseobject
  fetch(url).then(resp => {
    return cache.put(url, resp);
  });
});
```

Matching data: ``caches.match(request, options)``. Options include:

  * ignoreSearch: boolean. if true, ignores params passed in url.
  * ignoreMethod: boolean. if true, allows for non-GET/HEAD methods.
  * ignoreVary: boolean. something about the VARY header?
  * cacheName: string that specifies a specific cache to search within.

``caches.matchAll(request, options)`` is the same as
``caches.match(...)`` except it returns all responses from the cache
instead of just the first one.

Deleting data: ``cache.delete(request, options)``.









