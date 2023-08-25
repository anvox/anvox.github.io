---
layout: post
title: Config Sentry with NextJS and ApolloJS
date: 2023-08-25 00:55 +0700
categories: sentry nextjs apollojs
---

While integrating Sentry into a NextJS project quite straightforward, just follow their [guide](https://docs.sentry.io/platforms/javascript/guides/nextjs/manual-setup/), it's attempted to strip all the complex config to quickly send error to Sentry. I mean, just 3 block of codes, then errors are sent to Sentry.

```typescript
// both sentry.client.config.js and sentry.server.config.ts
import * as Sentry from '@sentry/nextjs';

if (process.env.NEXT_PUBLIC_SENTRY_DSN) {
  Sentry.init({
    release: `project-name@${process.env.NEXT_PUBLIC_VERSION}`,
    enabled: true,
    dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
    environment: process.env.NEXT_PUBLIC_SENTRY_ENV,
  });
}

// next.config.js
const { withSentryConfig } = require('@sentry/nextjs');
// ...
  sentry: {
    disableClientWebpackPlugin: true,
    disableServerWebpackPlugin: true,
  },
// ...
module.exports = withSentryConfig(nextConfig);
```

It's cool, but useless. Yes, for error on node server, it may acceptable. But errors on client browsers are nearly useless. Without source map, minified stacktrace in error report like a forest. I have to admit that I never find out to fix any bug by guessing root cause from those minified source code.
Besides, we're using apollojs, which mean all the `fetch`ing from server will be shown as `fetch POST https://backend.com/api/graphql [200] 21:07:17` 🥲

We lived with it for years, until I was getting mad of tons of Sentry report which broken our package, but had no clues of what are they, how to fix them. After hour of googling and dive in Sentry documentation with tries, surprisingly, we was only one step away from the finish line. I dont get into detail on how do I tested them out, just all the config here to save future-me one day.

```typescript
// both sentry.client.config.js and sentry.server.config.ts
import * as Sentry from '@sentry/nextjs';
import { excludeGraphQLFetch } from 'apollo-link-sentry';

const SENTRY_DSN: string = process.env.NEXT_PUBLIC_SENTRY_DSN;

if (SENTRY_DSN) {
  Sentry.init({
    release: `project@${process.env.NEXT_PUBLIC_VERSION}`,
    enabled: true,
    dsn: SENTRY_DSN,
    environment: process.env.NEXT_PUBLIC_SENTRY_ENV,
    ignoreErrors: [
      'Validation failed',
      "Can't find variable: zaloJSV2",
    ],
    tracesSampleRate:
      process.env.NEXT_PUBLIC_SENTRY_ENV === 'production' ? 0.15 : 0,
    allowUrls: [/https?:\/\/backend.com/],
    beforeBreadcrumb: excludeGraphQLFetch,
  });
}


// next.config.js
const { withSentryConfig } = require('@sentry/nextjs');
// ... nextConfig
  sentry: {
    hideSourceMaps: true,
    widenClientFileUpload: true,
    disableServerWebpackPlugin: true,
  },
// ...
const sentryWebpackPluginOptions = {
  org: 'org',
  project: 'project',
  authToken: process.env.SENTRY_AUTH_TOKEN,
  silent: false, // Enable to debug when build and upload artifact
  release: `project@${process.env.NEXT_PUBLIC_VERSION}`,
};

module.exports = withSentryConfig(nextConfig, sentryWebpackPluginOptions);

// apollo client
  return new ApolloClient({
    link: ApolloLink.from([
      new SentryLink({
        setTransaction: false,
        setFingerprint: false,
        attachBreadcrumbs: { includeError: true },
      ]),
// ...
```

Some take away:

* `release` in `Sentry.init` must be the sames with in `sentryWebpackPluginOptions`, or you'll find 2 tracked versions on Sentry.
* `sentryWebpackPluginOptions` `silent` is useful when checking if source map is generated and pushed?
* Performance trace is run automatedly, adjust sample rate to match Sentry package capacity.
* Session Replay is an interesting feature, but only provided to business and higher packages.
