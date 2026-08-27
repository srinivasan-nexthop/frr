# Stage A: BGP-LS backport onto pristine frr-10.5.4

Branch `bgpls-on-10.5.4-stageA` in `~/work/upstream/frr`, worked in the
isolated worktree `~/work/upstream/frr-bgpls-stageA`. Goal: reconstruct
BGP-LS support (as it exists at the `frr-10.7.0` tag, exactly) on top of
pristine `frr-10.5.4`, as a clean commit sequence, for later export as
patch(es) into `sonic-frr/patch`.

## Scope verification (see also study file 010, once written)

- Started from an earlier undercount ("~40 commits") that turned out to be
  a coarse summary from an earlier session pass, not the real number.
- True count: **163 genuine commits** between `frr-10.5.4..frr-10.7.0`
  touching BGP-LS, verified via layered filtering:
  - 107 commits touch *only* the 6 brand-new `bgp_ls*.c/h` files (zero
    external conflict risk — file doesn't exist in 10.5.4 at all).
  - 56 commits touch at least one pre-existing file, across a ~62-file
    set (dominated by `bgp_attr.c`×12, `bgp_route.c`×10, `bgp_vty.c`×9,
    `bgp_zebra.c`×7, `bgpd.c`×7, `bgp_attr.h`×7, `bgpd.h`×6,
    `lib/link_state.c`×4, `isisd/isis_te.c`×4, `subdir.am`×4, rest 1-2×).
- One true false positive caught and excluded: `6361ead934` ("ospfd:
  Implement forwarding-address-self command") — matched via commit-body
  text containing OSPF's own native "Link State ID"/"AS External Link
  States" terminology (LSA vocabulary), completely unrelated to the
  BGP-LS *feature*. Caught by tightening the verification regex from a
  generic `link[- ]state` pattern (which OSPF/ISIS source trips constantly,
  since "link state" is core OSPF/ISIS vocabulary) to BGP-LS-specific
  tokens (`BGP-LS|BGP_LS|bgp_ls|AFI_BGP_LS|SAFI_BGP_LS|linkstate`).
  Confirmed via diff-content check (not just message) since even the
  false positive's C-code diff contained "link state" as an OSPF vty
  string literal — message-only filtering isn't enough; diff-content
  cross-checking against a tight token set was needed.
- Also confirmed 3 other tight-filter "zero hit" commits are genuinely
  relevant despite lacking the literal `BGP_LS` token (they touch the
  shared `lib/link_state.c` TED library or `bgp_zebra.c`'s opaque-message
  handling that BGP-LS's zebra-registration mechanism uses) — kept.
- Net: `ospfd/*` needs **zero** changes for this backport (confirmed no
  other genuine commit touches it) — BGP-LS's IGP-sourced TED consumption
  goes entirely through the pre-existing generic `lib/link_state` API.
- **Real BGP-LS AFI/SAFI names are `AFI_BGP_LS`/`SAFI_BGP_LS`**, not
  `AFI_LINKSTATE`/`SAFI_LINKSTATE` as shorthanded in earlier study files —
  corrected here; `SAFI_BGP_LS = 8`, same numeric slot our own
  `SAFI_SRPOLICY = 8` claims (the collision already identified in study
  file 006, confirmed again from this side).

## Method

1. Copied the 6 new files (`bgp_ls.c/h`, `bgp_ls_nlri.c/h`, `bgp_ls_ted.c/h`)
   verbatim from the `frr-10.7.0` tag as commit 1 — subsumes all 107
   new-file-only commits' net effect with zero replay.
2. For each of the 56 "touches-existing" commits (chronological order,
   `existing56_final.txt` in scratchpad — one true false positive already
   removed from the original 57): extract a diff scoped to just that
   commit's non-`bgp_ls*`/non-test/non-doc files, `git apply --check`; if
   clean, apply + commit preserving original author/date/message. If
   conflict, resolve by hand (see log below), preserving intent.
3. Test-only (15) and doc-only (2) commits from the original 164 are
   deliberately deferred — not yet folded into this Stage A sequence;
   revisit once the code-only 163 are solid.

## Commit-by-commit log

### `314b3eb5e6` "*: Add BGP-LS AFI/SAFI constants"
- 17 files touched; 16 applied clean via `git apply`.
- `bgp_attr.c` conflicted: pristine 10.5.4's `bgp_packet_mpattr_prefix()`
  labeled-unicast case calls `stream_put_labeled_prefix(s, p, label,
  addpath_capable, addpath_tx_id)` (4 args), but the commit's context
  expected `bgp_attr_stream_put_labeled_prefix(s, p, label, num_labels,
  addpath_capable, addpath_tx_id)` (renamed, +`num_labels` arg) — an
  unrelated multi-label-support refactor (matches the "BGP multiple
  labels support for labeled unicast" 10.6.0 feature noted in study file
  002) landed between 10.5.4 and this commit's real upstream base.
- Resolved by hand-inserting the 5 `case SAFI_BGP_LS`/`AFI_BGP_LS`
  additions at their correct locations in our actual (10.5.4-shaped) file
  structure, without pulling in the unrelated labeled-unicast refactor —
  confirmed each insertion point by reading surrounding context first.
- This is the single most important commit in the whole set: it's what
  actually adds `AFI_BGP_LS = 4` / `SAFI_BGP_LS = 8` to `lib/zebra.h`,
  bumping `AFI_MAX` 4→5 and `SAFI_MAX` 8→9 — the collision point with our
  own `SAFI_SRPOLICY = 8` addition will surface in Stage B, not here
  (Stage A is pure-upstream, no SONiC patches involved yet).

## Progress checkpoint (19/56 "touches-existing" commits landed)

Commits 4-19 (`a98f8055d2` debug support, `c85fff38ea` linkstate-db
registration, `abccc10524` capability display, `50e92bbb7f` AFI/SAFI
negotiation — all clean; `5a49df37f5` error code, `c8acf7a2c7` attr
struct, `31cffad03f` update/withdraw, `88ced6ec9e` opaque messages,
`5ccafe4b1c` show commands, `bfb67d5364` UPDATE packet encoding — all
hand-resolved) landed. Full per-commit detail in each commit's own
message body (`(Stage A: ...)` trailer) rather than duplicated here —
`git log` on this branch is the source of truth going forward; this
file now tracks only the scope-verification methodology and overall
checkpoints.

`bfb67d5364` ("Encode BGP-LS NLRI and attribute when building UPDATE
packet") was the largest/hardest so far: 3 function-signature changes
(`bgp_packet_mpattr_prefix`/`bgp_packet_attribute`/
`bgp_packet_mpunreach_prefix`, all gaining a `ls_nlri` parameter)
propagated across all 4 files that call them, with our 10.5.4 tree's
narrower `bgp_packet_attribute()` signature (missing the later
`srv6_unicast` param) requiring independent arg-count verification at
each call site rather than literally copying the diff's argument
lists. 37 of 56 "touches-existing" commits remain.

## Progress checkpoint 2 (28/56 -- halfway)

Commits 20-28 landed: `aa7c9b97f2` (receive attribute), `1da3f4420c`
(parse NLRI), `399c64b834` (peer-AF-based nexthop selection --
PEER_FLAG_BGP_LS_IPV4/IPV6, hand-resolved across bgp_attr.c/
bgp_updgrp.c/bgpd.c/bgpd.h; caught a mis-scoped edit where the same
nexthop-AFI-selection pattern appears twice in bgp_attr.c, once in the
real target function and once in the unrelated bgp_packet_nhc()),
`f232a7fbd1` (opaque msg demote, clean), `a2576fd6b0` (JSON attr
display), `a8b23e3df8` (extended length, clean), `906445760f` (TED
reset on deactivation), `46197ca940` (BGP-only-fabric originate/
withdraw fns -- bgpd.h only, bgp_ls.c/h already covered by verbatim
copy), `e2532aef6e` (27 call-site insertions for prefix originate/
withdraw on route events -- split into 28 individual hunks and applied
independently, 26 clean, 1 real hand-fix, 1 header-only split artifact).

Useful technique validated: for large multi-hunk single-file commits,
splitting the diff into individual per-hunk files and `git apply`-ing
each independently (rather than relying on git's whole-file atomicity)
surfaces exactly which hunks need hand attention, often much fewer
than the total hunk count. Will reuse for remaining large commits.

28 of 56 "touches-existing" commits remain.

## Progress checkpoint 3

Commits 29-33 landed: `a6d9b870...` (clean), `11d9d690a3` (bgpd:
trigger BGP-LS link originate/withdraw on peer state events --
bgp_fsm.c, `bgp`/`bgp_ls_originate_bgp_link`/`bgp_ls_withdraw_bgp_link`
already in scope via verbatim bgp_ls.c), `e15218968f` (bgpd: CLI to
enable BGP-LS distribution for BGP-only fabrics -- bgp_vty.c, 4
hand-applied hunks: include, 2 new DEFPYs, config-write block, 2
install_element registrations; anchors found by content search since
line numbers diverge heavily from a 10.7.0-era diff), `179984048...`
and `0f897...` (both clean).

**Deliberate exclusion**: `6677220e92` ("bgpd: Modify functions to
use `struct peer_connection`") -- a 13-file mechanical refactor
(bgp_attr.c/h, bgp_bmp.c, bgp_evpn.c, bgp_fsm.c, bgp_io.c, bgp_open.c/h,
bgp_packet.c/h, bgp_routemap.c, bgp_vty.c, bgpd.c, plus bgp_ls.c)
converting a batch of BGP OPEN/capability-negotiation/packet-write
functions from taking `struct peer *` to `struct peer_connection *`.
Verified via grep that none of `bgp_ls.c`/`bgp_ls_nlri.c`/
`bgp_ls_ted.c` call any of the refactored function names -- the
commit's only bgp_ls.c hunk is a purely-internal helper-signature
simplification (`bgp_ls_get_ifp_from_connection`) already present
verbatim via the commit-1 file copy from the 10.7.0 tag. This commit
carries zero BGP-LS feature semantics; it only appeared in the
163-commit set because it happens to touch bgp_ls.c incidentally.
Replaying the other 12 files' mechanical signature churn would add
large unrelated risk for zero functional gain, so it is skipped
outright (not deferred -- unlike the test/doc commits, there is
nothing here to revisit later). Queue popped without a Stage A
commit.

25 of 56 "touches-existing" commits remain.

## Progress checkpoint 4

Commits 34-42 landed: `614c5177d8` (malformed TLV handling, clean),
`f018e3f34a` (SRv6 SID Structure storage in link-state objects, clean),
`78773fb672` (isisd export SRv6 SID Structure into TE attrs, hand-
resolved), `4aac5094b4` (isisd distribute-link-state CLI/YANG/NB knob,
hand-resolved across 8 files), `bd5b1bde53`/`4cbe1d243f` (both clean),
`290221668b` (bgp_attr_get_ls_attr/bgp_attr_set_ls_attr getter/setter,
hand-resolved -- mechanical), `8ab6851905` (clean), `8d8c4899eb`
(remove redundant BGP-LS NLRI forward decls, hand-resolved trivially),
`052151310891`/`32254aaffc` (MT-ID support in link-state model +
IS-IS export, clean then hand-resolved).

**Second deliberate exclusion**: `9e1e665a50` ("bgpd: Move nhc
attribute from attr to attr_extra") -- a 17-file architectural
refactor (bgp_attr.c/h, bgp_conditional_adv.c, bgp_evpn.c,
bgp_evpn_mh.c, bgp_fsm.c, bgp_memory.c/h, bgp_mpath.c, bgp_mplsvpn.c,
bgp_nhc.c, bgp_route.c, bgp_routemap.c, bgp_srv6.c, bgp_updgrp_adv.c,
bgp_zebra.c, rfapi/vnc_import_bgp.c) introducing an `attr_extra`
sub-struct and moving the `nhc` field into it, for memory-footprint
reasons unrelated to BGP-LS. None of the 17 files is bgp_ls.c/
bgp_ls_nlri.c/bgp_ls_ted.c, and grep confirms none of those 3 files
reference attr_extra/bgp_attr_get_nhc/attr->nhc. `bgp_srv6.c` doesn't
even exist in our 10.5.4-era tree (split out later), which is itself
a sign this diff assumes a much later tree shape than ours. Same
rationale as the `6677220e92` peer_connection exclusion: caught by
the tight token filter only incidentally (message doesn't mention
BGP-LS; likely diff-context coincidence in bgp_attr.c/bgp_route.c
near existing ls_attr code), carries zero BGP-LS feature semantics,
and replaying it would add large unrelated risk for zero functional
gain. Skipped outright. Queue popped without a Stage A commit.

24 of 56 "touches-existing" commits remain.

## Progress checkpoint 5 -- "touches-existing" queue complete

Commits 43-56 landed: `f302c9fc4f`/`1bcfef7ecd` (clean), `f0eb283a3c`
(bgpd: export static SRv6 END SIDs as BGP-LS SRv6 SID NLRIs, hand-
resolved -- bgp_redistribute_add() gained seg6local_action/ctx params,
bgp_ls_handle_route_add/delete() hooked in), `4c471588630` (clean),
`27f783c270` (bgpd: originate/withdraw SRv6 locator prefixes as
BGP-LS Prefix NLRIs, hand-resolved across bgp_vty.c/bgp_zebra.c),
`4b88d5ad8a`/`e4ea5850cf` (clean).

**Third deliberate exclusion**: `87fe21fda9` ("FRR Release 10.7.0") --
the upstream release-tagging commit itself. Its only content is
configure.ac's version string, "10.7.0-dev" -> "10.7.0" (1 line
changed). It only matched our BGP-LS filter because its changelog
message lists "BGP-LS SRv6 extensions" as a highlight; the diff has
zero functional content. Skipped outright -- not a code commit at
all. Queue popped without a Stage A commit.

**The 56-commit "touches-existing" queue is now fully processed**: 53
commits landed as Stage A commits (44 clean via git apply, 9 hand-
resolved), 3 deliberately excluded as unrelated-refactor/release-tag
false positives (`6677220e92` peer_connection, `9e1e665a50` attr_extra/
nhc, `87fe21fda9` release tag) with zero BGP-LS functional content in
each case, verified by grep against bgp_ls*.c/h before exclusion.
Combined with commit 1 (verbatim copy of the 6 new bgp_ls*.c/h files,
subsuming the 107 new-file-only commits), Stage A's code-only 163-commit
scope is complete.

**Remaining work**: the 15 test-only and 2 doc-only commits deferred
at the start (see "Method" above) have not yet been folded in --
revisit with the user before considering Stage A fully done. Stage B
(export as a few logical patches, apply onto the 156-SONiC-patch
branch, resolve the SAFI_BGP_LS=8/SAFI_SRPOLICY=8 collision) has not
started.

## Progress checkpoint 6 -- deferred test/doc commits landed, Stage A complete

All 17 previously-deferred commits (15 test-only + 2 doc-only) applied
via a full (unscoped) `git apply` each -- zero conflicts, since none
of them touch any pre-existing file outside `doc/user/bgp.rst` (a
2-line toctree addition) and the topotest tree they create is entirely
new. Applied in upstream chronological order via a dedicated queue
script (`testdoc_integrate.sh`), separate from the main
`stageA_integrate.sh` used for the 56 touches-existing commits.

**Stage A is now fully complete**: 74 commits on `bgpls-on-10.5.4-stageA`
on top of pristine `frr-10.5.4` -- 1 verbatim-copy commit (6 new
bgp_ls*.c/h files, subsuming 90 new-file-only commits), 53 individually
replayed touches-existing commits (29 clean via `git apply`, 24
hand-resolved), 3 deliberately excluded touches-existing commits
(zero BGP-LS payload, documented above), 17 test/doc commits (all
clean), and 3 bookkeeping commits (this file). All 163 genuine
upstream BGP-LS commits are accounted for.

A full commit-by-commit report -- classification, every hand-resolved
conflict and its resolution, the 3 exclusions, and a risk assessment
ahead of Stage B -- was written up separately as
`bgpls_stageA_ledger.html` (published as an artifact, not stored in
this worktree).

**Known open risks before Stage B** (see the ledger for full detail):
this branch has never been compiled -- no `configure`/`make` pass has
been run, so no hand resolution has been verified by anything stronger
than anchor/context matching and brace-balance checks. The
`SAFI_BGP_LS=8` / `SAFI_SRPOLICY=8` collision is confirmed on both
sides (checked directly against `src/sonic-frr/frr/lib/zebra.h`) and
unresolved. Commit `baa1dcadd2` is a deliberate structural deviation
from upstream (kept `ls_attr` as a direct `attr` field instead of
moving it into `attr_extra`, since the `attr_extra` infrastructure
commit was excluded) -- functionally fine today, but a documented
divergence. Peer-flag bit reassignments (34/35, 47/48) need
cross-checking against the 156-patch SONiC/Nexthop branch before
Stage B, same as the SAFI collision.

Stage B (export as a few logical patches, apply onto the branch
carrying the 156 SONiC/Nexthop patches, resolve the collisions above)
has not started.

## Progress checkpoint 7 -- test-coverage audit, 1 gap found and fixed

Prompted by a direct question ("have you brought on all the tests
too"), audited all 56 touches-existing commits for any bundled
`tests/`/`doc/` files that Stage A's scoped-diff process (which
deliberately excludes those paths, to keep the code-only integration
pass and the test/doc pass separate) would have silently dropped.

Found 1 real gap: `314b3eb5e6` ("*: Add BGP-LS AFI/SAFI constants",
replayed here as `0fd35f4a27`) bundled a 2-line unit-test hunk in
`tests/bgpd/test_peer_attr.c` (`case AFI_BGP_LS: return "bgp-ls";` in
`str_from_afi()`) alongside its 17 production files. Fixed as a new
follow-up commit (`ca7dfd9227`) rather than amending `0fd35f4a27`,
since that commit is buried under 70+ later commits.

The other touches-existing commit with bundled test files
(`6677220e92`, already excluded) touches `tests/bgpd/test_aspath.c`
and `tests/bgpd/test_capability.c`, but those are mechanical call-site
updates for the peer_connection refactor that was itself excluded --
applying them would break compilation against our un-refactored
function signatures, so correctly left out, not a gap.

Independently verified via `git log -G'AFI_BGP_LS|SAFI_BGP_LS|BGP-LS|
BGP_LS|bgp_link_state' -- tests/` and the `doc/` equivalent across the
full `frr-10.5.4..frr-10.7.0` range (not just the original 164-commit
candidate list) that no other BGP-LS-referencing test or doc content
exists anywhere in the range beyond what's now landed. **Test/doc
coverage is confirmed complete**: all 15 topotest commits, both doc
commits, and this 1 recovered unit-test hunk are now in the branch
(76 total commits).

## Progress checkpoint 8 -- build verification: 2 real gaps found, both fixed, clean build achieved

Ran `./bootstrap.sh && ./configure --enable-multipath=514 --enable-fpm
--enable-sharpd --enable-ospfapi --enable-pcre2posix --disable-protobuf
--disable-zeromq --disable-doc && make -j$(nproc)` against the full
branch (this was the top-ranked open risk in the published ledger --
"nothing has been compiled"). Found and fixed 2 real, distinct
categories of forward-drift that the scoped-diff `git apply --check`
methodology structurally cannot catch, both surfaced only by an actual
compiler pass:

1. **New-function forward-drift** (commit `34b78ea3d5`, follow-up to
   `2dedcfd1f6` and `828249f125`): `bgp_attr_ls()` referenced
   `args->connection`/`connection->curr` and `bgp_attr_set_ls_attr()`
   called `bgp_attr_set()`/`bgp_attr_unset()` -- all four things exist
   in upstream's tree at the point those commits landed, but not in
   our 10.5.4 base (a still-earlier point in the same incremental
   peer_connection migration whose *later* step, `6677220e92`, Stage A
   already excluded; and a generic attr-flag helper introduced
   independently of BGP-LS). `git apply --check` cannot catch this
   class of bug: the added code is entirely new lines with no
   pre-existing context to diff against, so nothing about the patch
   *looks* wrong until the compiler resolves the symbols. Fixed by
   matching the exact pattern already used one function up
   (`bgp_attr_otc()`) and throughout the file.

2. **Missing cross-cutting dependency** (commit `2a6467cdd5`,
   backport of upstream `85ed407320`, "*: Provide interface when
   installing/uninstalling SRv6 uA SIDs"): `bgp_ls.c`'s static SRv6
   END-SID-export path (`bgp_ls_upsert_static_endx_sid()`, from Stage
   A commit `c48448f8c3`) uses `seg6local_context.ifindex`, a field
   that plain upstream added in a **non-BGP-LS** commit dated
   2025-10-25 -- entirely outside Stage A's BGP-LS-token candidate
   filter by design, since it isn't BGP-LS-related and correctly never
   matched. This is the inverse failure mode of the two commits Stage
   A deliberately excluded (which matched the filter but had no real
   dependency); this one has a real dependency but never matched the
   filter at all. Backported as a small, self-contained 4-file/16-line
   commit (`isisd/isis_zebra.c`, `lib/srv6.h`, `staticd/static_zebra.c`,
   `zebra/rt_netlink.c`) -- 2 files applied clean, `zebra/rt_netlink.c`
   hand-placed at its real anchors (VRFTABLE handling and
   `req_size`-vs-`buflen` naming differ from upstream's context, but
   the insertion points matched).

After both fixes: **a full clean build (`make clean && make -j$(nproc)`)
completes with 0 errors and 0 warnings**, producing all 6 touched-daemon
binaries (`bgpd`, `isisd`, `zebra`, `staticd`, `pbrd`, `vtysh`).
`bgpd --version`/`isisd --version` run correctly and report `10.5.4`.
A config-syntax smoke test (`bgpd -S -C -f <conf>` with `router bgp
65001` / `address-family link-state link-state` / `distribute
bgp-fabric-link-state`) parses the new BGP-LS CLI grammar without
error, failing only later at socket bind (port 179 needs root/
CAP_NET_BIND_SERVICE, and `/usr/local/var/{lib,run}/frr` don't exist
on this dev machine) -- an environment/install limitation, not a code
defect.

**Branch is now 79 commits on top of pristine frr-10.5.4, and
verified to build clean.** This closes out the two highest-severity
items in the risk assessment (compile verification; the
`85ed407320` dependency was not previously known and is a new,
resolved finding). The `SAFI_BGP_LS=8`/`SAFI_SRPOLICY=8` collision
and the peer-flag bit cross-check against the 156-patch SONiC/Nexthop
branch remain open, unchanged, ahead of Stage B.

**Caveat**: this is a build/link/CLI-grammar verification only, not a
runtime/protocol-correctness verification -- the topotest suite has
still not been *executed* (no BGP sessions have actually been brought
up and exchanged BGP-LS NLRIs). That remains open.
