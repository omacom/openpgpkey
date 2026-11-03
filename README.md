# openpgpkey.omarchy.org

The [Web Key Directory](https://datatracker.ietf.org/doc/draft-koch-openpgp-webkey-service/) (WKD) for omarchy.org. Any OpenPGP client can fetch Omarchy's key from the address `pkgs@omarchy.org`, over HTTPS, with no keyserver.

That key signs the Omarchy packages and ISOs:

```
40DF B630 FF42 BCFF B047  046C F013 4EE6 80CA C571
Omarchy <pkgs@omarchy.org>
```

`omarchy.gpg` is the key. It is the same file that the `omarchy-keyring` package installs, and the [Omarchy manual](https://omarchy.org/manual/security/) lists the same fingerprint.

## Get the key

```bash
gpg --locate-external-keys pkgs@omarchy.org
```

GnuPG looks the key up through WKD by default. Check that the fingerprint gpg prints is the one above.

## Update the key

1. Copy the new `omarchy.gpg` from `pkgbuilds/omarchy-keyring/` in [omacom/omarchy-pkgs](https://github.com/omacom/omarchy-pkgs) over the one here.
2. Write the WKD files from it with GnuPG's own tool:

   ```bash
   gpg-wks-client -C public/.well-known/openpgpkey --install-key ./omarchy.gpg pkgs@omarchy.org
   ```

   It keeps only the `pkgs@omarchy.org` User ID and writes the key under its WKD hash. On Arch, the tool is `/usr/lib/gnupg/gpg-wks-client`.
3. Open a pull request with both changed files. Merging it deploys them.

## Check

```bash
gpg-wks-client --check pkgs@omarchy.org
```

It returns success when the key is available through WKD.

To see how the key was found, look it up from an empty keyring:

```bash
gpg --homedir "$(mktemp -d)" --verbose --locate-keys pkgs@omarchy.org
```

It prints `automatically retrieved 'pkgs@omarchy.org' via WKD`. A temporary keyring rules out a copy you already have, and it also rules out a cached failure in your own `dirmngr`.

[wkd.dp42.dev](https://wkd.dp42.dev/) runs the same checks from outside and shows each step.

## What is served

WKD has two lookup methods, and both serve the same file. The specification says a client tries the advanced method on the `openpgpkey` host first. It uses the direct method on `omarchy.org` itself only when that host does not exist in DNS.

| Method | URL |
|---|---|
| Advanced, the key | `https://openpgpkey.omarchy.org/.well-known/openpgpkey/omarchy.org/hu/scsaohmoz4qo7bwupkm398ssojww4ioz` |
| Advanced, the policy file | `https://openpgpkey.omarchy.org/.well-known/openpgpkey/omarchy.org/policy` |
| Direct, the key | `https://omarchy.org/.well-known/openpgpkey/hu/scsaohmoz4qo7bwupkm398ssojww4ioz` |
| Direct, the policy file | `https://omarchy.org/.well-known/openpgpkey/policy` |

`scsaohmoz4qo7bwupkm398ssojww4ioz` is the WKD hash of `pkgs`. The direct paths in `public/` are symbolic links to the advanced ones. WKD requires the policy file, which holds one comment line.

`public/_headers` serves the key as `application/octet-stream`, and lets browser clients read every file.

`public/_redirects` sends a visit to `https://openpgpkey.omarchy.org/` to this repository with a 302. It matches `/` only, so no WKD path changes.

No client or proxy keeps a copy of these files: `public/_headers` sends `Cache-Control: no-store`, and `wrangler.jsonc` turns Workers Cache off. Cloudflare serves the files of the active deployment from its asset store, so a merged key change is live once its deploy finishes.

## How it runs

The host is a Cloudflare Worker with static assets and no script, configured in `wrangler.jsonc`. Cloudflare serves the files from the location nearest the client.

- `openpgpkey.omarchy.org` is the Worker's custom domain, which gives the host its DNS record and certificate.
- `omarchy.org/.well-known/openpgpkey/*` is a Worker route. A route takes precedence over the website's custom domain, for that path only.

Cloudflare Workers Builds deploys `main`: each merge runs `npx wrangler deploy` inside Cloudflare, so GitHub stores no deploy credential.

| Workers Builds setting | Value |
|---|---|
| Repository | this one; the Cloudflare GitHub App gets access to it only |
| Production branch | `main` |
| Deploy command | `npx wrangler deploy` |
| Builds for other branches | off |

To deploy without GitHub, run `npm ci && npx wrangler deploy` from a checkout of `main`.

## CI

`.github/workflows/check.yml` has two jobs:

- `files`, on every pull request and push: `omarchy.gpg` is still `40DFB630…`, and running `gpg-wks-client` again changes no committed file.
- `live`, after each push to `main` and once a day: both methods serve this commit's key and policy file with `Cache-Control: no-store`, and `gpg-wks-client --check` passes.

## Undo

```bash
npx wrangler delete
```

This removes the Worker, its DNS record and its route on `omarchy.org`. No WKD lookup then finds the key.
