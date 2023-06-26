# Future work --- Software Heritage
[future]: #future-work-swh

## Motivation

Software Heritage is a great way to get source code that is no longer found at its original location online.
It would be really great to use it as a substituter to fetch source code that fails to be fetched the default way.

## Detailed Designed

- **Depends on**: Nix-agnostic content-addressing "stores"

[`GET /api/1/raw/(swhid)/`](https://docs.softwareheritage.org/devel/swh-web/uri-scheme-api.html#get--api-1-raw-(swhid)-)
is a relatively new Software Heritage API endpoint to get a git object for the given SWHID.
We can used this to write a Software Heritage store, which will accomplish the task layed out in the previous motivation section.

# Future work --- IPFS
[future]: #future-work-ipfs

## Motivation

### Binary Distribution

IPFS is a potential solution leveraging content addressing for a P2P CDN.
Out-of-the-box it matches Nix's data model extremely well.

### Source distribution and archival

IPFS also supports git hashing, and so we also provide a good way for people and institutions to "pin" the sources they need, especially if those sources include private ones Software Heritage won't have.
Finally, per [this IPFS plugin](https://blog.obsidian.systems/software-heritage-bridge/), we can have a 3 way integration between IPFS, Software Heritage, and Nix.
Data can be directly downloaded from Software Heritage via HTTPS, or indirectly via IPFS.

This would hopefully supercede the direct Software Heritage store outlined above.
This is because Software Heritage is primarily an archive, and we should be careful not to waste their limitted resources with copious egress bandwidth.
IPFS can act as a CDN to not put as much load on Software Heritage's servers.

### Not just IPFS

Many of the IPFS-specific logic could in fact live in a plugin if this is desired.
However, we still need to adjust core abstractions of Nix store layer (as described below) to interface with IPFS in the best possible way.
Those same adjustments would allow Nix to work better with *any* content-addressing system, so alternatives networks/projects to IPFS can also be just as easily experimented with.

As always with my work, the mantra (from Scheme) to follow is

> *x* should be designed not by piling feature on top of feature, but by removing the weaknesses and restrictions that make additional features appear necessary.

A ton of misc features have been added to Nix since 2.0, and we are very careful to not increase total ad-hoc complexity more than necessary.

## Detailed Design

Each item can be done separately provided its dependent items are also done.
These are the items we do *not* wish to commit to at this time, but leave for later.
(See [Detailed Design](#detailed-design) for other steps left for later.)

### Augmented `narinfo`

- **Purpose**: Binary distribution

*This is taken from [RFC PR 122](https://github.com/lucasew/rfcs/blob/binary-cache-ipfs/rfcs/0122-binary-cache-ipfs.md), which was abandoned by its author.*

The purpose of this is a "hybrid" store where the narinfo metadata is still served via HTTPS, but the data itself is served via IPFS.

Today, a narinfo looks like this:

```
StorePath: /nix/store/gdh8165b7rg4y53v64chjys7mbbw89f9-hello-2.10
URL: nar/0i6ardx43rdg24ab1nc3mq7f5ykyiamymh1v37gxdv5xh5cm0cmb.nar.xz
Compression: xz
FileHash: sha256:0i6ardx43rdg24ab1nc3mq7f5ykyiamymh1v37gxdv5xh5cm0cmb
FileSize: 40360
NarHash: sha256:1ddv0iqq47j0awyw7a8dmm8bz71c6ifrliq53kmmsfzjxf3rwvb8
NarSize: 197528
References: 7gx4kiv5m0i7d7qkixq2cwzbr10lvxwc-glibc-2.27 gdh8165b7rg4y53v64chjys7mbbw89f9-hello-2.10
Deriver: 5sj6fdfym58sdaf3r5p87v4l8sj2zlvn-hello-2.10.drv
Sig: cache.nixos.org-1:K0thQEG60rzAK8ZS9f1whb7eRlIshlMDJAm7xvX1oF284H+PTqlicv/wGW6BIj+wWWONHvUZ2MYc+KDArekjDA==
```

This RFC proposes new key-value pairs that in this example would be:

```
IpfsCid: Qmf8NfV2hnq44RoQw9vxmSpGYTwAovA8FUCxeCJCqmXeNN
IpfsEncoding: {"method":"wrapped-nar","chunking":{"leaf-format":"raw","strategy":"fixed-size"},"layout":"balanced","max-width":174}
```

Just as today, the `NarHash` and `NarSize` remain the *normative* way to lock down the store object the `narinfo` file describes.
Conversely, The `URL`, `FileHash` and `FileSize` by contrast are *informational*, describing not what the store object *is*, but *how to get it*.

The `IpfsCid` and `IpfsEncoding` are likewise informational, describing how to get the store object:

- `IpfsCid`: Native content address for IPFS.

- `IpfsEncoding`: Enough info to deterministically rebuild the IPFS representation from a non-IPFS copy of the store object.

   For now, `IpfsEncoding` will only support `unixfs-nar`, which works as follows:

   The NAR is itself wrapped in IPFS's [UnixFS](https://github.com/ipfs/specs/blob/main/UNIXFS.md).
   This other format can be extracted from the CID (which is conceptually a pair of encoding metadata and a hash).
   For now, only IPFS's "unixfs" is supported.
   `chunking`, `layout`, and `max-width` are tuning parameters for unixfs [described in the UnixFS spec](https://github.com/ipfs/specs/blob/main/UNIXFS.md#importing).

   "UNIXFS" is not used directly because it doesn't support the "executable bit" Nix does on files.
   NAR archive are not used directly because IPFS doesn't support arbitrarily large objects.

### IPFS Narinfo and "stateful" IPFS Store

- **Purpose**: Binary distribution
- **Depends on**: Augmented `narinfo`

Instead of a "hybrid" store, where the narinfo index is served with HTTP but the data itself is served with IPFS, we can do an all-IPFS store with the data itself and mutable index stored in IPFS.
The Narinfo instead of being encoded the legacy line-oriented text format can be IPFS's native DAG-CBOR IPLD codec, which is like JSON + content address links (but stored as CBOR).
This allows Narinfos to reference each other and be nicely structured so the index is legible from Nix-agnostic IPFS tools and recursive pinning comes for free.

Read-only is easier, since IPFS data is immutable but "writable" stores are supported by simple printing back a new CID for the new store root after some modifications, or modifying a mutable IPNS reference.
IPNS is historically slow, but the update is automatic.
Printing out a new CID for the index root allows the store administrator to update an out-of-bound mutable reference, but this cannot be automated because Nix doesn't know what the out-of-band method is.

### "stateless" IPFS store

- **Purpose**: Source distribution and archival
- **Depends on**: Nix-agnostic content-addressing "stores"

Use the above functionality to create a "stateless" IPFS store.
Opaque store path lookups always fail, but when the key is the new content address type, we can translate the key itself into a CID that we can look up.

Unlike the previous two flavours of IPFS store, this one is stateless in that there is no need for an index at all.
Only content-addressed data is looked up, and it doesn't need any nar-info metadata before the data is all there.

We need the previous step for querying without fetching any data.
In that case since there is no narinfo index we're looking up, we don't get any additional metadata back.
But the content address key a successful query used is enough to create a bare-bones `ValidPathInfo` with a `CA` field, which with the enough step is valid.

(A bare-bones `ValidPathInfo` might sound sub-par, but for plain old content-addressed data it is fine.
Most of the other metadata in `ValidPathInfo` is really just for input-addressed derivation outputs, and is thus obviated by CA derivation trust maps which contain the same data but more naturally.)

### Wrapped git objects with references

- **Purpose**: Binary distribution
- **Depends on**: Git file hashing

Merkelized formats like git file hashing are better than NAR because that allow for very natural deduplication and minimal transfers.
This is the same benefit we get today with Nix within a closure of multiple store objects, now also *within a single store object*.
But git has no notion of Nix-style references, so plain git hashing is only suitable for leaf store objects without references (like source code).

However, we can use IPLD to wrap git-hashed data with a reference set, and "has self reference" bit.
This easily creates a new content addressing scheme which handles all "shapes" of store objects.
This gives is a nice way to thus share arbitrary nix store data (provided it is content-addressed) over IPFS.

Like with "IPFS Narinfo", this format is also very easy to understand with nix-agnostic native IPFS tools.
This is because, once again, the reference graph is made native to IPFS not done indirectly with store path strings which must be looked up.

An interesting corollary to note:
Content addressing today is "shallow", in that references are arbitrary store paths.
With this form on content addressing, references are instead CIDs (native IPFS references) to other obligatorily content-addressed data.
This means the content addressing is "deep", such that any such content-addressed store object always has a content-addressed closure.
At the cost of interop with existing derivation outputs, this make such data easier to manage because there are fewer trust issues and degrees of freedom in general for something to go wrong.

### IPLD Derivations

- **Purpose**: Build plan distribution
- **Depends on**: Wrapped git objects with references,
                  IPFS as substitutor

Natively represent derivations in IPFS, again with the same benefits of leverage the native graph representations.

This is a culmination of all the features so far.
The derivations must be CA derivations (floating or fixed).
They must also produce wrapped git objects with references, though they can also depend on regular unwrapped git file hashed store objects.

The derivations and their outputs are thus all fully IPFS native, leveraging the IPFS graph and trust vs plain old data separation for the high standard of interoperability.
