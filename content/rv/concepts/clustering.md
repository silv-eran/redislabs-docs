---
Title: Clustering Redis Databases
description:
weight: $weight
alwaysopen: false
categories: ["RC Pro"]
---
While it is a design decision that Redis is a (mostly) single-threaded
process and this does keep it extremely performant yet simple, there are
times when clustering is advised. Redis Cloud Pro employs our
Redis Enterprise technology to scale Redis databases on your behalf.

A Redis Cloud Pro cluster is a set of managed Redis processes and
cloud instances, with each process managing a subset of the database's
keyspace. This approach overcomes scaling challenges via horizontal
scaling using multiple cores and multiple instance's resources.

In a Redis Cloud Pro cluster, the keyspace is partitioned into hash
slots. At any given time a slot resides on and is managed by a single
Redis server. An instance that belongs to a cluster can manage multiple
slots. This division of the key space, a.k.a. sharding, is achieved by
hashing the keys' names, or parts of these (key hash tags), in order to
obtain the slot in which a key should reside.

Even while employing multiple Redis processes, the use of a Redis
Cloud Pro cluster is nearly transparent to the application that
uses it. The cluster is accessible via a single endpoint that
automatically routes all operations to the relevant shards, without the
complexity of a cluster-aware Redis client. This allows applications to
benefit from using the cluster without performing any code changes, even
if they were not designed to use it beforehand.

When creating or editing a Redis database on Redis Cloud Pro, the
system will automatically calculate the number of shards needed based on
the database memory limit and required throughput.

## Multi-Key Operations

Operations on multiple keys in a sharded Redis Cloud Pro cluster
are supported with the following limitations:

1. **Multi-key commands:** Redis offers several commands that accept
    multiple keys as arguments. In a sharded setup, multi-key commands
    can only be used when all affected keys reside in the same slot.
    These commands are: BITOP, BLPOP, BRPOP, BRPOPLPUSH, MSETNX,
    RPOPLPUSH, SDIFF, SDIFFSTORE, SINTER, SINTERSTORE, SMOVE, SORT,
    SUNION, ZINTERSTORE, ZUNIONSTORE.
1. **Geo commands:** In GEORADIUS/GEOREADIUSBYMEMBER commands, the
    STORE and STOREDIST options can only be used when all affected keys
    reside in the same slot.
1. **Transactions:** All operations within a WATCH/MULTI/EXEC block
    should be performed on keys that are in the same slot.
1. **Lua scripts:** All keys that are used by the script must reside in
    the same slot and need to be provided as arguments to the
    EVAL/EVALSHA commands (as per the Redis specification).
1. **Renaming keys:** The use of the RENAME/RENAMENX commands is
    allowed only when both the key's original name and its new name are
    mapped to the same hash slot.

## Changing the Sharding Policy

The clustering configuration of a Redis Cloud Pro instance can be
changed. However, sharding policy changes will trigger the deletion
(i.e. FLUSHDB) of the data before they can be applied. These changes
are:

1. Changing the hashing policy from standard to custom or vice versa.
1. Changing the order of custom hashing policy rules.
1. Adding rules before existing ones in the custom hashing policy.
1. Deleting rules from the custom hashing policy.
1. Disabling clustering for the database.

### Standard Hashing Policy

When using the standard hashing policy, a Redis Cloud Pro cluster
behaves like the standard, open-source Redis cluster, and hashing is
performed as follows:

1. Keys with a hash tag: a key's hash tag is any substring between '{'
    and '}' in the key's name. That means that when a key's name
    includes the pattern '{...}', the hash tag is used as input for the
    hashing function. For example, the following key names have the same
    hash tag and would therefore be mapped to the same slot: foo{bar},
    {bar}baz & foo{bar}baz.
1. Keys without a hash tag: when a key doesn't contain the '{...}'
    pattern, the entire key's name is used for hashing.

You can use the '{...}' pattern to direct related keys to the same hash
slot, so that multi-key operations are supported on them. On the other
hand, not using a hash tag in the key's name results in a
(statistically) even distribution of keys across the keyspace's shards.
If your application does not perform multi-key operations, you don't
need to construct key names with hash tags.

### Custom Hashing Policy

A Redis Cloud Pro cluster can be configured to use a custom hashing
policy. A custom hashing policy is required when different keys need to
be kept together on the same shard to allow multi-key operations. Redis
Cloud Pro's custom hashing policy is provided via a set of Perl
Compatible Regular Expressions (PCRE) rules that describe the dataset's
key name patterns.

To configure a custom hashing policy, enter regular expression (RegEx)
rules that identify the substring in the key's name - hash tag - on
which hashing will be done. The hashing tag is denoted in the RegEx by
the use of the \`tag\` named subpattern. Different keys that have the
same hash tag will be stored and managed in the same slot.

Once you enable the custom hashing policy, the Redis Cloud Pro's
default RegEx rules that implement the standard hashing policy are:

|  RegEx Rule | Description |
|------------|-----------------|
|  .\*{(?\<tag\>.\*)}.\* | Hashing is done on the substring between the curly braces. |
|  (?\<tag\>.\*) | The entire key's name is used for hashing. |

You can modify existing rules, add new ones, delete rules or change
their order to suit your application's requirements.

### Custom Hashing Policy Notes and Limitations

1. You can define up to 32 RegEx rules, each up to 256 characters.
1. RegEx rules are evaluated by their order.
1. The first rule matched is used; strive to place common key name
    patterns at the beginning of the rule list.
1. Key names that do not match any of the RegEx rules will trigger an
    error.
1. The '.\*(?\<tag\>)' RegEx rule forces keys into a single slot as the
    hash key will always be empty. When used, this should be the last
    catch-all rule.
1. The following flag is enabled in our regular expression parser:
   - **PCRE_ANCHORED:** the pattern is constrained to match only at
        the start of the string which is being searched.
