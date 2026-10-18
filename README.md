# Samsung TV cleanup: a tested BU8000 procedure

Remove unwanted Home recommendations while preserving the app launcher and
TizenBrew. This is a field report and reversible procedure, verified on **one
Samsung UA75BU8000KXXL running Tizen 6.5, firmware T-KSU2EBUABC-1720.7**.

On the tested TV, disabling Samsung TV Plus and several advertising services did
**not** remove the “Free Movies from Samsung TV Plus” and “Top Picks” rows.
Those rows disappeared after we additionally disabled
`com.samsung.tv.ContentsBrowser` and reloaded Home Media. The owner confirmed the
visible result.

The package files were retained. This procedure disables components; it does not
recover their storage space. It does not establish that all advertising or
tracking is blocked.

This repository contains a tested manual and investigation record. No TV
application, proprietary binaries, or automatic bulk-removal tool is bundled.

## What was verified

| Check | Result |
| --- | --- |
| Home recommendations after the initial TV Plus cleanup | Still visible |
| `ContentsBrowser` app flag after the additional change | Disabled, confirmed by database readback |
| Home component list after reloading | No `ContentsBrowser` references in the newly written cache |
| Visible recommendation rows after the additional change | Gone, confirmed by the owner |
| Main Home launcher | Remained running |
| TizenBrew and TizenTube installations | Preserved; their application flags remained enabled |
| Recommendation change after another full reboot | Not independently verified |
| Recommendation change alone on an otherwise untouched TV | Not tested |
| Other TV models, regions, or firmware builds | Not tested |
| Permanent suppression of all tracking services | Not established; ACR restarted during a subsequent TV session |

See [VERIFIED-RESULTS.md](VERIFIED-RESULTS.md) for the complete change sequence,
artifact identities, and limitations. Treat that sequence as a record of this
device, not a universal removal list.

## Why disabling TV Plus was insufficient

The TV has separate components for playing TV Plus, delivering ads, and building
Home recommendations. We found the Home recommendation data in the Home Media
cache and traced its component definitions to `ContentsBrowser` / `CompTerra`.
Home's app-launcher component is separate.

One particularly misleading detail: the setting
`db/mw/number_of_virtualservices` already held the value that the inspected code
calls `VALUE_TVPLUS_DISABLED`. The recommendation rows were still visible. We
did not change that setting or the TV's country configuration.

Do **not** disable `com.samsung.tv.coba.adagent` simply because its name includes
“adagent.” It supplies resources used by Smart Hub. We deliberately preserved it.
The separate `com.samsung.tv.adagent-service` belongs to a shared PISA package;
disabling that one application is different from disabling its entire package.

## Connection and tooling

The tested host was Linux, using Samsung SDB 4.2.36 and Python 3.14. Python 3.12+
is required by the referenced command-transport project. The TV and computer
must be on the same local network.

1. Obtain Samsung's SDB from its official Tizen tooling. The tested standalone
   archive is
   [sdb_4.2.36_ubuntu-64.zip](https://download.tizen.org/sdk/tizenstudio/official/binary/sdb_4.2.36_ubuntu-64.zip).
   Its executable is `data/tools/sdb`. No SDB binary is included here.
2. Open **Apps** on the TV and enter **12345** to open Developer Mode. Set it to
   **On**, enter the computer's current local IPv4 address as **Host PC IP**, and
   fully restart the TV. See
   [Samsung's device connection instructions](https://developer.samsung.com/smarttv/develop/getting-started/using-sdk/tv-device.html).
3. Connect and inspect the TV. Replace the example address below with your TV's
   address; `192.0.2.50` is a documentation placeholder.

```bash
export SDB=/absolute/path/to/data/tools/sdb
export TV_IP=192.0.2.50

"$SDB" connect "$TV_IP:26101"
"$SDB" devices
"$SDB" -s "$TV_IP:26101" capability
"$SDB" -s "$TV_IP:26101" shell 0 applist
```

On this TV, capabilities reported `intershell_support:disabled` and
`sdbd_rootperm:disabled`. Ordinary `sdb shell` therefore did not provide a Linux
shell.

### The command transport used here

The commands in the following sections are **TV-side shell commands**. Do not
paste them into the host shell or assume that `sdb shell '<command>'` can run
them on retail firmware.

Our session used the SDB package-name command-execution path implemented by
[Chris Ritsen's samsung-tv-root project](https://github.com/chris-ritsen/samsung-tv-root).
That path exploits the package installer to execute commands as the restricted
SDK user. Developer Mode alone is not the same capability. We confirmed
`uid=901(sdk)`; we did **not** obtain root, run a kernel exploit, flash firmware,
or install a persistent agent.

The upstream checkout used for the session was:

```text
4ad2c4d3b0d0e0da4b5b7ebe4a787e94596f416b
```

An equivalent host-side Python session can use that checkout's transport without
invoking its root commands:

```bash
git clone https://github.com/chris-ritsen/samsung-tv-root.git
git -C samsung-tv-root checkout 4ad2c4d3b0d0e0da4b5b7ebe4a787e94596f416b
export PYTHONPATH="$PWD/samsung-tv-root/src"
python3
```

In the Python session, initialize a connection and a checked command helper:

```python
import os
from pathlib import Path
from samsung_tv_root.sdb import SdbClient

client = SdbClient(Path(os.environ["SDB"]), os.environ["TV_IP"])
client.connect()
client.require_device()

def tv(command):
    result = client.capture(command, timeout=20)
    print(result.output)
    body, marker, status = result.output.rpartition("\n[exit:")
    if not marker or status.strip() != "0]":
        raise RuntimeError("TV command failed or its completion was not verified")
    return body.strip()

tv("id")
```

The transport opens a temporary callback listener on the computer; the TV must
be able to connect back to it. It can also stage self-removing command files in
the TV's SDK staging directory. Only use this procedure with your own TV on a
trusted local network. Do not expose the TV's developer ports to the internet.

Compatibility of that command-execution path is firmware-dependent. If the TV
does not connect back, stop before making changes. Inspect the selected TV,
Developer Mode host address, route, and host firewall. A timeout is not proof
that a mutation failed: read the actual state before retrying a change.

## Read the original state first

In the same Python session:

```python
import shlex

DB = "/opt/dbspace/.pkgmgr_parser.db"
GUIDE = "com.samsung.tv.ContentsBrowser"
HOME = "com.samsung.tv.home.media"

def sql(query):
    return tv("sqlite3 -readonly " + DB + " " + shlex.quote(query))

owner_uid = tv("stat -c %u /opt/usr/home/owner")
if not owner_uid.isdecimal() or int(owner_uid) == 0:
    raise RuntimeError("Could not establish the owner account")

original = sql(
    "SELECT app_id,app_disable FROM package_app_info "
    f"WHERE app_id='{GUIDE}';"
)
if original not in (GUIDE + "|false", GUIDE + "|true"):
    raise RuntimeError("Recommendation component not found or state is unknown")

print("Owner UID:", owner_uid)
print("Original recommendation state:", original)
tv("app_launcher -S")
```

The tested owner UID was `5001`; discover yours instead of assuming that value.
Save the original state privately. If the guide is already disabled, this
procedure does not establish why recommendations remain on your firmware.

## Disable the recommendation component and reload Home

The following is the **additional change** that removed the rows in our modified
setup. It is a reasonable single-component experiment on a matching device,
but its effectiveness without the earlier cleanup has not been isolated.

```python
if original != GUIDE + "|false":
    raise RuntimeError("Guide was not enabled; do not repeat this change blindly")

tv(f"pkgcmd -D -t app -n {GUIDE} --uid {owner_uid}")
after = sql(
    "SELECT app_id,app_disable FROM package_app_info "
    f"WHERE app_id='{GUIDE}';"
)
if after != GUIDE + "|true":
    raise RuntimeError("Disable was not verified; stop and inspect before continuing")

tv(f"app_launcher -u {owner_uid} -k {HOME}")
tv(f"app_launcher -u {owner_uid} -s {HOME}")
```

This briefly closes and restarts Home Media. Press **Home** on the physical
remote afterward. Check that:

- The app launcher is usable and your wanted apps remain available.
- The unwanted recommendation rows have disappeared.
- TizenBrew and TizenTube still work.

Disabling this component can also remove wanted Universal Guide features and
recommendations from other services. If Home becomes blank or navigation breaks,
restore the component immediately using the commands below. If the restart
command failed, attempt the Home start command again after restoring it.

Run another full restart and repeat the checks before describing the result as
persistent. Also recheck after firmware or Smart Hub component updates.

### Optional component-cache verification

The tested Home Media data directory was:

```text
/home/owner/apps_rw/com.samsung.tv.home.media/data/
```

Read `components.cache.1` and `components.cache.2` through the command transport.
Compare their version headers; the `.1` suffix does not always mean newest.
After our change, the newer cache had no `com.samsung.tv.ContentsBrowser`
references. This confirms component discovery changed, **not** that the visible
screen is correct. Visual confirmation is still needed.

## Restore the recommendation component

If it was enabled before your experiment, use the same Python connection:

```python
tv(f"pkgcmd -A -t app -n {GUIDE} --uid {owner_uid}")
restored = sql(
    "SELECT app_id,app_disable FROM package_app_info "
    f"WHERE app_id='{GUIDE}';"
)
if restored != GUIDE + "|false":
    raise RuntimeError("Restore was not verified")

tv(f"app_launcher -u {owner_uid} -k {HOME}")
tv(f"app_launcher -u {owner_uid} -s {HOME}")
```

Press Home and check navigation. Do not delete Home databases, reset Smart Hub,
or uninstall shared launcher packages as a substitute for this rollback.

For the earlier experimental changes, see the separate rollback description in
[VERIFIED-RESULTS.md](VERIFIED-RESULTS.md#restoring-the-earlier-cleanup).

## Returning to TizenBrew

Some TizenBrew installation workflows require **Host PC IP = `127.0.0.1`** so the
TV can use its own local developer connection. The owner restored that address
after this investigation. Follow the instructions for your installed
[TizenBrew workflow](https://github.com/reisxd/TizenBrew/blob/main/docs/README.md).

That setting does not itself re-enable the disabled components. If you need
external access for rollback, temporarily set Host PC IP to the computer's
current local address again and restart the TV.

## Scope and contributions

This is a tested manual, not a universal debloater. No automatic cleanup script,
TV binary, certificate, or proprietary assembly is distributed. New reports
should include the exact model, firmware, Tizen version, individual change,
readback, visual result, rollback result, and reboot result. Mark failures and
unknowns explicitly.

Do not publish raw device-information responses, IP/MAC addresses, serial numbers,
DUIDs, pairing tokens, account data, or full content/history caches. The artifact
hashes in the results document identify the inspected software, not the TV owner.

## References

- [Samsung: disable Samsung TV Plus](https://www.samsung.com/au/support/tv-audio-video/disable-samsung-tv-plus/)
  documents the UI disable flow; it is regional guidance, not proof that every
  Home recommendation row disappears on the tested Indian model.
- [Samsung: connecting a TV](https://developer.samsung.com/smarttv/develop/getting-started/using-sdk/tv-device.html)
  documents normal developer setup.
- [samsung-tv-root](https://github.com/chris-ritsen/samsung-tv-root)
  supplied the SDK command transport; no root session was required here.
- [Vetro](https://github.com/andrearicci93/vetro)
  informed the package-state investigation and the protected Smart Hub dependency.
- [REA](https://github.com/morluto/rea)
  supplied offline managed-assembly inspection.

## License

This documentation is licensed under the [MIT License](LICENSE).
Copyright (c) 2026 kbssrikar7.

Referenced tools and Samsung software retain their respective licenses and are
not bundled in this repository.
