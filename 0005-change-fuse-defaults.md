- Start Date: 2024-06-24
- RFC PR: [electron/rfcs#0000](https://github.com/electron/rfcs/pull/6)
- Status: **Proposed**

# Change Electron Fuse Security Defaults

## Summary

We'd like to change the current default values for certain fuses exposed via [Electron Fuses](https://www.electronjs.org/docs/latest/tutorial/fuses).

## Motivation

The motivation for all of these swaps (outlined in details below) is entirely security focused. Ensuring that fuses are in the "hardened" state by default will protect apps, limit the amount of work that developers need to do to make secure apps and overall improve ths security posture of the Electron ecosystem.

## Guide-level explanation

The fuses that should have their defaults changed are listed below, along with the theoretical impact of the change, and proposed timelines.

### [`runAsNode`](https://www.electronjs.org/docs/latest/tutorial/fuses#runasnode)

Currently defaults to enabled, we should switch this to default to disabled.

This flag makes it so that `ELECTRON_RUN_AS_NODE=1` does not cause your Electron app to become a floating node executable. The docs propose safer alternatives, primarily `utilityProcess.fork`.

We should deprecate (log a deprecation warning when `ELECTRON_RUN_AS_NODE` is used) and then swap the default. Proposed timeline is:
* Deprecate in Electron 33
* Swap default in Electron 35

### [`cookieEncryption`](https://www.electronjs.org/docs/latest/tutorial/fuses#cookieencryption)

Current defaults to disabled, we should switch this to default to enabled.

This flag enables encryption of cookies using user/device specific encryption keys. This is industry standard for cookie storage and only has one drawback, namely that it is a one way transition. Once you have run a version of Electron that has cookie encryption enabled, downgrading is a destructive operation.

We should swap the default without deprecation (it is unclear how to log a deprecation warning in a targeted way) but call it out at the top of the release notes + blog post. Proposed timeline is:
* Swap default in Electron 33

### [`nodeOptions`](https://www.electronjs.org/docs/latest/tutorial/fuses#nodeoptions)

Current defaults to enabled, we should switch this to default to disabled.

`NODE_OPTIONS` and `NODE_EXTRA_CA_CERTS` are basically unusable by default in Electron apps anyway due to how apps are traditionally launched. (They don't get environment variables by default on macOS for instance).

We should deprecate (log a deprecation warning when one of the env vars is used) and then swap the default. Proposed timeline is:
* Deprecate in Electron 33
* Swap default in Electron 35

### [`nodeCliInspect`](https://www.electronjs.org/docs/latest/tutorial/fuses#nodecliinspect)

Current defaults to enabled, we should switch this to default to disabled **when packaged**

This requires adding a new fuse mode, technical implementation withstanding we will need a way to indicate that a fuse is either:
* Enabled in dev and disabled when packaged
* Disabled in dev and enabled when packaged

Given the current fuse schema this should be easily expandable as each fuse has a whole byte of space to store information (currently just using 0/1 but have a full byte).

We should swap the default without deprecation (it is unclear how to log a deprecation warning in a targeted way) but call it out at the top of the release notes + blog post. Proposed timeline is:
* Swap default in Electron 33

### Other fuses of interest

Other security relevant fuses such as `grantFileProtocolExtraPrivileges`, `embeddedAsarIntegrityValidation` and `onlyLoadAppFromAsar` aren't included here for their own reasons.

`grantFileProtocolExtraPrivileges` is a new fuse and has not been tested enough or have a good enough disabled story for your average Electron app to change the default at this time. This can be revisited at some point in the future.

`embeddedAsarIntegrityValidation` and `onlyLoadAppFromAsar` could be enabled by default easily in packaging tools. But enabling by default in Electron itself is quite violent and would require substantial discussion, more-so than the fuses mentioned in this existing proposal and as such will be left to a later date. Ideally asar integrity is enabled by default on macOS and Windows at some point.

## Reference-level explanation

Technical implementation here is super easy, defaults are swapped in [`fuses.json5`](https://github.com/electron/electron/blob/main/build/fuses/fuses.json5)

## Drawbacks

These are breaking changes, despite deprecation warnings and release notes this may negative impact apps that upgrade and don't realize. The breakages should be fairly obvious to apps relying on these features though and therefore shouldn't slip through even basic testing.

## Rationale and alternatives

Only alternative is that we're happy with the status quo for all / a subset of these fuses.

## Prior art

N/A

## Unresolved questions

N/A

## Future possibilities

N/A
