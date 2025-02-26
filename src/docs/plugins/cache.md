---
eleventyNavigation:
  key: Cache Assets
  order: 0.5
  excerpt: A utility to fetch and cache network requests.
---
# Cache Assets

Fetch network resources and cache them so you don’t bombard your API (or other resources). Do this at configurable intervals—not with every build! Once per minute, or once per hour, once per day, or however often you like!

With the added benefit that if one successful request completes, you can now work offline!

This plugin can save *any* kind of asset—JSON, HTML, images, videos, etc.

* Fetch a remote URL and saves it to a local cache.
* If the remote server goes down or linkrots away—we keep and continue to use the local asset (save remote images!)
* If cache expires and the network connection fails, will continue to use the cached request and make a new request when the network connectivity is restored.
* Control concurrency so we don’t make too many network requests at the same time.
* Requires **Node 10+**
* [`eleventy-cache-assets` on GitHub](https://github.com/11ty/eleventy-cache-assets)

---

[[toc]]

## Installation

* [`eleventy-cache-assets` on npm](https://www.npmjs.com/package/@11ty/eleventy-cache-assets)

```
npm install @11ty/eleventy-cache-assets
```

{% callout "warn" %}<strong>Important Security and Privacy Notice</strong>
<p>
  This plugin caches complete network responses. Unless you’re willing to perform a full review of everything this plugin caches to disk for privacy and security exposure, it is <em>strongly</em> recommended that you add the <code>.cache</code> folder to your <code>.gitignore</code> file so that network responses aren’t checked in to your git repository.
</p>
<p>
  Are you 100% sure that private e-mail addresses aren’t being returned from a cached API? I’m guessing no—add <code>.cache</code> to your <code>.gitignore</code> file. Right now. Do it.
</p>
{% endcallout %}

## Usage

### Cache a JSON file from an API

Consider the following example, perhaps in an Eleventy [Global Data File](/docs/data-global/).

```js
const Cache = require("@11ty/eleventy-cache-assets");

module.exports = async function() {
  let url = "https://api.github.com/repos/11ty/eleventy";

  /* This returns a promise */
  return Cache(url, {
    duration: "1d", // save for 1 day
    type: "json"    // we’ll parse JSON for you
  });
};
```

### Options

#### Change the Cache Duration

After this amount of time has passed, we’ll make a new network request to the URL to fetch fresh data. Use `duration: "*"` to never fetch new data. The `duration` option also currently supports the following shorthand values:

* `s` is seconds (e.g. `duration: "43s"`)
* `m` is minutes (e.g. `duration: "2m"`)
* `h` is hours (e.g. `duration: "99h"`)
* `d` is days
* `w` is weeks, or shorthand for 7 days (e.g. `duration: 2w` is 14 days)
* `y` is years, or shorthand for 365 days (not _exactly_ one year) (e.g. `duration: 2y` is 730 days)

#### Type

* `type: "json"`
* `type: "text"`
* `type: "buffer"` (default: use this for non-text things)

#### Cache Directory

The `directory` option let’s you change where the cache is stored. It is strongly recommended that you add this folder to your `.gitignore` file.

{% callout "warn" %}Read the <a href="#installation">Important Security and Privacy Notice</a>.{% endcallout %}

```js
const Cache = require("@11ty/eleventy-cache-assets");

Cache("https://…", {
	directory: ".cache"
});
```

{% callout "info" %}Eleventy Cache Assets can work inside of a Netlify Function (or AWS Lambda) by using <code>directory: "/tmp/.cache/"</code>.{% endcallout %}

#### Remove URL query params from Cache Identifier

(Version 2.0.3 and newer) If your fetched URL contains some query parameters that aren’t relevant to the identifier used in the cache, remove them using the `removeUrlQueryParams` option. This is useful if an API adds extra junk to your request URLs.

* `removeUrlQueryParams: true` (`false` is default)

```js
const Cache = require("@11ty/eleventy-cache-assets");

Cache("https://www.zachleat.com/img/avatar-2017-big.png?Get=rid&of=these", {
	removeUrlQueryParams: true
});
```


## What happens with a request fails?

1. If this is the first ever request to this URL (no entry exists in your cache folder), it will fail. Use a `try`/`catch` if you’d like to handle this gracefully.
2. If a failure happens and a cache entry already exists (*even if it’s expired*), it will use the cached entry.

```js
const Cache = require("@11ty/eleventy-cache-assets");

module.exports = async function() {
	try {
    let url = "https://api.github.com/repos/11ty/eleventy";

		/* This returns a promise */
		return Cache(url, {
			duration: "1d",
			type: "json"
		});
	} catch(e) {
		return {
			// my failure fallback data
		}
	}
};
```

## Running this on your Build Server

If you’re attempting to use this plugin on a service like Netlify and you are definitely not checking in your `.cache` folder to `git`, note that the cache will be empty with every build and new requests will go out. It’s important to be aware of this, even if it’s what you want for your production build!

## More Examples

### Cache a Remote Image

This is what [`eleventy-img`](/docs/plugins/image/) uses internally.

```js
const Cache = require("@11ty/eleventy-cache-assets");

module.exports = async function() {
  let url = "https://www.zachleat.com/img/avatar-2017-big.png";
  let imageBuffer = await Cache(url, {
    duration: "1d",
    type: "buffer"
  });
  // Use imageBuffer as an input to the `sharp` plugin, for example

  // (Example truncated)
}
```

### Fetch Google Fonts CSS

Also a good example of using `fetchOptions` to pass in a custom user agent. Full option list is available on the [`node-fetch` documentation](https://www.npmjs.com/package/node-fetch#options).

```js
const Cache = require("@11ty/eleventy-cache-assets");

let url = "https://fonts.googleapis.com/css?family=Roboto+Mono:400&display=swap";
let fontCss = await Cache(url, {
	duration: "1d",
	type: "text",
	fetchOptions: {
		headers: {
			// lol
			"user-agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_5) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/74.0.3729.169 Safari/537.36"
		}
	}
});
```

### Fetching GitHub Stars for a repo

* This specific example has been previously described in our quick tips section: head over to read [Quick Tip #009—Cache Data Requests](/docs/quicktips/cache-api-requests/).

### Manually store your own data in the cache

**You probably won’t need to do this.** If you’d like to store data of your own choosing in the cache (some expensive thing, but perhaps not related to a network request), you may do so! Consider the following [Global Data File](/docs/data-global/):

```js
const {AssetCache} = require("@11ty/eleventy-cache-assets");

module.exports = async function() {
  // Pass in your unique custom cache key
  // (normally this would be tied to your API URL)
  let asset = new AssetCache("zachleat_twitter_followers");

  // check if the cache is fresh within the last day
  if(asset.isCacheValid("1d")) {
    // return cached data.
    return asset.getCachedValue(); // a promise
  }

  // do some expensive operation here, this is simplified for brevity
  let fakeTwitterApiContents = { followerCount: 1000 };

  await asset.save(fakeTwitterApiContents, "json");

  return fakeTwitterApiContents;
};
```

### Change Global Plugin Concurrency

```js
const Cache = require("@11ty/eleventy-cache-assets");
Cache.concurrency = 4; // default is 10
```

### Command line debug output

```js
DEBUG=EleventyCacheAssets* node your-node-script.js
DEBUG=EleventyCacheAssets* npx @11ty/eleventy
```
