---
title: Presets (Unstable)
---

# Presets (Unstable)

The [Remix Vite plugin][remix-vite] supports a `presets` option to ease integration with other tools and hosting providers.

Presets can only do two things:

- Configure the Remix Vite plugin on your behalf.
- Validate the resolved config.

The config returned by each preset is merged in the order they were defined. Any config directly passed to the Remix Vite plugin will be merged last. This means that user config will always take precedence over any presets.

## Using a preset

Presets are designed to be published to npm and used within your Vite config. For example, Remix ships with a preset for Cloudflare:

```ts filename=vite.config.ts lines=[3,10]
import {
  unstable_vitePlugin as remix,
  unstable_cloudflarePreset as cloudflare,
} from "@remix-run/dev";
import { defineConfig } from "vite";

export default defineConfig({
  plugins: [
    remix({
      presets: [cloudflare()],
    }),
  ],
  // etc.
});
```

## Creating a preset

Presets conform to the following `Unstable_Preset` type:

```ts
type Unstable_Preset = {
  name: string;

  remixConfig?: () =>
    | RemixConfigPreset
    | Promise<RemixConfigPreset>;

  remixConfigResolved?: (args: {
    remixConfig: ResolvedVitePluginConfig;
  }) => void | Promise<void>;
};
```

### Defining preset config

As a basic example, let's create a preset that configures a [server bundles function][server-bundles]:

```ts filename=my-cool-preset.ts
import type { Unstable_Preset as Preset } from "@remix-run/dev";

export function myCoolPreset(): Preset {
  return {
    name: "my-cool-preset",
    remixConfig: () => ({
      serverBundles: ({ branch }) => {
        const isAuthenticatedRoute = branch.some((route) =>
          route.id.split("/").includes("_authenticated")
        );

        return isAuthenticatedRoute
          ? "authenticated"
          : "unauthenticated";
      },
    }),
  };
}
```

### Validating config

It's important to remember that other presets and user config can still override the values returned from your preset.

In our example preset, the `serverBundles` function could be overridden with a different, conflicting implementation. If we want to validate that the final resolved config contains the `serverBundles` function from our preset, we can do this with the `remixConfigResolved` hook:

```ts filename=my-cool-preset.ts lines=[22-26]
import type {
  Unstable_Preset as Preset,
  Unstable_ServerBundlesFunction as ServerBundlesFunction,
} from "@remix-run/dev";

const serverBundles: ServerBundlesFunction = ({
  branch,
}) => {
  const isAuthenticatedRoute = branch.some((route) =>
    route.id.split("/").includes("_authenticated")
  );

  return isAuthenticatedRoute
    ? "authenticated"
    : "unauthenticated";
};

export function myCoolPreset(): Preset {
  return {
    name: "my-cool-preset",
    remixConfig: () => ({ serverBundles }),
    remixConfigResolved: ({ remixConfig }) => {
      if (remixConfig.serverBundles !== serverBundles) {
        throw new Error("`serverBundles` was overridden!");
      }
    },
  };
}
```

The `remixConfigResolved` hook should only be used in cases where it would be an error to merge or override your preset's config.

[remix-vite]: ./vite
[server-bundles]: ./server-bundles
