# RISEFX fork of `zulip/zulip`

This is the **application source** for RISEFX's self-hosted Zulip. It is a thin
fork: upstream's stable branch plus two patches. It is never deployed directly —
it is consumed as a build input by
[`RISEFX/docker-zulip`](https://github.com/RISEFX/docker-zulip), which bakes it
into `ghcr.io/risefx/zulip-server`.

For upstream's own documentation, see [`README.md`](README.md).

## Where this sits in the release pipeline

```
RISEFX/zulip           @ 12.1-risefx1  ─┐
                                        ├─► ghcr.io/risefx/zulip-server:12.1-0-risefx1
zulip/docker-zulip     @ 12.1-0        ─┘   (built by RISEFX/docker-zulip)
  (unmodified upstream recipe)
```

A tag pushed here does **not** build anything. It only makes a reviewed,
immutable source ref that `RISEFX/docker-zulip` can pin. Tag here first, then tag
there.

## Branches

| Branch        | Purpose                                                       |
| ------------- | ------------------------------------------------------------- |
| `12.x-risefx` | **The fork.** Upstream `12.x` + the two patches below.         |
| `main`        | Mirror of upstream `main`. Not used for releases.              |

`12.x-risefx` is currently based on upstream `12.x` at `152fe8c74c` (`version:
Update version after 12.1 release.`) with zero drift — the only commits it adds
are the two patches.

The branch is renamed on every major version: when upstream cuts `13.x`, the
patches move to a new `13.x-risefx` branch. See [Updating](#updating-to-a-new-upstream-release).

## Tags

Format: `<Z>-risefx<P>`

- `Z` — the upstream Zulip version, e.g. `12.1`. Must match `LATEST_RELEASE_VERSION` in [`version.py`](version.py).
- `P` — RISEFX patch revision, starting at `1`. Bump when the patches change but `Z` does not.

Example: `12.1-risefx1`.

The trailing digit is **mandatory** — `RISEFX/docker-zulip`'s build workflow
matches `^([0-9]+\.[0-9]+)-([0-9]+)-risefx([0-9]+)$` and will not resolve a tag
without it.

Existing tags: `12.1-risefx1`, and `12.1-risefx` (see [Known issues](#known-issues)).

## What this fork changes

Two functional commits on top of upstream (plus this README). Both are small and
both are intended to survive rebases untouched.

### 1. `riselink` URL scheme — `f32285f861`

[`zerver/lib/markdown/__init__.py`](zerver/lib/markdown/__init__.py)

Adds `riselink` to `html_safelisted_schemes`, so `riselink:` URLs written in
messages survive Markdown rendering and sanitization instead of being stripped.

It is added to `html_safelisted_schemes` only, **not** to `auto_linked_schemes`.
`html_safelisted_schemes` is splatted into `allowed_schemes`, which gates
`sanitize_url()` (`__init__.py:1438`), so an explicit Markdown link
`[label](riselink:...)` survives. `auto_linked_schemes` is what builds the
bare-URL autolink regex (`__init__.py:239`), so a bare `riselink:...` typed into a
message is **not** turned into a link. Links must be written in `[label](...)`
form.

This is only half the feature. For a user to actually *open* one of these links,
the desktop client must also be told to hand `riselink:` to the OS — see
[`RISEFX/zulip-desktop`](https://github.com/RISEFX/zulip-desktop) and its
`whitelistedProtocols` setting. A `riselink:` URL rendered by this patch but
opened in a client without that setting is bounced through an interstitial HTML
page rather than launched.

### 2. Restrict `/user_uploads` to internal networks — `dc8a37e9d0`

[`puppet/zulip/files/nginx/zulip-include-frontend/app`](puppet/zulip/files/nginx/zulip-include-frontend/app)

Wraps `location /user_uploads` in an nginx `allow`/`deny` list, so uploaded files
are only served to loopback and RISEFX internal ranges:

```nginx
location /user_uploads {
# restrict-uploads-10net
        allow 127.0.0.1;
        allow 10.10.8.0/21;
        allow 10.10.250.0/24;
        allow 10.20.8.0/21;
        allow 10.30.8.0/21;
        allow 10.40.8.0/21;
        allow 10.50.8.0/21;
        deny  all;
    ...
}
```

The `# restrict-uploads-10net` marker exists so the block is greppable after a
rebase:

```bash
git grep -n restrict-uploads-10net -- puppet/
```

Because this is baked into the image at build time, changing the allowed ranges
means a new `-risefx<P>` tag and a new image. It is not runtime-configurable.

> **Note:** nginx `allow`/`deny` evaluates the socket peer address. If Zulip runs
> behind a reverse proxy or load balancer, every request will appear to come from
> the proxy and this list must contain the proxy's address, not the client's.
> Verify against your actual deployment topology.

## Updating to a new upstream release

### One-time setup

This clone has only `origin` (`RISEFX/zulip`). Add upstream:

```bash
cd ~/workspace/repos/risefx/zulip
git remote add upstream https://github.com/zulip/zulip.git
git fetch upstream --tags
```

### Case A — patch release on the same major (e.g. 12.1 → 12.2)

Upstream pushes it to the existing `12.x` branch.

```bash
git fetch upstream --tags
git switch 12.x-risefx
git rebase upstream/12.x
```

### Case B — new major release (e.g. 12.x → 13.0)

Upstream cuts a new `13.x` branch. Move the patches onto it under a new branch
name.

```bash
git fetch upstream --tags

OLD_BASE=$(git merge-base 12.x-risefx upstream/12.x)
git switch -c 13.x-risefx 12.x-risefx
git rebase --onto upstream/13.x "$OLD_BASE"
```

### Verify the rebase (both cases)

Confirm exactly the two patches sit on top, and that they came through unchanged:

```bash
# Expect exactly 3 commits: the two patches, plus this README.
git log --oneline upstream/13.x..13.x-risefx

# Expect three "=" rows, meaning everything applied with no textual change.
git range-diff upstream/12.x...12.x-risefx upstream/13.x...13.x-risefx
```

If `range-diff` shows a `!` row, upstream moved code under one of the patches.
Read that diff before continuing — in particular, check the patches still land
where they are supposed to:

```bash
git grep -n riselink -- zerver/lib/markdown/__init__.py
git grep -n restrict-uploads-10net -- puppet/zulip/files/nginx/zulip-include-frontend/app
```

Upstream reorganizes `html_safelisted_schemes` and the nginx config
occasionally; a clean rebase does not prove the patch is still doing its job.

### Tag and push

`Z` must equal `LATEST_RELEASE_VERSION` in `version.py` on the rebased branch:

```bash
grep LATEST_RELEASE_VERSION version.py

git tag 13.0-risefx1
git push origin 13.x-risefx
git push origin 13.0-risefx1
```

Then continue in [`RISEFX/docker-zulip`](https://github.com/RISEFX/docker-zulip)
to build the image — see that repo's `README.risefx.md`.

### Re-spinning patches without an upstream bump

If only the patches change (say, a new internal subnet) at the same upstream
version, bump `P` and leave `Z` alone:

```bash
git tag 13.0-risefx2
git push origin 13.x-risefx 13.0-risefx2
```

## Known issues

These are recorded, not fixed. Decide on each before the next release.

- **`/thumbnail` is not restricted.** The allowlist covers only `location
  /user_uploads`. Zulip serves image previews of uploaded files from `location
  /thumbnail` in the same nginx config, which still reaches uwsgi from any
  address. For a change titled *"block images from remote access"* this looks
  like a gap. `location /avatar` is likewise unrestricted, though that may well
  be intentional.
- **No IPv6 loopback.** The block allows `127.0.0.1` but not `::1`. The
  `location /api/internal/` block a few lines below allows both. If nginx accepts
  on `::1`, local health checks and the like will be denied.
- **`12.1-risefx` is a dead tag.** It points at the same commit as
  `12.1-risefx1`, but the docker build workflow's regex requires a trailing
  digit, so it can never be referenced. Consider deleting it to avoid confusion.
- **Whitespace.** Two of the `allow` lines in the nginx block are indented with a
  tab, the rest with spaces. Harmless to nginx, but it will show up in every
  `git show` of this patch forever.
