# Sniff Subrating response compatibility (draft)

On cannong with `22.2-20261102-UNOFFICIAL-cannon` and Redmi Buds 8 Lite,
Bluetooth repeatedly aborts after the controller returns Command Status
`UNSUPPORTED_REMOTE_OR_LMP_FEATURE` (`0x1a`) for `SNIFF_SUBRATING` (`0x0811`).
The HCI layer was waiting for Command Complete. Eight identical process
crashes were captured in a 194-second reproduction. The phone itself stayed
running; the separately investigated low-voltage resets are outside this fix.

This directory carries the matching platform change because the failure is in
`packages/modules/Bluetooth`, not in a device-tree implementation. The product
opt-in in `system.prop` requires the platform patch below. It does not implement
the behavior by itself. There is no automatic patching hook.

## Source and integration

- Platform: `LineageOS/android_packages_modules_Bluetooth`, `lineage-22.2`,
  baseline `0d2d485867791015ef4ee235c2a668b83bea1f1d`.
- Device: `jacksedart/android_device_xiaomi_cannon`, `lineage-22.2`,
  baseline `71122c811cb168590b8b4d364e183c975a72d19f`.

Readable platform commit: [de2f41c](https://github.com/joedexdev/android_packages_modules_Bluetooth/commit/de2f41ca771327113e40b6ae4fcaeede9d62358b).
The carried mail patch contains the same changes and preserves its author.

These are public reference revisions, not a verified source manifest for the
installed ROM. Review against the actual build checkout, especially if it has
other Bluetooth patches. Start with a clean platform working tree.

From the Android source root, check and then apply the platform commit:

```sh
git -C packages/modules/Bluetooth apply --check \
    ../../../device/xiaomi/cannon/patches/bluetooth/0001-Bluetooth-add-an-opt-in-SSR-error-status-quirk.patch
git -C packages/modules/Bluetooth am \
    ../../../device/xiaomi/cannon/patches/bluetooth/0001-Bluetooth-add-an-opt-in-SSR-error-status-quirk.patch
```

`BoardConfig.mk` already includes `system.prop` in the system image.

The path is relative to `packages/modules/Bluetooth`, as selected by `git -C`.
Build with the maintainer's established product/release configuration and
signing pipeline. Keep the platform patch and device opt-in paired in the
build manifest/process. Do not merge the opt-in as a claim that an otherwise
unpatched build is fixed. If adopted upstream, remove the carried patch only
after verifying the integrated implementation and its opt-in semantics.

## Scope and regression controls

The platform change is enabled only by the new property
`bluetooth.hci.sniff_subrating_status_quirk.enabled=true`. It defaults false.
The candidate handles only a valid `0x1a` Command Status for a pending SSR
command that expects Command Complete. It builds a full typed failed
`SniffSubratingComplete`, preserving the command's connection handle, status
and credits, then uses normal queue advancement and timeout cleanup.

It does not change normal completions, other opcodes or error statuses,
peer-feature discovery, firmware, codecs, profiles, pairing, encryption,
Sniff/SSR enablement, or SELinux policy. It does not fabricate success or
broadly suppress assertions. The existing `bluetooth.` property context and
Bluetooth property access were checked in the public 22.2 policy; confirm the
installed product property and permissions in the actual build.

Seven host regression cases are included: typed error/handle preservation
with queue progress; zero credits; normal success completion; disabled opt-in;
unexpected success status; another error status; and another opcode. The
normal typed GD completion checker can consume the full failed response.
The legacy SSR completion path retains its existing default logging behavior.
Repeated nonfatal SSR errors may still be logged; this is not a controller
firmware repair or peer-capability learning mechanism.

## Validation and merge gate

Completed locally: patch application/reversal and whitespace checks against
the pinned public baselines, a `git am` round trip using the documented relative
path with an identical resulting Git tree, callback-path review, formatting of
changed C++ ranges, and actionlint validation of the workflow. The accompanying CI checks patch applicability only; it does not
compile or execute C++ tests.

The following remain required before treating this as a validated fix:

- [ ] Build and run `bluetooth_test_gd_unit` on a supported Linux Android builder,
      including `HciSsrStatusQuirkTest.*`, `HciSsrStatusQuirkDeathTest.*` and the
      existing HCI tests.
- [ ] Build the complete product and verify the opt-in is present and readable
      without property-access denials.
- [ ] Reproduce with Redmi Buds 8 Lite: Bluetooth stays alive, crash count stays
      stable, audio works, and no HCI command-credit stall or timeout appears.
- [ ] Exercise media pause/resume, idle transitions and repeated reconnections.
- [ ] Test headset call/microphone routing and at least one other headset.
- [ ] Compare sleep/idle power and verify unpatched/flag-disabled behavior as
      appropriate. No measured battery-regression claim is made here.

For source rollback, revert the applied platform commit and revert/remove the
matching product opt-in before rebuilding. Runtime rollout and rollback use
the maintainer's normal matching-ROM process. No experimental image has been
built or flashed as part of this proposal.

## Prior work and upstream routing

The approach follows the existing HCI compatibility path and prior
[TrebleDroid handling of the same error](https://github.com/TrebleDroid/platform_packages_modules_Bluetooth/commit/cd1aff46e5a863e8e891b29008bf7e0c5757290e).
This candidate narrows the condition to the captured opcode/error, adds an
opt-in, and synthesizes the full typed SSR result rather than a short generic
completion. It is an adapted proposal, not a cherry-pick or accepted upstream fix.

[LineageOS platform submissions use Gerrit](https://lineageos.github.io/lineage_wiki/how-to/submitting-patches/).
The device PR carries this patch for jacksedart's review/integration because no
separate public jacksedart Bluetooth-module fork was found. It remains a draft
until native build and hardware validation are completed.
