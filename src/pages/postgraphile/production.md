---
layout: page
path: /postgraphile/production/
title: Production Considerations
---

## Production Considerations

When it comes time to deploy your PostGraphile application to production,
there's a few things you'll want to think about including topics such as
logging, security and stability. This article outlines some of the issues
you might face, and how to solve them.

### Database Access Considerations

PostGraphile is just a node app / middleware, so you can deploy it to any
number of places: Heroku, Now.sh, a VM, a container such as Docker, or of
course onto bare metal. Typically you won't run PostGraphile on the same
hardware/container/VM as the database, so PostGraphile needs to be able to
connect to your database without you putting your DB at risk.

A standard way of doing this is to put the DB behind a firewall. However,
if you're using a system like Heroku or Now.sh you probably can't do that,
so instead you must make your DB accessible to the internet. When doing so
here are a few things we recommend:

1.  Only allow connections over SSL (`force_ssl` setting)
2.  Use a secure username (not `root`, `admin`, `postgres`, etc which are all fairly commonly used)
3.  Use a super secure password; you can use a command like this to generate one:
    `openssl rand -base64 30 | tr '+/' '-_'`
4.  Use a non-standard port for your PostgreSQL server if you can (pick a random port number)
5.  Use a hard-to-guess hostname, and never reveal the hostname to anyone who doesn't need to know it
6.  If possible, limit the IP addresses that can connect to your DB to be just those of your hosting provider.

Heroku have some instructions on making RDS available for use under Heroku
which should also work for Now.sh or any other service:
https://devcenter.heroku.com/articles/amazon-rds

### Common Middleware Considerations

In a production app, you typically want to add a few common enhancements, e.g.

- Logging
- Gzip or similar compression
- Security protections
- Rate limiting

Since there's already a lot of options and opinions in this space, and they're
not directly related to the problem of serving GraphQL from your PostgreSQL
database, PostGraphile does not include these things by default. We recommend
that you use something like Express middlewares to implement these common
requirements. This is why we recommend [using PostGraphile as a
library](/postgraphile/usage-library/) for production usage.

Picking the Express (or similar) middlewares that work for you is beyond the
scope of this article; below is an example of where to place these middlewares.

```js
const express = require("express");
const { postgraphile } = require("postgraphile");

const app = express();

/* Example middleware you might want to put in front of PostGraphile */
// app.use(require('morgan')(...));
// app.use(require('compression')({...}));
// app.use(require('helmet')({...}));

app.use(postgraphile(process.env.DATABASE_URL || "postgres:///"));

app.listen(process.env.PORT || 3000);
```

Should you want to use something like PostGraphile's built in logging, but
send it to your own logging provider, you can compose a PostGraphile server
plugin to do so. Here's an example plugin that uses Nuxt's consola library
for logging, you could use it as a base for your own plugin:
https://github.com/graphile/postgraphile-log-consola

### Denial of Service Considerations

When you run PostGraphile in production you'll want to ensure that people
cannot easily trigger denial of service (DOS) attacks against you. Due to the
nature of GraphQL it's easy to construct a small query that could be very
expensive for the server to run, for example:

```graphql
allUsers {
  nodes {
    postsByAuthorId {
      nodes {
        commentsByPostId {
          userByAuthorId {
            postsByAuthorId {
              nodes {
                commentsByPostId {
                  userByAuthorId {
                    postsByAuthorId {
                      nodes {
                        commentsByPostId {
                          userByAuthorId {
                            id
                          }
                        }
                      }
                    }
                  }
                }
              }
            }
          }
        }
      }
    }
  }
}
```

There's lots of techniques for protecting your server from these kinds of
queries; a great introduction to this subject is [this blog
post](https://dev-blog.apollodata.com/securing-your-graphql-api-from-malicious-queries-16130a324a6b)
from Apollo.

These techniques should be used in conjunction with common HTTP protection
methods such as rate limiting which are typically better implemented at a
separate layer; for example you could use [Cloudflare rate
limiting](https://www.cloudflare.com/rate-limiting/) for this, or an
Express.js middleware.

#### Simple: Query Whitelist ("persisted queries")

If you do not intend to open your API up to third parties to run arbitrary
queries against then using persisted queries as a query whitelist to protect
your GraphQL endpoint is a great and highly recommended solution. This
technique ensures that only the queries you use in your own applications can
be executed on the server (but you can of course change the variables).

This technique has a few caveats:

- Your API will only accept queries that you've approved, so it's not suitable if you want third parties to run arbitrary queries
- You must be able to generate a unique ID from each query; e.g. a hash
- You must use "static GraphQL queries" - that is the queries must be known at build time of your application/webpage, and only the variables fed to those queries can change at run-time
- You must have a way of sharing these queries between the application and the server
- You must be careful not to use variables in dangerous places; for example don't write `allUsers(first: $myVar)` as a malicious attacker could set `$myVar` to 2147483647 in order to cause your server to process as much data as possible.

PostGraphile currently doesn't have this functionality built in, but it's
fairly easy to add it when using PostGraphile as an express middleware, a
simple implementation might look like this:

```js{9-18}
const postgraphile = require("postgraphile");
const express = require("express");
const bodyParser = require("body-parser");

const app = express();
app.use(bodyParser.json());

/**** BEGINNING OF CUSTOMIZATION ****/
const persistedQueries = require("./persistedQueries.json");

app.use("/graphql", async (req, res, next) => {
  // TODO: validate req.body is of the right form
  req.body.query = {}.hasOwnProperty.call(persistedQueries, req.body.id)
    ? persistedQueries[req.body.id]
    : null;
  next();
});
/**** END OF CUSTOMIZATION *** */

app.use(postgraphile());

app.listen(5000);
```

i.e. a simple middleware mounted before postgraphile that manipulates the request body.

I (Benjie) personally use my forks of Apollo's `persistgraphql` tools to help
me manage the persisted queries themselves:

- https://github.com/benjie/persistgraphql
- https://github.com/benjie/persistgraphql-webpack-plugin

These forks generate hashes rather than numbers; which make the persisted
queries consistent across multiple builds and applications (website, mobile,
browser plugin, ...).

**NOTE**: even if you're using persisted queries, it can still be wise to
implement advanced protections as it enables you to catch unnecessarily
expensive queries during development, before you start facing performance
bottlenecks down the line.

#### Advanced

Using a query whitelist puts the decision in the hands of your engineers
whether a particular query should be accepted or not. Sometimes this isn't
enough - it could be that your engineers need guidance to help them avoid
common pit-falls (e.g. forgetting to put limits on collections they query), or
it could be that you wish arbitrary third parties to be able to send queries to
your API without the queries being pre-approved and without the risk of
bringing your servers to their knees.

**You are highly encouraged to purchase the [Pro Plugin [PRO]](/postgraphile/pricing/),
which implements these protections in a deeply integrated and PostGraphile
optimised way, and has the added benefit of helping sustain development and
maintenance on the project.** You can read the [@graphile/pro README on
npm](https://www.npmjs.com/package/@graphile/pro).

The following details how the Pro Plugin addresses these issues, including
hints on how you might go about solving the issues for yourself. Many of these
techniques can be implemented outside of PostGraphile, for example in an
express middleware or a nginx reverse proxy between PostGraphile and the
client.

#### Sending queries to read replicas

Probably the most important thing regarding scalability is making sure that
your master database doesn't bow under the pressure of all the clients
talking to it. It's wise to perform load testing to figure out at what point
this will occur, and have a plan for it. One way to reduce this pressure is
to offload read operations to read replicas (clones of your primary database)

- this reduces the load on your primary database significantly, and reduces
  the need for complex caching layers. In GraphQL it's easy to tell if a
  request will perform any writes or not: if it's a `query` then it's
  read-only, if it's a `mutation` then it may perform writes.

Using `--read-only-connection <string>` [PRO] you may give PostGraphile a
separate connection string to use for queries, to compliment the connection
string passed via `--connection` which will now be used only for mutations.

(If you're using middleware, then you should use the `readOnlyConnection`
option instead.)

> NOTE: We don't currently support the multi-host syntax for this connection
> string, but you can use a PostgreSQL proxy such a PgPool or PgBouncer between
> PostGraphile and your database to enable connecting to multiple read
> replicas.

#### Pagination caps

It's unlikely that you want users to request `allUsers` and receive back
literally all of the users in the database. More likely you want users to use
cursor-based pagination over this connection with `first` / `after`. The Pro
Plugin introduces the `--default-pagination-cap [int]` [PRO] option (library
option: `defaultPaginationCap`) which enables you to enforce a pagination cap
on all connections. Whatever number you pass will be used as the pagination
cap (allowing requests smaller or equal to this cap to go through, and
blocking those above it), but you can override it on a table-by-table basis
using [smart comments](/postgraphile/smart-comments/) - in this case the
`@paginationCap`[PRO] smart comment.

```sql
comment on table users is
  E'@paginationCap 20\nSomeone who can log in.';
```

#### Limiting GraphQL query depth

Most GraphQL queries tend to be only a few levels deep, queries like the deep
one at the top of this article are generally not required. You may use
`--graphql-depth-limit [int]` [PRO] to limit the depth of any GraphQL queries
that hit PostGraphile - any deeper than this will be discarded during query
validation.

#### [EXPERIMENTAL] GraphQL cost limit

The most powerful way of preventing DOS is to limit the cost of GraphQL
queries that may be executed against your GraphQL server. The Pro Plugin
contains a early implementation of this technique with heuristically
estimated costs. You may enable a cost limit with `--graphql-cost-limit [int]` [PRO] and the calculated cost of any GraphQL queries will be made
available on `meta` field in the GraphQL payload.

If your GraphQL query is seen to be too expensive, here's some techniques to
bring the calculated cost down:

- If you've not specified a limit (`first`/`last`) on a connection, we assume
  it will return 1000 results. You should always specify a limit.
- Cost is based on number of expected results (without looking at the
  database!) so lowering your limits on connections will also lower the costs.
- Connections multiply the cost of their children by the number of results
  they're expected to return, so lower the limits on parent connections.
- Nested fields multiply costs; so pulling a connection inside a connection
  inside a connection is going to be expensive - to address this, try placing
  lower `first`/`last` values on the connections or avoiding fetching nested
  data until you need to display it (split into multiple requests / only
  request the data you need for the current view).
- Subscriptions are automatically seen as 10x as expensive as queries - try
  and minimise the amount of data your subscription requests.
- Procedure connections are treated as more expensive than table connections.
- `totalCount` on a table has a high cost
- `totalCount` on a procedure has a higher cost
- Using `pageInfo` adds significant cost to connections
- Computed columns are seen as fairly expensive - in future we may factor in
  PostgreSQL's `COST` parameter when figuring this out.

Keep in mind cost analysis is hard and the real cost of a query varies wildly
based on what your database has been dealing with recently, what indexes are
available, and many more factors. Our cost estimation is based on analysis of
a large test suite, but feel free to reach out with any bad costs/queries so
we can improve this feature.
