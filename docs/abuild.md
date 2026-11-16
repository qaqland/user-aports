# abuild cheat sheet

How packages are built here, in the order you actually run it.

## Workflow

Normal case, a package that already builds:

```sh
cd user/<pkg>
abuild checksum   # fetch sources into ~/.cache/abuild, (re)write sha512sums
abuild deps       # install depends + makedepends + checkdepends
abuild            # full build, then refresh and sign the repo index
```

`abuild -f` forces a rebuild; you only need it when abuild prints
"Package is up to date" and skips the run. Artifacts land in
`~/packages/user/x86_64/`.

`abuild deps` needs no sudo/doas: it uses `SUDO_APK=abuild-apk` (setuid helper,
group `abuild`) and passes `--repository "$REPODEST/$repo"`, so packages built by
this repo resolve as dependencies.

When something fails, run the stages one by one instead:

```sh
abuild clean      # drop src/ pkg/ tmp/ from earlier runs
abuild unpack     # extract sources into src/            (prepare does NOT unpack)
abuild prepare    # run prepare(); the default applies the patches
abuild build      # run build()
abuild check      # run check()
abuild package    # run package() only
abuild rootpkg    # package() + subpackages, create the .apk as fakeroot
```

`abuild` with no command runs the same stages plus index refresh and signing.

## Verify

```sh
abuild -rK        # -r: install missing deps, -K: keep src/ pkg/ tmp/ afterwards
```

`-rK` is the inspection run: afterwards look at `pkg/` to see what really got
packaged (`package()` output, split results) instead of guessing from the `.apk`.

```sh
abuild -f rootbld # build in a clean chroot (needs the abuild-rootbld package)
```


The dependency gate: the chroot only has base plus the deps you declared, so a
missing or wrong dependency fails here (host packages cannot hide it). This is what
an aports MR will hit, so run it before submitting.

```sh
abuild-apk add ~/packages/user/x86_64/<pkg>-<ver>-r0.apk   # install for testing
```

Downgrades are accepted, so this also works when only `pkgrel` was lowered.

## Housekeeping

```sh
abuild checksum    # after touching source= or patches (it also fetches)
abuild index       # re-sign/refresh the repo index (after deleting apks by hand)
abuild clean       # drop src/ pkg/ tmp/
abuild cleanoldpkg # remove old .apk versions from REPODEST
abuild undeps      # uninstall the makedepends virtual package
```

## Patches

Patches are plain files listed in `source=`, with one line each in `sha512sums`:

```sh
source="https://example.org/foo-$pkgver.tar.gz
	001-build-fix.patch
	"
```

`abuild prepare` runs the package's `prepare()` hook; the default applies them in
`$source` order with `patch -p1`, so:

- if the APKBUILD defines its own `prepare()`, it has to call `default_prepare`
  first, otherwise nothing is applied;
- write them so `-p1` works: `diff -u --label a/<path> --label b/<path>` on the
  pristine tree, or `git diff` in a git checkout of it;
- after changing a patch: `abuild checksum`, then `abuild unpack && abuild prepare`.

