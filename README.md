<!--
SPDX-FileCopyrightText: 2021 Anders Rune Jensen

SPDX-License-Identifier: CC-BY-4.0
-->

:warning: **This repo was moved to https://github.com/ssbc/ssb-subset-replication-spec.** This archival will remain in this GitHub org `ssb-ngi-pointer` to demonstrate the outcome of the work done by the SSB NGI Pointer team during 2020 and 2021. The SSB NGI Pointer team is no longer active because we completed our grant project.

# Subset replication for SSB

Status: Design phase

This module aims to provide methods for synchronizing a subset of one
or more SSB feeds. The main motivation is to enable faster initial
sync by only replicating a certain subset of messages that can then be
supplimented with a frontier sync such as [EBT].

Feeds in classical SSB can said to contain multiple subset of
messages, this could be messages of a certain type (eg. contact or
chess moves) or could be private messages pertaining to a [private
group].

In classical SSB, feeds up to a hops count (normally 2 or 3) are
replicated in full. There also exists a method to get a particular
message out of order, the security of this hinges on the fact that you
get the ooo message by the hash and so is a special case of engtangled
messages.

Here we are interested in adding APIs that, given [ssb-meta-feed],
allows two peers to exchange feeds and subset of feeds to allow for
partial replication.

We are mainly interested in two classes of messages: a subset of
messages of a classical SSB feed and a tangle of messages given a
common root message.

## getSubset(query, options): source

This method can be seen as a more general version of
createHistoryStream where the id, or what you are interesting in,
first needs to be defined. Query specifies what data you are
interested in.

The query parameter is an instance of the query language defined
in `options.querylang`.

To support pagination, `startFrom`, `pageSize` and `descending` can be
specified in the options parameter.

It is up to the receiving end of the query to determine if the query
results in too much load. This could be if someone tries to get data
for a large number of feeds without proper pagination, or if the query
includes arguments in a form that strains the query planner.

To get the latest 10 public post messages of a particular feed the
following `query` and `options` arguments can be used:

```json
{
  "author": "@6CAxOI3f+LUOVrbAl0IemqiS7ATpQvr9Mdw9LC4+Uv0=.ed25519",
  "type": "post",
  "private": false
}
```

```
{
  "querylang": "ssb-ql-0",
  "descending": true,
  "pageSize": 10
}
```

Result:

```
[msg1, msg2, ..., msg10]
```

### Query language

#### ssb-ql-0

We start by supporting a proof-of-concept query language called
"ssb-ql-0" which is a precursor for the more powerful "ssb-ql-1"
(see below).

Queries in ssb-ql-0 are merely JSON objects with three *required*
fields:

- `author`: a valid feed ID as a string
- `type`: a message type, can be any non-empty string, or null,
  which signals that messages of any type are accepted
- `private`: a boolean 

If `private: true` then `type` **MUST** be `null`. This is to prevent
leaking metadata about private messages to the public.

#### ssb-ql-1

*Status: under design. We recommend using ssb-ql-0 in the meantime.
ssb-ql-1 has challenges such as determining when two queries are
equivalent, which may be an NP complete problem. Read more on this
SSB thread:* `%FjK8SdR2Q8n3XBwFKiI8ebdSr/Tp7bMzhr2R6YMq+b0=.sha256`

Here we define a mini query language that can be used to specify a
subset of data. The goal of this format is to be easy to parse and
also be restrictive in the number of operations supported to make it
easier to map to indexes and to limit the attack surface.

The query uses an object notation for operators that looks very
similar to JSON. The objects contains two keys: `op` the name of the
operation and a second parameter for the arguments given the
operation.

Base operators:

op     | args name   | args type
------ | ---------   | ---------
and    | args        | [op, ...]
or     | args        | [op, ...]
type   | string      | string
author | feed        | string

Note that `ssb-ql-1` is only for public messages and is incapable of
querying private messages, unlike `ssb-ql-0`.

The spec is open for implementations to add new operators relatively
easy. This allows experimentation and implementations should simply
error if presented with an unsupported operator. A new potential
operator could be `isPrivate` or `isBox2` but extreme care must be
taken not to leak information using these queries around private
information.

## getTangle (TBD)

Tangles in SSB behave differently than other types of messages in
that they are linked together across multiple feeds using links. This
allows for a different kind of synchonization than `getSubset`. There
are two common examples of where this could be useful: for getting all
the messages of an old message threads that is outside the frontier of
messages and to synchronize a private group (as all messages in those
are part of a single tangle). While private groups are by definition
private to external observers, the fact that nodes connecting in SSB
are authenticated by their ID and that rooms allow direct connections
between peer allows synchronization of a private group directly with
another group member who also knows which messages belong to the group.

As private groups can grow quite large it becomes important that there
is a way to synchronize only subsets but also changes or the latest
part of a tangle. The membership part of a private group has a tangle
root and thus can be replicated on its own.

Consider using [set-reconciliation] for this.

# Prior work

Earlier threads on SSB discussing partial replication:

- %L9m5nHRqpXM4Zkha1ENTk5wNOXQMduve8Hc9+F0RLZI=.sha256
- %VNCOf3pfP7hjL3lcFvBSSOzRPtR0WHDtvlXvkBcha3I=.sha256
- %gaYxXAEoSKf3dnw2OlI2EuoduFmkiU4kU+CncbBImPk=.sha256
- %jM6Dw0BEg9IPVkDH5VaoiVSbt3gCnUfSoO1BhX2Ntpw=.sha256
- %SEXkvEQCJd2jBrUXDQcFPVix/5eUffoKBURWyy5yYrU=.sha256

[JITDB]: https://github.com/arj03/jitdb
[ssb-meta-feed]: https://github.com/arj03/ssb-meta-feed
[set-reconciliation]: https://github.com/AljoschaMeyer/set-reconciliation
[EBT]: https://github.com/ssbc/epidemic-broadcast-trees/
[private group]: https://github.com/ssbc/private-group-spec
