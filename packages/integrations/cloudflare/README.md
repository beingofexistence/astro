# @astrojs/cloudflare

An SSR adapter for use with Cloudflare Pages Functions targets. Write your code in Astro/Javascript and deploy to Cloudflare Pages.

## Install

Add the Cloudflare adapter to enable SSR in your Astro project with the following `astro add` command. This will install the adapter and make the appropriate changes to your `astro.config.mjs` file in one step.

```sh
# Using NPM
npx astro add cloudflare
# Using Yarn
yarn astro add cloudflare
# Using PNPM
pnpm astro add cloudflare
```

If you prefer to install the adapter manually instead, complete the following two steps:

1. Add the Cloudflare adapter to your project's dependencies using your preferred package manager. If you’re using npm or aren’t sure, run this in the terminal:

```bash
npm install @astrojs/cloudflare
```

2. Add the following to your `astro.config.mjs` file:

```js ins={3, 6-7}
// astro.config.mjs
import { defineConfig } from 'astro/config';
import cloudflare from '@astrojs/cloudflare';

export default defineConfig({
  output: 'server',
  adapter: cloudflare(),
});
```

## Options

### Mode

`mode: "advanced" | "directory"`

default `"advanced"`

Cloudflare Pages has 2 different modes for deploying functions, `advanced` mode which picks up the `_worker.js` in `dist`, or a directory mode where pages will compile the worker out of a functions folder in the project root. For most projects the adapter default of `advanced` will be sufficient; the `dist` folder will contain your compiled project.

#### `mode:directory`

Switching to directory mode allows you to use [pages plugins](https://developers.cloudflare.com/pages/platform/functions/plugins/) such as [Sentry](https://developers.cloudflare.com/pages/platform/functions/plugins/sentry/) or write custom code to enable logging.

```ts
// astro.config.mjs
export default defineConfig({
  adapter: cloudflare({ mode: 'directory' }),
});
```

In `directory` mode, the adapter will compile the client-side part of your app the same way as in `advanced` mode by default, but moves the worker script into a `functions` folder in the project root. In this case, the adapter will only ever place a `[[path]].js` in that folder, allowing you to add additional plugins and pages middleware which can be checked into version control.

To instead compile a separate bundle for each page, set the `functionPerPath` option in your Cloudflare adapter config. This option requires some manual maintenance of the `functions` folder. Files emitted by Astro will overwrite existing `functions` files with identical names, so you must choose unique file names for each file you manually add. Additionally, the adapter will never empty the `functions` folder of outdated files, so you must clean up the folder manually when you remove pages.

```diff
import {defineConfig} from "astro/config";
import cloudflare from '@astrojs/cloudflare';

export default defineConfig({
     adapter: cloudflare({
        mode: 'directory',
+       functionPerRoute: true
    })
})
```

Note that this adapter does not support using [Cloudflare Pages Middleware](https://developers.cloudflare.com/pages/platform/functions/middleware/). Astro will bundle the [Astro middleware](https://docs.astro.build/en/guides/middleware/) into each page.

## Enabling Preview

In order for preview to work you must install `wrangler`

```sh
pnpm install wrangler --save-dev
```

It's then possible to update the preview script in your `package.json` to `"preview": "wrangler pages dev ./dist"`. This will allow you to run your entire application locally with [Wrangler](https://github.com/cloudflare/wrangler2), which supports secrets, environment variables, KV namespaces, Durable Objects and [all other supported Cloudflare bindings](https://developers.cloudflare.com/pages/platform/functions/#adding-bindings).

## Access to the Cloudflare runtime

You can access all the Cloudflare bindings and environment variables from Astro components and API routes through `Astro.locals`.

If you're inside an `.astro` file, you access the runtime using the `Astro.locals` global:

```astro
const env = Astro.locals.runtime.env;
```

From an endpoint:

```js
// src/pages/api/someFile.js
export function get(context) {
  const runtime = context.locals.runtime;

  return new Response('Some body');
}
```

Depending on your adapter mode (advanced = worker, directory = pages), the runtime object will look a little different due to differences in the Cloudflare API.

If you're using the `advanced` runtime, you can type the `runtime` object as following:

```ts
// src/env.d.ts
/// <reference types="astro/client" />
import type { AdvancedRuntime } from '@astrojs/cloudflare';

declare namespace App {
  interface Locals extends AdvancedRuntime {
    user: {
      name: string;
      surname: string;
    };
  }
}
```

If you're using the `directory` runtime, you can type the `runtime` object as following:

```ts
// src/env.d.ts
/// <reference types="astro/client" />
import type { DirectoryRuntime } from '@astrojs/cloudflare';

declare namespace App {
  interface Locals extends DirectoryRuntime {
    user: {
      name: string;
      surname: string;
    };
  }
}
```

## Environment Variables

See Cloudflare's documentation for [working with environment variables](https://developers.cloudflare.com/pages/platform/functions/bindings/#environment-variables).

```js
// pages/[id].json.js

export function get({ params }) {
  // Access environment variables per request inside a function
  const serverUrl = import.meta.env.SERVER_URL;
  const result = await fetch(serverUrl + "/user/" + params.id);
  return {
    body: await result.text(),
  };
}
```

## Headers, Redirects and function invocation routes

Cloudflare has support for adding custom [headers](https://developers.cloudflare.com/pages/platform/headers/), configuring static [redirects](https://developers.cloudflare.com/pages/platform/redirects/) and defining which routes should [invoke functions](https://developers.cloudflare.com/pages/platform/functions/routing/#function-invocation-routes). Cloudflare looks for `_headers`, `_redirects`, and `_routes.json` files in your build output directory to configure these features. This means they should be placed in your Astro project’s `public/` directory.

### Custom `_routes.json`

By default, `@astrojs/cloudflare` will generate a `_routes.json` file with `include` and `exclude` rules based on your applications's dynamic and static routes.
This will enable Cloudflare to serve files and process static redirects without a function invocation. Creating a custom `_routes.json` will override this automatic optimization and, if not configured manually, cause function invocations that will count against the request limits of your Cloudflare plan.

See [Cloudflare's documentation](https://developers.cloudflare.com/pages/platform/functions/routing/#create-a-_routesjson-file) for more details.

## Troubleshooting

For help, check out the `#support` channel on [Discord](https://astro.build/chat). Our friendly Support Squad members are here to help!

You can also check our [Astro Integration Documentation][astro-integration] for more on integrations.

### Meaningful error messages

Currently, errors during running your application in Wrangler are not very useful, due to the minification of your code. For better debugging, you can add `vite.build.minify = false` setting to your `astro.config.js`

```js
export default defineConfig({
  adapter: cloudflare(),
  output: 'server',

  vite: {
    build: {
      minify: false,
    },
  },
});
```

## Contributing

This package is maintained by Astro's Core team. You're welcome to submit an issue or PR!

[astro-integration]: https://docs.astro.build/en/guides/integrations-guide/
