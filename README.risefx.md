# RISEFX fork of `zulip/zulip`

This is the **application source** for RISEFX's self-hosted Zulip at
`chat.risefx.com`. It is a thin fork: an upstream stable branch plus two
patches and this README. It is never deployed directly. Instead,
[`RISEFX/docker-zulip`](https://github.com/RISEFX/docker-zulip) builds it into
the image `ghcr.io/risefx/zulip-server`.

For upstream's own documentation, see [`README.md`](README.md).

## Where this sits in the release pipeline

```
RISEFX/zulip           @ 13.3-risefx1  ─┐
                                        ├─► ghcr.io/risefx/zulip-server:13.3-0-risefx1
zulip/docker-zulip     @ 13.3-0        ─┘   (built by RISEFX/docker-zulip)
  (unmodified upstream recipe)
```

A tag pushed here does **not** build anything. It only creates a reviewed,
immutable source ref that `RISEFX/docker-zulip` can pin. Tag here first, and
then tag there.

The desktop client fork,
[`RISEFX/zulip-desktop`](https://github.com/RISEFX/zulip-desktop), is released
independently.

## Branches

| Branch        | Purpose                                                         |
| ------------- | --------------------------------------------------------------- |
| `13.x-risefx` | **The fork for Zulip 13.** Upstream `13.x` + the commits below. |
| `12.x-risefx` | **The fork for Zulip 12.** Upstream `12.x` + the same commits.  |
| `main`        | Mirror of upstream `main`. Not used for releases.               |

`main` is the default branch on GitHub, so this README is only visible on the
`*-risefx` branches.

Each `*-risefx` branch is its upstream stable branch plus exactly three
commits:

1. `Add 'riselink' to html_safelisted_schemes`
2. `nginx: Restrict uploaded files to RISEFX internal networks.`
3. `docs: Add RISEFX fork documentation.` (this README)

This README is always the last commit, so you update it with
`git commit --amend`. To see the current base and the three commits:

```bash
git log --oneline -4 13.x-risefx
```

When upstream cuts a new major version, the patches move to a new
`<major>.x-risefx` branch. Keep the previous branch until production no longer
runs that major version, because a security release may still need a build
from it.

## Tags

Format: `<Z>-risefx<P>`, for example `13.3-risefx1`.

- `Z` is the upstream Zulip version. It must match `LATEST_RELEASE_VERSION` in
  [`version.py`](version.py).
- `P` is the RISEFX patch revision. It starts at `1` for each `Z`. Bump it
  when the patches change but `Z` does not.

The trailing digit is **mandatory**, because the `RISEFX/docker-zulip` build
reads `P` from its own tag `<Z>-<R>-risefx<P>` and then looks up
`<Z>-risefx<P>` here. The build fails if that tag does not exist.

Create lightweight tags, like the existing ones. To list the published tags:

```bash
git ls-remote --tags origin 'refs/tags/*-risefx*'
```

## What this fork changes

### 1. `riselink` URL scheme

[`zerver/lib/markdown/__init__.py`](zerver/lib/markdown/__init__.py)

This patch adds `riselink` to `html_safelisted_schemes`. As a result,
`riselink:` URLs in messages survive Markdown rendering and sanitization
instead of being stripped.

The patch adds the scheme to `html_safelisted_schemes` only, **not** to
`auto_linked_schemes`. The two lists have different effects:

- `html_safelisted_schemes` is part of `allowed_schemes`, which `sanitize_url()`
  checks. An explicit Markdown link `[label](riselink:...)` therefore survives.
- `auto_linked_schemes` builds the regex for bare URLs. A bare `riselink:...`
  typed into a message is therefore **not** turned into a link.

Links must be written in `[label](...)` form.

This patch is only half of the feature. The desktop client must also hand
`riselink:` URLs to the operating system, which needs the
`whitelistedProtocols` setting from
[`RISEFX/zulip-desktop`](https://github.com/RISEFX/zulip-desktop). A client
without that setting opens the URL through an interstitial HTML page instead of
launching the RISE application.

### 2. Restrict `/user_uploads` to internal networks

[`puppet/zulip/files/nginx/zulip-include-frontend/app`](puppet/zulip/files/nginx/zulip-include-frontend/app)

This patch puts an nginx `allow`/`deny` list into `location /user_uploads`.
Uploaded files, including their image previews under
`/user_uploads/thumbnail/`, are then served only to these networks:

| Range             | NetBox VLAN  | What it is                            |
| ----------------- | ------------ | ------------------------------------- |
| `127.0.0.1`       | —            | Loopback                              |
| `10.x0.8.0/21`    | `production` | Production workstations, one per site |
| `10.x0.36.128/25` | `VPN admin`  | Admin VPN client range, one per site  |

The second octet encodes the site: `10` is BER, `20` is CGN, `30` is STR, `40`
is MUC and `50` is LON. Every other client gets `403`. The list does not
include the artist VPN client ranges (`10.x0.240.0/21`) or the vendor VPN
client ranges (`10.x0.35.128/25`).

The admin VPN moved to `10.x0.36.128/25` on 2026-10-22, which replaced
`10.10.250.0/24` ([Redmine #74377](https://redmine.rise.fx/issues/74377)).

The `# restrict-uploads-10net` marker makes the block easy to find after a
rebase:

```bash
git grep -n restrict-uploads-10net -- puppet/
```

To check the ranges against NetBox, look each one up:

```bash
curl -s -H "Authorization: Bearer $NETBOX_TOKEN" -H "Accept: application/json" \
  "https://netbox.rise.fx/api/ipam/prefixes/?prefix=10.10.36.128/25"
```

The image build bakes this file in, so the list is not configurable at
runtime. Changing a range needs a new `-risefx<P>` tag and a new image.

> **Note:** nginx `allow`/`deny` checks the socket peer address. If a reverse
> proxy or load balancer sits in front of Zulip, every request appears to come
> from the proxy. The list would then have to name the proxy instead of the
> clients.

## Updating to a new upstream release

The recommended way is the `/zulip-release` skill in
[`RISEFX/docker-zulip`](https://github.com/RISEFX/docker-zulip). It starts
from the tracking issue that docker-zulip's watcher opens for each upstream
release, and it runs the steps below with checks between them. The manual
procedure follows.

### One-time setup

This clone needs an `upstream` remote:

```bash
git remote add upstream https://github.com/zulip/zulip.git
git fetch upstream
```

### Case A — patch release on the same major (for example 13.3 → 13.4)

Upstream adds the release to the existing `13.x` branch.

```bash
git fetch upstream
git switch 13.x-risefx
git rebase upstream/13.x
```

### Case B — new major release (for example 13.x → 14.0)

Upstream cuts a new `14.x` branch. Copy the patches onto it under a new branch
name and keep the old branch.

```bash
git fetch upstream

OLD_BASE=$(git merge-base 13.x-risefx upstream/13.x)
git switch -c 14.x-risefx 13.x-risefx
git rebase --onto upstream/14.x "$OLD_BASE"
```

### Verify the rebase (both cases)

Confirm that exactly the three commits sit on top and that they came through
unchanged:

```bash
# Expect exactly 3 commits.
git log --oneline upstream/14.x..14.x-risefx

# Expect three "=" rows.
git range-diff upstream/13.x..13.x-risefx upstream/14.x..14.x-risefx
```

For Case A, compare against the published branch instead:
`git range-diff origin/13.x-risefx~3..origin/13.x-risefx upstream/13.x..13.x-risefx`.

A `!` row means that upstream changed code under one of the patches. Read that
diff before you continue. A clean rebase does not prove that a patch still
works, because upstream occasionally reorganizes `html_safelisted_schemes` and
the nginx config. Check that both patches still land where they should:

```bash
git grep -n riselink -- zerver/lib/markdown/__init__.py
git grep -n restrict-uploads-10net -- puppet/zulip/files/nginx/zulip-include-frontend/app
```

Then update the volatile parts of this README, such as the branch table and
the version in the pipeline example. Amend them into the README commit.

### Tag and push

`Z` must equal `LATEST_RELEASE_VERSION` in `version.py` on the rebased branch:

```bash
grep LATEST_RELEASE_VERSION version.py

git tag 14.0-risefx1
git push --force-with-lease origin 14.x-risefx
git push origin 14.0-risefx1
```

The rebase rewrites the branch, so pushing it needs `--force-with-lease`. A new
branch pushes without it. Never move or delete a pushed tag, because built
images name it.

Then continue in [`RISEFX/docker-zulip`](https://github.com/RISEFX/docker-zulip)
to build the image, as described in that repo's `README.risefx.md`.

### Changing the patches without an upstream bump

If only the patches change at the same upstream version, for example for a new
internal subnet, bump `P` and leave `Z` alone. Make the change in the patch
commit itself rather than in a new commit, so that the branch keeps exactly
three commits. Use `git rebase -i` with `edit` on that commit. Apply the same
change to every `*-risefx` branch that still gets releases.

```bash
git tag 13.3-risefx2
git push --force-with-lease origin 13.x-risefx
git push origin 13.3-risefx2
```

## Known issues

These are recorded, not fixed. Decide on each one before the next release.

- **`/thumbnail` bypasses the upload restriction.** The legacy endpoint
  `/thumbnail?url=user_uploads/<realm_id>/<path>` calls `serve_file()` in
  `zerver/views/thumbnail.py`. For local storage, `serve_file()` hands the file
  to nginx through `X-Accel-Redirect` to the internal location
  `/internal/local/uploads`, which has no `allow`/`deny` list. A logged-in user
  outside the allowed networks can therefore still download any upload that
  they have access to. Copying the allow list into `location /thumbnail` would
  close this gap.
- **No IPv6 loopback.** The block allows `127.0.0.1` but not `::1`, while
  upstream's `location /api/internal/` allows both. If nginx accepts
  connections on `::1`, local requests to `/user_uploads` get `403`.
