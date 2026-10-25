# gdirect

[中文](README.zh-CN.md) · [Changelog](CHANGELOG.md)

gdirect browses, downloads, uploads and mounts your Google Drive from the terminal. It finds Google IPs that are reachable from your network and sends Drive traffic straight to them, with multi-connection, resumable downloads.

## How it works

| Traffic | Route |
| --- | --- |
| Drive API, downloads, uploads (`browse`, `ls`, `download`, `upload`) | A scanned Google IP. The hostname and certificate check stay the same; the TLS SNI can be changed with `--sni` / `--no-sni` |
| Sign-in, account verification and token refresh (`accounts.google.com`, `oauth2.googleapis.com`, `www.googleapis.com`) | Your normal system network (DNS, proxy) |
| `mount` (rclone) | Your normal system network |

The first transfer command scans candidate IPs automatically and keeps the fastest working one for 24 hours. `login` and `mount` do not scan.

Network requirements:

- Sign-in needs browser access to `accounts.google.com` and application access to `oauth2.googleapis.com` and `www.googleapis.com`. Token refresh needs `oauth2.googleapis.com`. If Google is blocked, keep a proxy available for these requests, including during transfers.
- If you use a TUN-mode proxy or VPN (for example Clash/Mihomo TUN), route the scanned Google IPs **DIRECT** and turn off fake-ip for them. Otherwise gdirect's traffic is captured again and the scan finds nothing. `gdirect scan --list` shows the IPs.

### System proxy without TUN

On macOS and Windows, sign-in, account verification and token refresh automatically read manual HTTP/HTTPS system proxy settings. You can leave your proxy client in rule mode with system proxy enabled and TUN disabled. IP scans, pinned Drive API requests, uploads and libaria2 downloads bypass application proxies and connect to the scanned IP directly.

For Clash/Mihomo, place these rules before any conflicting DIRECT rules, replacing `PROXY` with your proxy group name. Keep your existing Google rules for the browser's other sign-in resources:

```yaml
rules:
  - DOMAIN,accounts.google.com,PROXY
  - DOMAIN,oauth2.googleapis.com,PROXY
  - DOMAIN,www.googleapis.com,PROXY
  # Your existing rules follow.
```

The `www.googleapis.com` rule lets account verification use the proxy. Pinned transfers to the same hostname bypass the proxy client, so this rule does not send those transfers through your proxy. This holds only with TUN disabled: if TUN is on with domain sniffing, the sniffed SNI matches this rule and transfers to the scanned IPs are proxied. With TUN on, omit this rule or exclude the scanned IPs from sniffing. With TUN disabled, you do not need a `PROCESS-NAME,gdirect,DIRECT` rule or global DIRECT rules for Drive domains. The browser must bypass the proxy for `127.0.0.1` so the local OAuth callback can complete.

On Linux, or when using PAC or a proxy client that does not set manual system proxy settings, supply the HTTP proxy address for each command. The port below is an example; use your client's HTTP or mixed port:

```sh
HTTPS_PROXY=http://127.0.0.1:7890 gdirect login --client-secret ~/Downloads/client_secret.json
HTTPS_PROXY=http://127.0.0.1:7890 gdirect browse
```

`HTTP_PROXY` / `HTTPS_PROXY` (or their lowercase forms) take precedence over the corresponding system settings. Make sure `NO_PROXY` / `no_proxy` does not exclude the Google endpoints above. Pinned transfers still bypass these environment proxies.

`mount` uses a separate rclone process. Pass `HTTPS_PROXY` when starting a mount so rclone also uses your proxy without TUN. Mounted file traffic follows rclone's network path; it does not use gdirect's scanned IP routing:

```sh
HTTPS_PROXY=http://127.0.0.1:7890 gdirect mount
```

## Install

Prebuilt for macOS 15+ (Apple Silicon / Intel), Linux x86_64 with glibc 2.35+ (Ubuntu 22.04+, Debian 12+, recent Fedora/Arch/openSUSE), and Windows x86_64. Alpine/musl, older glibc and Linux ARM64 need a [source build](#build-from-source).

macOS / Linux (requires `curl`):

```sh
curl -fsSL https://raw.githubusercontent.com/mizorewww/gdirect/main/scripts/install.sh | bash
```

Windows PowerShell:

```powershell
& ([scriptblock]::Create((Invoke-RestMethod 'https://raw.githubusercontent.com/mizorewww/gdirect/main/scripts/install.ps1')))
```

The installer puts gdirect in `~/.local/bin` (Windows: `%LOCALAPPDATA%\gdirect\bin`) and tells you how to add it to PATH. On Windows, open a new terminal after installing.

To use `mount`, add `--with-mount` (PowerShell: `-WithMount`). This also installs rclone 1.75.2+ and, on Linux, FUSE3 through your package manager (uses sudo). On Windows it installs rclone and WinFsp with winget; WinFsp may ask for administrator approval.

Homebrew (macOS / Linux):

```sh
brew tap mizorewww/gdirect https://github.com/mizorewww/gdirect
brew install mizorewww/gdirect/gdirect
brew install rclone   # only for mount; Linux also needs your distro's fuse3 package
```

Upgrade with `brew update && brew upgrade gdirect`, or by rerunning the installer.

You can also download a [release archive](https://github.com/mizorewww/gdirect/releases/latest). Keep `cacert.pem` next to the executable, because downloads use it to verify certificates.

## One-time Google setup (about 10 minutes)

gdirect does not ship a shared Google client, so you create your own in a Google Cloud project, once. Follow the [Google setup guide](docs/configuration/README.md) to download a **Desktop app** OAuth client JSON, then:

```sh
gdirect login --client-secret ~/Downloads/client_secret.json
```

A browser opens. After you approve, the terminal prints `Logged in`. The JSON file is not needed again.

## Daily use

```sh
gdirect                                   # same as gdirect browse
gdirect browse -o ~/Downloads             # download directory (default: ./downloads)
gdirect ls root --json
gdirect download FILE_ID_OR_URL           # saves to ./downloads/<name>
gdirect download FILE_ID -o file.zip
gdirect download DOC_ID --export application/pdf -o doc.pdf
gdirect upload ./file.zip
gdirect upload ./file.zip --parent FOLDER_ID --name backup.zip
```

| Key in `browse` | Action |
| --- | --- |
| `j/k`, `↑/↓` | Select |
| Enter, `l/→` | Open folder |
| `h/←`, Backspace | Parent folder |
| `d` | Queue download. Google Docs/Sheets/Slides are exported as PDF |
| `+/-` | Connections for the next download (1–16), saved as your default |
| `r` | Refresh |
| `c` | Cancel the current download and keep the partial data |
| `q` | Quit |

- **Resume:** if a download or upload is interrupted, run the same command again.
- **Downloads** that already finished are recognised by their checksum and reported as already downloaded. Files are checked against Drive's MD5 after the download.
- **Uploads** always create a new file. They never overwrite a file with the same name.
- **Connections:** `-j N` sets the number of connections for one command only. The saved default (8 at first) changes only with `+/-` in `browse`.

Advanced transfer options for `download` and `browse` (`ls` and `upload` accept `--ip` only):

| Option | Meaning |
| --- | --- |
| `--ip IP` | Use this Google IP instead of the scanned one |
| `--sni HOST` / `--no-sni` | Send a different TLS SNI, or none. Certificates are still checked against the real hostname |
| `--ca-cert FILE` | Use this CA bundle instead of the bundled `cacert.pem` |
| `--export MIME` | (`download`) Export a Google Docs file, for example `application/pdf` |

Run `gdirect scan` to scan again and `gdirect scan --list` to see the results, including the last error for each IP. To test your own candidates, run `gdirect scan --ips FILE`. The file holds one public IP per line, and `#` starts a comment.

For scripts, `GDIRECT_ACCESS_TOKEN` can supply an access token instead of a login (`mount` still needs `gdirect login`).

## Mount

Mounting requires the dependencies from `--with-mount` or the Homebrew step above.

```sh
gdirect mount                        # ~/GoogleDrive
gdirect mount ~/MyDrive --read-only
gdirect unmount
```

- On macOS the mount uses the built-in NFS client, so no FUSE driver is needed. Linux uses FUSE3.
- On Windows the mount runs in the foreground and stops with Ctrl-C.
- The mount directory must be empty on macOS/Linux. On Windows it must not exist yet.
- Writes go to a local cache first and upload in the background. The cache tries to stay under 5 GiB, but open or unsent files can exceed that. Uploads are finished when the log printed by `mount` shows `to upload 0, uploading 0`.
- `unmount` keeps files that have not been uploaded yet, and the next `mount` uploads them.
- `-j` sets how many rclone transfers run at once.

## Your data and logout

Credentials, scanned IPs and settings are stored in one private SQLite file:

- macOS: `~/Library/Application Support/gdirect/state.sqlite3`
- Linux: `~/.local/share/gdirect/state.sqlite3`
- Windows: `%LOCALAPPDATA%\gdirect\state.sqlite3`

Use `--db PATH` to choose another file. `gdirect logout` removes the stored credentials and keeps the scanned IPs. Unmount before logging out. To switch Google accounts, run `gdirect login --reauth`.

Sign-in problems, the 7-day expiry and other questions are covered in [Troubleshooting](docs/configuration/README.md#troubleshooting).

## Build from source

Requires Rust 1.91+, a C++ compiler, make, curl and OpenSSL 3.

```sh
git clone https://github.com/mizorewww/gdirect.git
cd gdirect
# macOS: brew install openssl@3
# Debian / Ubuntu: sudo apt-get install build-essential pkg-config libssl-dev ca-certificates curl xz-utils
./scripts/build-libaria2.sh
cargo install --path . --locked
```

Windows builds use the MinGW GNU toolchain. In an MSYS2 MINGW64 shell, install `make patch tar xz mingw-w64-x86_64-gcc mingw-w64-x86_64-openssl mingw-w64-x86_64-pkgconf`, run `rustup target add x86_64-pc-windows-gnu`, set `OPENSSL_PREFIX=$(cygpath -m /mingw64)`, then run the two commands above with `--target x86_64-pc-windows-gnu` added to `cargo install`.
