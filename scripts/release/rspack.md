# Standalone react-server-dom-rspack releases

`react-server-dom-rspack` is published independently while its React integration
is under review in https://github.com/react/react/pull/35473. Its npm version is
independent of the embedded React renderer version. Keep its entry in
`ReactVersions.js` and `packages/react-server-dom-rspack/package.json` in sync.

## 0.1.1 security backport

The published 0.1.0 package has `gitHead`
`f148045f80fb8841f3a9b098cda2aeaa7a20bb69`. This release starts from that commit
and backports the runtime changes and regression tests from React commit
`61aa00df822d0eb76d0362eb8b6fdd3527693efe` (`v19.3.1`, #37845), addressing
https://github.com/react/react/security/advisories/GHSA-xqc2-xf4c-x3h3.

The backport updates shared Flight code, rather than copying compiled webpack
bundles or replacing Rspack's manifest, reference and Server Action integration.
It includes typed-array ownership and detached-buffer checks, private response
state, server-reference validation, and separated stream/FormData storage.
The renderer remains based on React 19.3.0; this is not a full React 19.3.1 merge.
The React, React DOM and Rspack peer ranges match the published 0.1.0 package.

Streamed reply fields now use an `S` namespace. Deploy the client and server
packages together and invalidate cached client assets to avoid mixing the old
and new wire formats.

## Build and validate

Use Node.js 24 and a Java runtime supported by the repository's Closure compiler.
Install dependencies with `yarn install --frozen-lockfile`. From the repository
root, run:

```sh
yarn test --releaseChannel stable --runInBand ReactFlightRspack ReactFlightDOMRspack
yarn test --releaseChannel stable --prod --runInBand ReactFlightRspack ReactFlightDOMRspack
RELEASE_CHANNEL=stable node scripts/rollup/build.js react-server-dom-rspack --type=NODE_DEV,NODE_PROD
```

The last command rebuilds `build/` and produces all twelve development and
production bundles in `build/node_modules/react-server-dom-rspack`. Use the
direct Rollup entry point here: the multi-package `yarn build` wrapper rewrites
React peer versions for a coordinated React release.

After committing and reviewing the source changes, create and inspect the npm
archive:

```sh
cd build/node_modules/react-server-dom-rspack
npm pkg set gitHead="$(git rev-parse HEAD)"
npm pack --dry-run
npm pack
```

Check that the archive version is `0.1.1`, the peer ranges match 0.1.0, all
browser/edge/node entry points resolve, and both development and production
bundles contain the security fixes. Test the packed archive in a consumer with
the supported React versions before publishing.

The package tests can also target a separately installed archive. Install it
alongside `react` and `react-dom` in a temporary consumer directory, then run
from the repository root:

```sh
IS_BUILD=1 RSPACK_PACKAGE_DIRECTORY=/absolute/consumer/node_modules/react-server-dom-rspack yarn test --releaseChannel stable --runInBand ReactFlightDOMRspackPackage
```

These tests check exports and binary transport in all three runtimes, in both
development and production. For this release, use React 19.1.0 and 19.3.1 as the
consumer baselines. The binary security tests should fail against the published
0.1.0 archive and pass against 0.1.1.

## Publish the reviewed archive

Run from the same directory, using an npm account with package publish access:

```sh
npm publish ./react-server-dom-rspack-0.1.1.tgz --tag latest --access public
npm view react-server-dom-rspack@0.1.1 version gitHead dist.integrity
npm view react-server-dom-rspack dist-tags
```

For later security updates, resolve the advisory to its upstream commit, apply
the complete runtime patch and its regression tests to the published source
baseline, and rerun source and packed-package validation. A package version
bump or a change only under `packages/react-server-dom-rspack/` is insufficient
because the Flight implementation is compiled from shared packages.
