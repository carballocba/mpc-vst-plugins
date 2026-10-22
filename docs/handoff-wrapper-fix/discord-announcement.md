# Discord announcement (draft, not posted)

Drafted 2026-10-22. Discord's limit is 2000 characters per message, so this is two posts: the news, then the how-to for developers.

## Post 1: what changed

**Q-Links now work on the automation screen: wrapper fix, please rebuild** 🎛️

Some of our plugins had a bug: on track edit → automation, the physical Q-Links wouldn't move a plugin parameter, or it snapped back, while the on-screen knobs worked. Big thanks to the port author who tracked it down.

**Cause:** our VST2 wrapper told MPC to rebuild its parameter screen on every parameter write. A Q-Link turn is a stream of writes, so the screen kept rebuilding under your finger.

**Fix** (mpc-vst-plugins #486 to #489):
- A plain parameter write no longer asks for a screen rebuild. Prev/next buttons still do.
- At most one rebuild per 50 ms.
- Port authors can opt a parameter back in with `"refresh_display": true`.
- The build now writes `params.lock` and fails if a parameter's index changes, so a new parameter can't silently break saved projects, automation or Q-Link maps.

**For players:** all our own wrapper-based plugins have new releases with the fix: Mini-JV, Machinemodule, Boris, Clementine-XT, Model-72, Dexed, Morpho-PE, Sturm, Profit-08, Daemon, TR-MPC, Luckydip, Maze Voice and OTT. Update them through the plugin manager or from the releases pages. Checked on a Force with the Mini-JV; the rest are tested offline and being checked on device.

**Existing builds are not fixed:** the wrapper is compiled into each plugin, so every author has to rebuild. Developer details are in the next message.

## Post 2: what it means for developers

**Developers: how to get the fix**

Your plugin keeps the bug until you rebuild against the new wrapper.
1. Update the wrapper: `git pull` in mpc-vst-plugins, or bump your submodule, or if you vendored `vst2_wrap.c`, copy in the new one plus the new `tools/gen_vst.py` (`param_t` gained a field).
2. If a label or readout follows another parameter and isn't polled (a `dynamic_name` / `dynamic_display` label, or a `"poll": false` readout bound with `get=`), add `"refresh_display": true` to that parameter. Otherwise it stops following. Most ports need nothing.
3. `tools/test_port.sh <vst.json>` must pass, then `tools/build_port.sh <vst.json>`.
4. Test Q-Links on **track edit → automation** on a device, and check any stepper readouts.
5. Commit the new `params.lock`. From now on, add parameters at the end of the list.
6. Bump the version, release, and send a catalog PR (docs/RELEASING.md).

Also new in docs/PORTING.md: engine lock rules (no file or network I/O in set/get), poll cost, and why render is not a heartbeat. Full write-up and questions: <DISCUSSION_LINK>
