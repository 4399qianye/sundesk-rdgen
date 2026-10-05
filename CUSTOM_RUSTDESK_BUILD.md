# Custom RustDesk Build Repositories

Use two repositories for this setup:

- `sundesk-rustdesk`: a fork of RustDesk containing the `low-latency-video` changes.
- `sundesk-rdgen`: a fork of this repository containing the generator workflows.

The names are recommendations. The workflows accept any repository name through Actions
variables, so an owner/repository pair such as `your-account/sundesk-rustdesk` is valid.

## Build Repository Configuration

Configure these in the `sundesk-rdgen` repository settings under **Actions > Variables**:

```text
RUSTDESK_REPOSITORY=your-account/sundesk-rustdesk
RUSTDESK_REF=low-latency
LOW_LATENCY_VIDEO=true
```

For a private source repository, add this under **Actions > Secrets**:

```text
RUSTDESK_REPO_TOKEN=<token with Contents: Read on sundesk-rustdesk>
```

The Windows workflows build the x64 virtual HID package (`rustdesk_hid.sys`,
`rustdesk_hid.inf`, and `rustdesk_hid.cat`) with the Windows Driver Kit before
running `build.py`. The package is staged into the Windows release directory,
included in the portable client and MSI, and signed through `SIGN_BASE_URL` and
`SIGN_API_KEY` when those secrets are configured. The installer registers the
package with `pnputil` and starts the `RustDeskHid` service.
The x86 Windows workflow does not include this x64 driver.

Without a code-signing service, the generated client is suitable for testing
only. A normal Windows installation requires a Microsoft-trusted signature for
the kernel driver; signing the EXE or MSI alone is not sufficient.

The existing `GHUSER`, `REPONAME`, `GHBRANCH`, and `GHBEARER` settings in the Django service
continue to point at `sundesk-rdgen`, because that repository owns the `workflow_dispatch`
files. The source repository only needs to preserve the RustDesk directory layout and its
submodules.

## Transport Scope

`low-latency-video` keeps RustDesk's ID/password authentication, rendezvous discovery, UDP/TCP
hole punching, WebRTC race, relay fallback, and input/control protocol. It adds a separate encoded
video plane:

- Direct KCP connections use the already-punched UDP socket and encrypted `RDMV` datagrams.
- Datagrams contain a protocol version, frame id, fragment id, fragment count, random nonce, and
  `secretbox` payload. Incomplete frames are discarded instead of blocking control traffic.
- Relay/WebRTC connections use the `MediaFrame` envelope over the authenticated control stream;
  the old `VideoFrame` path remains available for clients that do not advertise the new capability.
- Older clients do not request `low_latency_video`, so the server keeps the original `VideoFrame`
  path for them.
- The encoder uses periodic keyframes in the low-latency profile so a dropped frame sequence can
  recover without restarting the RustDesk session.

This is a RustDesk-compatible media replacement, not a Moonlight-compatible Sunshine RTSP/RTP
implementation. Moonlight clients cannot connect to it directly because the ID/password session,
identity handshake, control messages, and media negotiation remain RustDesk protocol contracts.

## VIIPER Game Input

The Windows artifact also carries `viiper.exe`. Install the signed `usbip-win2`
package on the target Windows machine and launch the client normally; the
standard HID mouse/keyboard backend is enabled by default.

```text
RUSTDESK_VIIPER=0
```

VIIPER is started locally by RustDesk and controlled through its localhost TCP
API. Set `RUSTDESK_VIIPER=0` to force the old fallback. `usbip-win2` is a
separate prerequisite because it provides the signed generic USB/IP Windows
driver.
