# RealtyClan fork: ATTRIBUTE_MALLOC patch on `bit_TV_to_utf8`

This fork carries one source patch on top of upstream libredwg
(at the time of the diverge, `0.13.4.8168_dirty` on
`origin/master`):

| Commit     | Subject                                              |
|------------|------------------------------------------------------|
| `43f9d791` | bits: drop ATTRIBUTE_MALLOC from bit_TV_to_utf8      |
| `066181dc` | test/bits: add aliasing-return regression for bit_TV_to_utf8 |

Both live on `pre-dev`. The PR is upstreamed at
[LibreDWG PR #1250](https://github.com/LibreDWG/libredwg/pull/1250)
(via the `fix/bit-tv-to-utf8-aliasing-malloc-attr` branch); this
fork carries the patch until that lands.

## Why the patch

`bit_TV_to_utf8` (in `src/bits.c`) is documented to "return … the
unchanged src string, or a copy". In practice it returns the input
pointer on at least four paths — empty strings, ASCII-only strings
of length ≤ N, etc. — without copying.

The function was annotated with `ATTRIBUTE_MALLOC` (gcc's
`__attribute__((malloc))`). That annotation **promises** to the
compiler that the return value is a fresh allocation, distinct from
every other live pointer. With `-O2`, gcc/clang exploit that
promise to enable aliasing optimisations: callers can treat the
returned pointer as non-aliasing with anything else, and may pass
it onward to `free()` paths under the assumption that no one else
holds a live reference.

When `bit_TV_to_utf8` actually returns the *input* pointer, both
the original owner and the caller hold the "same" pointer that the
compiler thinks is two distinct allocations. The downstream
`free()` / cleanup paths then perform double-frees / use-after-
frees, which the heap allocator detects → SIGABRT.

## Why the bug shows up at all

The corruption is only reliably reachable on drawings whose
`OBJECTS` section uses the unstable-class code paths for `MATERIAL`,
`TABLESTYLE`, or `MLEADERSTYLE` — common on R2018 drawings from
AutoCAD 2018+. Older drawings, or R2018 drawings that don't
exercise those classes, often complete cleanly even on stock
libredwg. That's why this bug isn't seen by every user.

A real-world reproducer from RealtyClan's terraflow pipeline:

```
Reading DWG file Panchvati-GeoRef.dwg
Warning: Unstable Class object 502 MATERIAL (0x481) 45/0
Warning: Unstable Class object 504 TABLESTYLE (0xfff) 96/0
Warning: TODO TABLESTYLE r2010+ missing fields
… (cleanup-time SIGABRT, rc=134)
```

The DXF that hits disk before the SIGABRT is truncated — typically
~29 KB for a drawing that should produce 40+ MB. No `ENDSEC` for
the last in-progress section, no `EOF`. ezdxf rejects it.

## What the fix does

Drop `ATTRIBUTE_MALLOC` from the declaration in `src/bits.h`. That's
it. The function still has the same behaviour; the compiler just
no longer applies aliasing optimisations that assume the return is
a fresh allocation.

Regression test (`066181dc`) verifies that the input-pointer-return
path continues to work after the change.

## How to build

```bash
sh autogen.sh
./configure --disable-bindings --disable-shared \
  CFLAGS='-O2 -g -fno-omit-frame-pointer'
make -j8
# binary lands at programs/dwg2dxf (~13 MB)
```

The `--disable-bindings --disable-shared` flags trim build time
significantly without affecting the CLI binaries we use. `-O2`
plus the frame-pointer flag matches what we ship in production
for sane backtraces under crash.

## Consumers in the RealtyClan monorepo

- `services/terraflow` — the FastAPI ingest service. Its
  `DwgConverter._find_libredwg()` walks up from the source file
  looking for `services/libredwg/programs/dwg2dxf` and prefers it
  over `$PATH` or `~/.local/bin`. See
  `services/terraflow/docs/LIBREDWG_PATCH.md` for the full bug
  background + diagnostic commands.

## Keeping in sync with upstream

When PR #1250 (or any equivalent) lands upstream:

1. Verify upstream master has the equivalent `ATTRIBUTE_MALLOC`
   removal (the change may be applied differently — e.g., as a
   typedef change or a comment-only docs fix).
2. Rebase `pre-dev` onto the new upstream master.
3. Drop `43f9d791` and `066181dc` if upstream covers both the
   fix and an equivalent regression test.
4. Run `make check` to confirm parity.
5. Tag the merge so terraflow's `_find_libredwg` continues to
   pick up the right binary after a rebuild.

Until then this fork stays on `pre-dev`.
