# Error-code stability and upgrade policy

This document states what integrators (frontends, indexers, scripts, other
contracts) can rely on staying the same between releases of
`donation-vault` and `ngo-registry`, what may change, and how a change that
breaks those guarantees is handled and announced.

It exists because the guarantees were not always kept: error numbers have
been renumbered in the past, and `Stream` gained the `cancelled` and
`status` fields in place, so `Stream` entries written by older code no
longer decode under the current code. The rules below are what every change
from now on must follow.

## 1. Error codes are append-only

Clients see an error as its number (`Error(18)`), not its name, so the
number is the contract.

- **Never renumber.** Once a variant ships with a number, that number keeps
  that meaning forever.
- **Never reuse.** A number that was ever assigned is never given to a
  different error, even if the original variant is no longer returned.
- **Never remove a variant.** If an error can no longer happen, keep the
  variant and say in its doc comment that it is no longer returned. A
  removed variant would let a later change quietly reuse its number.
- **Append new errors at the end** with the next unused number, and add a
  row to the error table in the [README](../README.md#error-codes) in the
  same pull request.
- **The meaning of a code may widen, not change.** An existing code may be
  returned by additional entry points if the condition is the same (for
  example, `NgoNotVerified` is now also returned by a rate increase in
  `modify_rate`). Returning an existing code for a *different* condition is
  not allowed: add a new code instead.

A reviewer should reject any diff to a `#[contracterror]` enum that changes
an existing `= N` value or deletes a line from it.

Making an entry point return an error it did not return before (a new
validation) is allowed in a normal release, but it must be listed in the
changelog, because a client that assumed the call would succeed now sees a
failure.

## 2. Storage-layout changes across an upgrade

`upgrade` replaces the Wasm in place and keeps every storage entry. The new
code must therefore be able to read every entry the old code wrote.

### Why layout matters

A `#[contracttype]` struct is stored as a map of all its fields, and is
decoded strictly: an entry with a missing or extra field fails to decode,
and the call traps. A `#[contracttype]` enum is stored by variant name. So,
for any type that is written to storage (the vault's `Stream` and
`StreamStatus`, the registry's `Ngo`, and every `DataKey` variant in both
contracts):

| Change | Safe in place? |
| --- | --- |
| Add a new `DataKey` variant | Yes |
| Add a new enum variant to a stored enum (e.g. `StreamStatus`) | Yes, provided old code is never redeployed over data that uses it |
| Add, remove, rename, or retype a field of a stored struct | **No** |
| Rename or remove a `DataKey` variant, or change its payload type | **No** — existing entries become unreachable |
| Change the storage type (instance / persistent) of a key | **No** — the old entries stay where they are |

### Rule

A change marked **No** above must not be released as a plain upgrade. It
must use one of these patterns, chosen in the pull request and written in
its description:

1. **Versioned key, lazy migration (preferred).** Leave the old type and
   key untouched and add a new type under a new key (for example a
   `StreamV2` stored at `DataKey::StreamV2(id)`). Every read tries the new
   key first and, if it is absent, reads the old key, converts the value,
   and writes it back under the new key. Old entries are migrated the next
   time they are touched, so no single transaction has to rewrite every
   stream.
2. **Explicit migration entry point.** Add an admin-gated, idempotent
   `migrate` entry point that converts a bounded batch of entries per call
   (bounded so a call stays inside the per-transaction budget), and a
   stored schema version (for example `DataKey::SchemaVersion`) that it
   advances when it finishes. Fund-moving entry points must refuse to run
   (or must use pattern 1) until the version is current.

In both cases the pull request must include a test that writes an entry in
the **old** layout, runs the new code against it, and checks that it reads
back correctly.

The contracts have no migration entry point or schema version today. The
first change that needs one adds it, following pattern 2.

### Upgrade procedure

For any upgrade of a deployed contract:

1. `pause` the vault (fund-moving entry points stop; `cancel_stream` still
   works, so donors can always exit).
2. Call `upgrade` with the new Wasm hash. It emits an `upgrade` event.
3. If the release includes a migration (pattern 2), run `migrate` until it
   reports completion.
4. Spot-check with read-only calls (`get_config`, `get_stream` on old and
   new streams).
5. `unpause`.

Record the new Wasm hash and the date in `deployments.json` and in the
release notes.

## 3. What may change between releases

| Surface | Stability |
| --- | --- |
| Error numbers | Append-only (section 1). Never breaking. |
| Stored data layout | Never broken in place (section 2). |
| Entry-point names, argument types, and return types | Breaking if changed; follows section 4. |
| Event topics and data shapes (see [EVENTS.md](EVENTS.md)) | Breaking if changed; follows section 4. |
| Struct shapes returned by getters (e.g. `Config`) | Breaking if changed; follows section 4. |
| New entry points, new events, new error codes | Not breaking. Listed under *Added*. |
| New validations on existing entry points | Not breaking, but listed in the changelog (section 1). |

## 4. Announcing breaking changes

A change is breaking if a client written for the previous release would
decode something wrongly, call something that no longer exists, or need code
changes to keep working.

1. **Mark it in the pull request.** The title or description says
   `BREAKING` and explains who is affected and what they must change.
2. **Changelog.** Add an entry under *Changed* in
   [CHANGELOG.md](../CHANGELOG.md), prefixed with **BREAKING**, naming the
   contract, the old and new shape, and what integrators must do.
3. **Release notes.** The release that ships it repeats the BREAKING
   entries at the top, with the new Wasm hashes.
4. **Notice before mainnet.** A breaking change is deployed to testnet
   first and announced at least 14 days before the mainnet `upgrade`, so
   indexers and frontends can ship compatible decoders. A fix for an
   actively exploitable vulnerability may skip the notice period; it is
   then announced as soon as the upgrade lands, following
   [SECURITY.md](../SECURITY.md).
5. **On-chain signal.** Integrators can watch for the `upgrade` event
   emitted by both contracts to know exactly when new code took effect.
