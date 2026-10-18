# Verified results and experimental history

Investigation date: **2026-10-18**. One owner-operated TV was investigated.

## Target and environment

| Property | Observed value |
| --- | --- |
| Model | Samsung UA75BU8000KXXL |
| Firmware reported by owner | T-KSU2EBUABC-1720.7, BT-S |
| Platform reported by SDB | Tizen 6.5, ARMv7, 32-bit |
| Kernel reported by the TV | Linux 5.4.77 |
| Host | Linux |
| SDB | Samsung SDB 4.2.36 |
| Command identity | `uid=901(sdk)`, SMACK context `System` |
| Owner UID | `5001`, read from owner-home directory ownership |
| Normal interactive shell / SDB root permission | Disabled |
| Command transport source revision | `4ad2c4d3b0d0e0da4b5b7ebe4a787e94596f416b` |

No root session was established. The upstream archive-root compatibility check
stopped when Samsung's custom `dotnet` rejected `--list-runtimes`. Subsequent work
used the already functioning SDK command-execution transport. Neither the
archive-root exploit nor a hardware-specific root exploit was run.

## First pass: playback, advertising, and recognition components

The following application flags were originally enabled (`app_disable=false`).
They were changed individually with:

```text
pkgcmd -D -t app -n <app-id> --uid 5001
```

| App ID | Package ID | Additional package-level disable |
| --- | --- | --- |
| `com.samsung.tv.tvPlusDeeplinker` | `com.samsung.tv.tvPlusDeeplinker` | Yes, `tpk` |
| `com.samsung.tv.tvplus` | `com.samsung.tv.tvplus` | Yes, `tpk` |
| `adplatform.ADPlayer` | `adplatform` | Yes, `wgt` |
| `com.samsung.tv.targeted-ad-service` | `com.samsung.tv.targeted-ad-service` | Yes, `tpk` |
| `com.samsung.tv.acr` | `com.samsung.tv.acr-service-app` | Yes, `tpk` |
| `com.samsung.tv.iacr` | `com.samsung.tv.iacr` | Yes, `tpk` |
| `com.samsung.tv.adagent-service` | `com.samsung.tv.pisa-service` | **No: shared package preserved** |

**These are historical changes on the tested TV, not a recommended bulk-removal
list.** Their independent necessity and compatibility on other devices were not
established.

Application-level disabling did not prevent already running native targeted-ad
and ACR services from reappearing during the session. Attempting to stop an
already disabled app with `app_launcher` returned “not available for the user.”
A temporary enable → stop as owner → disable sequence stopped each process
briefly, but targeted advertising returned.

Package-level disabling used:

```text
pkgcmd -D -t <tpk-or-wgt> -n <package-id> --global
```

Samsung emitted uninstall-style progress messages for this operation, but the
package database retained the entries with `package_disable=true`. Files for the
targeted-ad package were confirmed to remain on disk. Its package-level
disable → enable → disable round trip was verified by database readback.
Do not interpret these messages as proof that storage was reclaimed.

After that pass, all seven app flags and six package flags read disabled, and
none of the seven targeted apps appeared in the running-app inventory after
opening Home. TizenBrew, the installed YouTube/TizenTube-related application,
Home components, the app store, and the protected
`com.samsung.tv.coba.adagent` component remained enabled.

**Visible result:** the owner still saw Samsung TV Plus recommendations, free
movies, and top-picks rows. The first pass had not achieved the Home-screen goal.

## Second pass: the recommendation component

Inspection found the following in Home Media's data:

- A `.ContentsBrowserCache` containing “For You,” “Live,” and “apps” tab data,
  content-row identifiers, and recommendation-provider metadata.
- Component mappings connecting recommendation components to
  `com.samsung.tv.ContentsBrowser` and its `CompTerra.dll` assembly.
- A separate core Home component supplying the launcher.

The inspected code's `DeviceInfo.get_isTVPlusEnabled` reads the cached
`numberOfVirtualServices` value, initialized from
`db/mw/number_of_virtualservices`. `CommonDefines.VALUE_TVPLUS_DISABLED` is a
static readonly integer initialized to **zero**. The live value already read
zero while recommendation rows remained visible. It was left unchanged.

The Universal Guide configuration also contained `SupportUGCountry:true`.
No country, region, consent, or Universal Guide configuration value was edited.

The `ContentsBrowser` application and package flags both originally read
enabled. Only its **application flag** was changed:

```text
pkgcmd -D -t app -n com.samsung.tv.ContentsBrowser --uid 5001
```

The database then reported:

```text
com.samsung.tv.ContentsBrowser|true
```

Home Media was stopped and restarted:

```text
app_launcher -u 5001 -k com.samsung.tv.home.media
app_launcher -u 5001 -s com.samsung.tv.home.media
```

The start command reported success and a new running Home process. After opening
Home, the newly written `components.cache.2` (version 11) contained **zero**
`com.samsung.tv.ContentsBrowser` references; the previous `.1` cache (version 10)
contained 20. The cache suffix alone was not used to determine freshness.

**Visible result:** the owner confirmed that the recommendation rows disappeared.
This is the confirmed UI outcome. The guide and rollback were retained, and the
owner subsequently restored Developer Mode Host PC IP to `127.0.0.1` for
TizenBrew's local workflow.

This experiment was performed after the first pass. It does not prove that the
recommendation-component change alone is sufficient on an untouched TV.

## Persistence and remaining limits

- A later TV session had new core process IDs, consistent with a restart.
  TV Plus and targeted-ad package flags still read disabled. The ACR package
  flag instead read enabled, and an ACR process was running. The mechanism that
  restored it was not determined.
- The owner confirmed the recommendation-row change before another independently
  verified full reboot of that final configuration. No post-reboot readback or
  visual check of that change was captured.
- No firmware-update survival test was performed.
- No packet capture established complete suppression of tracking or advertising
  requests. DNS filtering was not configured.
- No storage savings or performance improvement was measured.
- Installed/enabled state was checked for preserved apps. This investigation did
  not independently run a post-change playback test in every streaming app.
- Disabling `ContentsBrowser` may remove other Universal Guide functionality.
  Other models may couple these components differently.

## Inspected software and evidence

The artifacts were read from the owner's TV and inspected offline without loading
or executing them on the host. No proprietary binaries or raw content caches are
included in this folder.

| Artifact | SHA-256 | Module MVID |
| --- | --- | --- |
| `HomeMedia.dll` | `b5f94d987b16c9f99eb21d1f8b80933fa807046d3d57d9cb71cfbb40f6e62545` | `14d115a4-136f-4188-98ff-cd8679c23249` |
| `CompHome.dll` | `861d6d2e309d8427f4d24f0e5dd28ddfa33fb0c7db60b321e215a454e08a9352` | Not recorded in this public report |
| `CompTerra.dll` | `f42d0e984844a5d8b6a096e60ecc28d99664d19fc73905a9398c8f8e71a81176` | `6b530eef-3162-4a43-80e1-d396c0c1ae10` |

REA session evidence references:

| Inspection | Evidence ID |
| --- | --- |
| HomeMedia managed members | `ev_96ce816cd54ee12b2655ab7e6d60dc414ea2c86e2cb97af0d5ab37a039b39daf` |
| CompHome assembly metadata | `ev_9c1e9fe813ef6ae63f71447d3410e462fa8abaec867bfa646ea4d30d3f74aa7b` |
| CompTerra managed members, retained parent | `ev_2f96f109a38ed2a1692d195cf2bfcda04512a9d07980e66df7b47bbeac9808bc` |
| CompTerra selected configuration/device/tab types | `ev_85c7d9e67ee4f352481a359a7d016fc52d21ad6f776d4ae0e14d7f8f143da16f` |
| CompTerra CommonDefines | `ev_c78cba38ce3e773af4463fc789d16612479363c329230e0dd3466b652ac60a25` |

These identifiers refer to the original analysis session; they are not public
download links or a portable evidence bundle. The table preserves provenance
without redistributing the target artifacts. CompTerra analysis reported one
user-string terminal-flag discrepancy; readable UTF-16 was preserved. Static
metadata and CIL observations do not establish runtime behavior by themselves.

CompHome's assembly-reference inventory did not name a ContentsBrowser assembly.
That bounded observation informed the experiment; it does not prove the absence
of dynamic dependencies.

Hashes of `/etc/passwd`, `/usr/share/sdbd/public.pem`, and `/etc/hosts` were
unchanged across the first cleanup pass. This was a check of those three files,
not a claim that every TV file was unchanged.

## Restoring the earlier cleanup

Use the TV command transport described in [README.md](README.md). These commands
are **not host shell commands**. Restore only packages/apps that your own recorded
baseline showed enabled. The list below matches the tested session's baseline.

Enable the six disabled packages first:

```text
pkgcmd -A -t tpk -n com.samsung.tv.targeted-ad-service --global
pkgcmd -A -t tpk -n com.samsung.tv.acr-service-app --global
pkgcmd -A -t tpk -n com.samsung.tv.iacr --global
pkgcmd -A -t tpk -n com.samsung.tv.tvplus --global
pkgcmd -A -t tpk -n com.samsung.tv.tvPlusDeeplinker --global
pkgcmd -A -t wgt -n adplatform --global
```

Then enable the seven applications, substituting the discovered owner UID:

```text
pkgcmd -A -t app -n com.samsung.tv.tvPlusDeeplinker --uid 5001
pkgcmd -A -t app -n com.samsung.tv.tvplus --uid 5001
pkgcmd -A -t app -n adplatform.ADPlayer --uid 5001
pkgcmd -A -t app -n com.samsung.tv.targeted-ad-service --uid 5001
pkgcmd -A -t app -n com.samsung.tv.acr --uid 5001
pkgcmd -A -t app -n com.samsung.tv.iacr --uid 5001
pkgcmd -A -t app -n com.samsung.tv.adagent-service --uid 5001
```

The recommendation component has its own additional restore command:

```text
pkgcmd -A -t app -n com.samsung.tv.ContentsBrowser --uid 5001
```

Read back `package_info.package_disable` and `package_app_info.app_disable` for
each target. Both should be `false` where enabled was the original state. Reload
Home as described in the README and check the screen. A complete all-target
rollback was prepared but not executed; only the targeted-ad package round trip
and TV Plus launcher app round trip were experimentally verified.
