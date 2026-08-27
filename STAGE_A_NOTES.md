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
