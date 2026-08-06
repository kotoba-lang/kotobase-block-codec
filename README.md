# kotobase-block-codec

Deterministic compression for the bytes **inside** a kotobase block — above the
CID, below the AEAD.

Design: [ADR-2608060500](https://github.com/com-junkawasaki/root/blob/main/90-docs/adr/2608060500-kotobase-block-compression-frame.edn).

```clojure
(require '[kotobase.blockcodec.core :as bc])

(bc/frame   payload)   ; bytes -> bytes.  Unary. Total. Pure.
(bc/unframe framed)    ; bytes -> bytes.  The identity on pre-codec data.
```

## Why the seam is exactly here

A kotobase block's CID is the sha2-256 of the bytes handed to the block store,
and `ipld/get-verified-block` rehashes them coming back. That single fact
places this layer:

- **Below the CID is worthless.** The stored bytes are ciphertext —
  `kotobase-peer.core/put-tx-block!` encrypts the whole quad payload,
  `arrangement.core/index-root` encrypts each leaf value. Measured, ciphertext
  deflates to **ratio 1.003**: it *grows*. A provider that compressed its BLOBs
  would burn CPU to store more.
- **Above the CID changes the CID.** Which is the constraint, not a snag: two
  writers that disagree by one byte make two blocks, and dedup and
  `kotobase-engine-prolly`'s "CID-identical to a full rebuild" quietly stop
  holding.

So the frame sits on the plaintext payload, immediately before `encrypt-fn`,
and `frame` takes **no options** — not a level, not a threshold, not an
injected codec. Every knob is a constant of the format, because a knob is a way
for two writers to disagree.

This is deliberately the opposite of the injected-`encrypt-fn` discipline
(ADR-2607051000), and the difference is the CID: that ADR chose a *random*
nonce and said so, because an AEAD's output need not be reproducible. This
codec's output is inside the content address and must be byte-identical on
every runtime, forever.

**Corollary — a host codec may not be substituted as a fast path.** DEFLATE
output is not unique, so `java.util.zip` / node `zlib` / `CompressionStream`
would each yield a valid stream with different bytes, hence a different CID.
`org-ietf-deflate` being portable `.cljc` is what makes one answer possible on
JVM and Worker alike.

## The frame

```
0x01 ++ <RFC 1950 zlib stream>   compressed
0x00 ++ <payload>                identity, escaped
<payload>                        identity, bare
```

Identity is normally written **bare**, so a payload the codec cannot shrink is
byte-identical to what the same writer produced before this library existed:
turning compression on re-addresses only the blocks it actually shrinks, and
existing history keeps its structural sharing. The `0x00` escape covers a
payload that would otherwise begin with a tag byte.

`unframe` therefore reads an untagged payload as legacy data — sound wherever
payloads are DAG-CBOR node bytes, since a node is a map and its first byte is
0xA0–0xBB. `legacy-safe?` is the assertion to run before wiring this into a
seam whose payloads are something else.

Because `unframe` is the identity on legacy data, **the read path can be
deployed before the write path**, which is the correct rollout order.

## What it actually saves

`measure` reports a real sample rather than a constant, for the same reason
`kotobase-storage-kura/effective-multiplier` does — quoting 0.10 to a consumer
whose payloads are ciphertext would be a lie about what they are buying.

| payload | raw | framed | ratio |
|---|---|---|---|
| tx block, 5 quads | 212 B | 78 B | 0.368 |
| tx block, 20 quads | 862 B | 145 B | 0.168 |
| tx block, 60 quads | 2,622 B | 316 B | 0.121 |
| tx block, 200 quads | 8,982 B | 937 B | 0.104 |
| tx block, 800 quads | 36,472 B | 3,810 B | 0.104 |
| AEAD ciphertext, 60 KB | 60,001 B | 60,001 B | **1.000** |
| mixed 20 % quads / 80 % ciphertext | 419,720 B | 258,820 B | 0.617 |

Quad payloads compress ~10x because subject and predicate strings repeat.
Ciphertext does not compress at all, and the never-expand rule means it costs
nothing to try.

CPU is the reason the level is 1 and not 6: on a 61 KB payload, level 6 buys
~13 % more shrink (ratio 0.089 vs 0.101) for ~4x the time (1050 ms vs 268 ms),
and that time is spent on a Cloudflare Worker's write path. Warmed, level 1
runs about 270 KB/s under nbb.

## Tests

```sh
clojure -M:test
clojure -M:lint
nbb --classpath "src:test:../org-ietf-deflate/src" run-tests.cljs
```

`golden_test` pins the literal framed bytes and runs on both runtimes. It is
supposed to be annoying to change: a diff there says every `zlib-1` block from
now on has a different address than one framed before, and the way to say that
is a new tag byte, not an edit.

It has already earned its place twice. It caught a fixture that was silently
zero-filled on ClojureScript — `(map int "did")` is the code points on the JVM
and `[0 0 0]` on ClojureScript — which had made the whole ClojureScript suite
self-consistently test the wrong payload. And it caught a real defect in
`org-ietf-deflate`, whose Huffman construction tie-broke on hash-map iteration
order and so emitted *different, equally valid* streams on the two runtimes
(fixed upstream in `e755803`; that repo now has its own cross-runtime pin).

Both are invisible to a round-trip test, which is the whole argument for
pinning bytes.
