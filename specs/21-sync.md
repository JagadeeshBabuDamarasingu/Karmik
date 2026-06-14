# Karmik — Sync & Multi-Device

## Overview

Karmik is local-first: all data lives on the device. But users have multiple devices (phone +
laptop, work Mac + personal Android) and expect their agents, memories, and conversations to
follow them. This spec defines three sync strategies, from most privacy-safe to most convenient,
that users can choose between.

**Spec 11 ("no cloud sync") applied to Stage 1 only.** Stage 2 introduces opt-in sync
with no change to the local-first default.

---

## Sync Strategies

### Strategy 1 — Local Network Sync (Default When Enabled)

Peer-to-peer sync over the local LAN. No cloud, no account, no third party sees any data.

**Discovery**: devices find each other via mDNS (`_karmik._tcp.local`). Each Karmik instance
advertises its hostname, port (default 5174), and device ID.

**Transport**: TLS 1.3 (self-signed certificate per device, pinned on trust establishment).

**Pairing**: user scans a QR code or enters a 6-digit PIN displayed on the other device.
Pairing uses a Diffie-Hellman key exchange to derive a shared secret. The shared secret
is stored in the platform keystore on both devices. A trusted device list is maintained
(`trusted_devices` table in `karmik.db`).

**Sync protocol**: on connection, devices exchange a vector clock summary. The receiving
device requests only the changes it is missing (delta sync — not full DB transfer).

**When**: sync runs when both devices are on the same LAN and at least one is awake.
Triggered by: app foreground, connection to known SSID, manual "Sync now" button.

**Platform**: Android, macOS, Windows, Linux. iOS: can receive but cannot advertise
(Bonjour restriction in background); iOS syncs when app is open.

### Strategy 2 — Encrypted Cloud Sync (Opt-In)

Stores an E2E-encrypted sync blob in the user's own cloud storage account. Karmik never
has access to the plaintext — the cloud provider only sees opaque encrypted bytes.

**Storage backends supported**:

| Provider | Implementation |
|---|---|
| iCloud Drive | CloudKit / `NSFileCoordinator` on Apple platforms |
| Google Drive | Google Drive API v3 (OAuth2, stored in platform keystore) |
| Dropbox | Dropbox API v2 (OAuth2) |
| WebDAV | User provides URL + credentials (self-hosted Nextcloud, ownCloud, etc.) |

**Encryption**: AES-256-GCM. Encryption key derived via HKDF from a device-local "sync
passphrase" (user-chosen 20+ character passphrase, or auto-generated and displayed once).
The passphrase is never uploaded — without it, the cloud blob is indecipherable.

**Sync object**: a single file `karmik_sync.enc` in the app's cloud storage folder.
Structure: encrypted JSON containing the sync delta (not the full DB). On first sync from
a new device, the user enters their sync passphrase to decrypt and restore.

**Conflict resolution**: see [Conflict Resolution](#conflict-resolution) below.

### Strategy 3 — Self-Hosted Karmik Sync Server (Power Users)

Users who run a home server or NAS can self-host the Karmik Sync Server — a lightweight
Go binary or Docker image.

**Docker**:
```bash
docker run -d \
  -p 5175:5175 \
  -v /data/karmik-sync:/data \
  -e KARMIK_AUTH_TOKEN=your-secret-token \
  ghcr.io/karmik/sync-server:latest
```

**Protocol**: same as local network sync (delta protocol over TLS), but via internet.
Users configure the server URL in Settings → Sync → "Self-hosted server".

**Authentication**: Bearer token (user configures). Traffic is TLS-encrypted. The server
stores only encrypted blobs — it never has the decryption key.

**Benefits over cloud sync**: no dependency on third-party storage providers; full data
ownership; no file size limits; can serve many devices efficiently with delta protocol.

---

## Sync Scope

Not all data syncs by default. Users can opt each category in or out.

| Data category | Default sync | Notes |
|---|---|---|
| Agent configurations | ✓ On | Names, system prompts, pinned context, tool permissions |
| Long-term memories | ✓ On | Vector embeddings are NOT synced — regenerated on destination |
| Conversation history | ✗ Off (opt-in) | Large data; users may prefer per-device conversations |
| Task list | ✓ On | Synced as structured rows |
| Code snippets | ✓ On | |
| Reading list | ✓ On | Cached article text NOT synced (too large; re-fetched on demand) |
| Notes (Markdown files) | ✗ Off | Managed by notes backend (Obsidian/iCloud Notes handles its own sync) |
| Model downloads | ✗ Never | Models are large; re-downloaded per device |
| Plugin code | ✓ On | Plugin manifests and scripts |
| Settings/preferences | ✓ On | Per-device overrides (e.g., notification settings) are excluded |
| Audit log | ✗ Never | Audit log is device-local only |
| Sync passphrase / device keys | ✗ Never | Never leaves the device |

---

## Conflict Resolution

Different data types use different strategies:

| Type | Strategy | Rationale |
|---|---|---|
| Agent config | Last-write-wins (by `updatedAt` timestamp) | Config edits are infrequent; LWW is sufficient |
| Memories | Append-only merge (union by `id`) | Memories are never updated — only created and deleted |
| Memory deletions | Tombstone (soft delete, synced as `deleted_at`) | Prevents deleted memories re-appearing after sync |
| Tasks | Last-write-wins per task ID | Task status changes are authoritative from the editing device |
| Conversation messages | Append-only merge by `id` | Messages don't change after creation |
| Settings | Last-write-wins per key | Simple key-value |

**Clock skew**: device clocks can drift. Timestamps are recorded in UTC. For LWW, a 5-second
tolerance window avoids false conflicts from minor clock differences. For true simultaneous
edits, the device with the lexicographically larger device ID "wins" (deterministic tiebreak).

---

## Device Management

Each Karmik install has a unique **device ID** (UUID v4, generated at first launch, stored
in platform keystore).

```dart
class SyncDevice {
  final String id;               // UUID
  final String name;             // "Pixel 8 Pro", "MacBook Air"
  final String platform;         // android | ios | macos | windows | linux
  final DateTime pairedAt;
  final DateTime lastSeenAt;
  final bool isTrusted;
}
```

**Trusting a new device**:
1. On Device A: Settings → Sync → "Add device" → displays QR code (contains device ID + ECDH public key)
2. On Device B: scan QR → confirm device name → "Trust"
3. ECDH key exchange completes → shared secret derived → stored in keystore on both devices
4. Initial sync begins (full sync on first connect; delta thereafter)

**Revoking a device**: Settings → Sync → Trusted Devices → "Remove". The removed device
loses sync access. Its local data is not deleted — it just stops receiving updates.

**Max trusted devices**: 10 (reasonable for personal use; enforced by the sync server and
local enforcement on LAN sync).

---

## Sync Status UI (Settings → Sync)

```
Sync
══════════════════════════════════════════════

Strategy
  ○ Off
  ● Local Network (LAN)      [active — 2 devices]
  ○ Cloud Storage            [iCloud Drive ▾]
  ○ Self-Hosted Server       [https://nas.home:5175]

Last sync: 2 minutes ago
[Sync now]

Sync scope
  Agent configurations       [✓]
  Long-term memories         [✓]
  Conversation history       [✗]
  Tasks                      [✓]
  Code snippets              [✓]
  Reading list               [✓]
  Plugins                    [✓]
  Settings                   [✓]

Trusted Devices
  ─────────────────────────────────────
  Pixel 8 Pro (Android)      Last seen: just now   [Remove]
  MacBook Air (macOS)        Last seen: 1 hour ago [Remove]
  ─────────────────────────────────────
  [+ Add device]

Encryption
  Sync passphrase: ••••••••••••••••    [Change] [Show]
  Encryption: AES-256-GCM + HKDF      ✓ All data encrypted before leaving device

══════════════════════════════════════════════
```

---

## Privacy Mode

| Sync strategy | Privacy Mode behavior |
|---|---|
| Local Network Sync | Allowed — LAN traffic only; TLS encrypted |
| Cloud Sync (iCloud/Google/Dropbox) | Blocked — sends data to third-party internet servers |
| Cloud Sync (WebDAV on LAN IP) | Allowed in LAN-only mode |
| Self-Hosted Server (LAN IP) | Allowed |
| Self-Hosted Server (public URL) | Blocked |

The sync passphrase itself never leaves the device in any mode.

---

## Vector Embedding Regeneration

Vector embeddings (for memories, snippets, docs) are **not synced** — they are large (384
floats × thousands of entries) and platform-specific (quantization may differ). Instead:

1. The raw content (memory text, snippet code, etc.) is synced.
2. On the destination device, embeddings are regenerated lazily — on first semantic search,
   Karmik re-embeds any records that have no local embedding.
3. A background job (`karmik.reembed`) runs after each sync to pre-generate embeddings
   while the device is charging, so searches don't feel slow.

---

## Platform Availability

| Feature | Android | iOS | macOS | Windows | Linux | Web |
|---|---|---|---|---|---|---|
| LAN sync (advertise) | ✓ | ~ (foreground only) | ✓ | ✓ | ✓ | — |
| LAN sync (receive) | ✓ | ✓ | ✓ | ✓ | ✓ | — |
| Cloud sync | ✓ | ✓ | ✓ | ✓ | ✓ | — |
| Self-hosted server sync | ✓ | ✓ | ✓ | ✓ | ✓ | — |
| QR code pairing (display) | ✓ | ✓ | ✓ | ✓ | ✓ | — |
| QR code pairing (scan) | ✓ | ✓ | — (use PIN) | — (use PIN) | — (use PIN) | — |

**Web**: sync is not available on Web (no persistent local storage to sync; Web is a
read-only remote-inference session by design).
