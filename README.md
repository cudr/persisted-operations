# @graphile/persisted-operations

Persisted operations (aka "persisted queries", "query allowlist") support for
PostGraphile's GraphQL server. Applies to both standard POST requests and to
websocket connections.

See
[Production Considerations](https://www.graphile.org/postgraphile/production/)
for why you might want this.

We recommend that all GraphQL servers (PostGraphile or otherwise) that only
intend first party clients to use the GraphQL schema should use persisted
operations to mitigate attacks against the GraphQL API and to help track the
fields that have been used. This package is our solution for PostGraphile's
server, for other servers you will need different software.

<!-- SPONSORS_BEGIN -->

## Crowd-funded open-source software

To help us develop software sustainably under the MIT license, we ask all
individuals and businesses that use our OSS to help support ongoing maintenance
and development via sponsorship.

### [Click here to find out more about sponsors and sponsorship.](https://www.graphile.org/sponsor/)

And please give some love to our featured sponsors 🤩:

<table><tr>
<td align="center"><a href="http://chads.website"><img src="https://graphile.org/images/sponsors/chadf.png" width="90" height="90" alt="Chad Furman" /><br />Chad Furman</a> *</td>
<td align="center"><a href="https://storyscript.com/?utm_source=postgraphile"><img src="https://graphile.org/images/sponsors/storyscript.png" width="90" height="90" alt="Storyscript" /><br />Storyscript</a> *</td>
<td align="center"><a href="https://postlight.com/?utm_source=graphile"><img src="https://graphile.org/images/sponsors/postlight.jpg" width="90" height="90" alt="Postlight" /><br />Postlight</a> *</td>
</tr></table>

<em>\* Sponsors the entire Graphile suite</em>

<!-- SPONSORS_END -->

## Usage

_See: [Server Plugins](https://www.graphile.org/postgraphile/plugins/)._

PostGraphile CLI:

```
postgraphile \
  --plugins @graphile/persisted-operations \
  --persisted-operations-directory path/to/folder/ \
  ...
```

PostGraphile library:

```
import { postgraphile, makePluginHook } from "postgraphile";
import PersistedOperationsPlugin from "@graphile/persisted-operations";

const pluginHook = makePluginHook([PersistedOperationsPlugin]);

const postGraphileMiddleware = postgraphile(databaseUrl, "app_public", {
  pluginHook,
  persistedOperationsDirectory: `${__dirname}/.persisted_queries/`,
});
```

### Library Options

This plugin adds the following options to the PostGraphile library options:

```
/**
 * This function will be passed a GraphQL request object (normally `{query:
 * string, variables?: any, operationName?: string, extensions?: any}`, but
 * in the case of persisted operations it likely won't have a `query`
 * property), and must extract the hash to use to identify the persisted
 * operation. For Apollo Client, this might be something like:
 * `request?.extensions?.persistedQuery?.sha256Hash`
 */
hashFromPayload?(request: any): string;

/**
 * We can read persisted operations from a folder (they must be named
 * `<hash>.graphql`); this is mostly used for PostGraphile CLI. When used
 * in this way, the first request for a hash will read the file
 * synchronously, and then the result will be cached such that the
 * **synchronous filesystem read** will only impact the first use of that
 * hash. We periodically scan the folder for new files, requests for hashes
 * that were not present in our last scan of the folder will be rejected to
 * mitigate denial of service attacks asking for non-existent hashes.
 */
persistedOperationsDirectory?: string;

/**
 * An optional string-string key-value object defining the persisted
 * operations, where the keys are the hashes, and the values are the
 * operation document strings to use.
 */
persistedOperations?: { [hash: string]: string };

/**
 * If your known persisted queries may change over time, or you'd rather
 * load them on demand, you may supply this function. Note this function is
 * both **synchronous** and **performance critical** so you should use
 * caching to improve performance of any follow-up requests for the same
 * hash. This function is not suitable for fetching operations from remote
 * stores (e.g. S3).
 */
persistedOperationsGetter?: PersistedOperationGetter;
```

All these options are optional; but you should specify exactly one of
`persistedOperationsDirectory`, `persistedOperations` or
`persistedOperationsGetter` for this plugin to be useful.

## Generating Persisted Operations

### GraphQL-Code-Generator

We recommend that you generate persisted queries when you build your clients
using the awesome
[graphql-code-generator](https://github.com/dotansimha/graphql-code-generator)
project and the
[graphql-codegen-persisted-query-ids](https://www.npmjs.com/package/graphql-codegen-persisted-query-ids)
(tested with v0.1.2) plugin. A config might look like:

```yaml
schema: "schema.graphql"
documents: "src/**/*.graphql"
hooks:
  afterAllFileWrite:
    - node addToPersistedQueries.js
generates:
  client.json:
    plugins:
      - graphql-codegen-persisted-query-ids:
          output: client
          algorithm: sha256
  server.json:
    plugins:
      - graphql-codegen-persisted-query-ids:
          output: server
          algorithm: sha256
```

I also use the file `addToPersistedQueries.js` to write the `server.json` file
contents out to separate GraphQL files every time the code is built for easier
version control:

```js
// addToPersistedQueries.js
const map = require("./server.json");
const { promises: fsp } = require("fs");

async function main() {
  await Promise.all(
    Object.entries(map).map(([hash, query]) =>
      fsp.writeFile(`${__dirname}/.persisted_queries/${hash}.graphql`, query)
    )
  );
}

main().catch((e) => {
  console.error(e);
  process.exit(1);
});
```

Then you pass the `.persisted_queries` folder to PostGraphile:

```
postgraphile \
  --plugins @graphile/persisted-operations \
  --persisted-operations-directory $(pwd)/.persisted_queries/ \
  -c mydb
```
