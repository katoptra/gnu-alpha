# gnu-alpha

The pipeline in this repository copies GNU's alpha release tree, `alpha.gnu.org/gnu`, from
GNU's primary site into Cloudflare R2. The mirror serves the tree at
`https://gnu-alpha.katoptra.org/`. `README.md` identifies the upstream, and it tells how to
use the mirror and how to fork it. The README of [katoptra/lib](https://github.com/katoptra/lib)
is the manual for the parts that all mirrors share. This file gives the rules that each
change must obey.

This repository does not start runs. An external scheduler dispatches `sync.yml`. Only the
scheduler has the times of the runs. A run does `reconcile` if it is 23.5 hours or more since
a run did the last reconcile.

`Taskfile.yml` contains only the root vars and the two includes, and each verb is lib's.

## Constraints

- **Where a change goes.** `Taskfile.yml` has no verbs. Make all changes to verbs in lib: in
  the toolbox or in the rsync engine. Then each mirror that includes that file gets the
  change. An example is a change to the movement of bytes, to the pages or to the checks of
  `smoke`. The includes have no `excludes:`.
- **Root vars.** Root vars hold only the values of this mirror. Do not put an engine default
  in a root var, because then the command line cannot set it
  ([lib README, Rules a mirror keeps](https://github.com/katoptra/lib#rules-a-mirror-keeps)).
  Do not set a lib var again. In an engine verb, the engine uses a root var, not a
  `KEY=value` from the command line. Thus, `MAX_BATCHES`, `BATCH_GB` and `RECONCILE` are
  not root vars.
- **`SOURCE` must be `ftp.gnu.org`, the primary site, not a secondary mirror.** A secondary
  mirror gets a change 1 h or more after `ftp.gnu.org`. GNU recommends a secondary mirror,
  to decrease the load on `ftp.gnu.org`, but the owner selected the primary site. Do not
  change `SOURCE` to a secondary mirror without the approval of the owner. Before such a
  change, compare the paths and sizes of the secondary mirror with the listing of
  `rsync://ftp.gnu.org/alpha/`. The alpha list is on GNU's mirror page, in "Mirroring the
  GNU Alpha release server".
- **Paths are GNU's, at the root of the bucket, with the dot-files.** Only the mirror uses
  the prefix `.state/` and the file name `katoptra.org.directory.index.html`. The
  alpha tree does not have these two names.
- **GNU's mirror guidelines are applicable to each page that the mirror serves.** Text must
  be "as short as possible, and strictly explanatory". Images and logos are not permitted.
  The only permitted link is a link for bug reports.
- **The footer is the only text that the mirror adds.** The engine makes it from `UPSTREAM`
  and `HOST`. Do not set `PAGE_FOOT`. A change to `UPSTREAM` or `HOST` gets to each page at
  the next run. That run sends approximately 430 PutObjects.
- **The bug-report link is `mailto:gnu-alpha@katoptra.org`.** That address must receive
  e-mail. The mail records of the zone are not in this repository. They are in the same
  location as the zone rules.
- **Storage.** For each change that adds storage, calculate the new storage. Compare it with
  the 122 GB baseline and the 160 GB ceiling.
- **Writing.** Use ASD-STE100 and the rules in the
  [Writing section of the org CONTRIBUTING](https://github.com/katoptra/.github/blob/main/CONTRIBUTING.md#writing).
  Read that section before you write.

## Must knows

- **Symlinks become objects.** `backgammon`, `dmd` and `fetish` are symlinks upstream, to
  `gnubg`, `shepherd` and `coreutils`. With `-L`, rsync gets the trees that they point to.
  Thus, the bucket contains these 0.58 GB two times, and clients get the same files at
  those URLs as from all other GNU mirrors. Each `-latest` name is also a symlink, and the
  bucket contains a copy of its file. Do not add an exclude for these paths.
- **rsync gives the exit code 23 for each listing.** Two symlinks that point to no file
  cause it: `cflow/gdbm-latest.tar.gz.sig` and `gnutrition/gnutrition-latest.tar.gz`. The
  engine accepts the exit code 23. rsync also gives this code for a directory that it
  cannot read ([lib, The rsync engine](https://github.com/katoptra/lib#the-rsync-engine)).
  `LIST_FLOOR: 6500` stops a run if the listing decreases by more than approximately 700
  files.
- **The zone rules are not in this repository.** README step 4 gives them. The canary finds
  a missing rule: `smoke` reads `emacs/.readme.shtml` as `libwww-perl`.
- **No `Content-Encoding`.** GNU recommends no `Content-Encoding` header. On each run,
  `smoke` reads a `.tar.gz` and makes sure that it has no `Content-Encoding`.
- **Freshness is upstream's.** `mirror-updated-timestamp.txt` contains the epoch time of
  alpha.gnu.org, which writes it each hour. This mirror copies the file from ftp.gnu.org.
  This mirror does not write a timestamp.
- **The keyring is in a different tree.** Each release has its `.sig` file in this tree.
  The GNU keyring that verifies them is at the root of `https://gnu.katoptra.org/`.
- **A run failure is the only alert.** The healthchecks.io check has a cron expression with
  the same times as the slots of the mirror, in UTC, and a grace time of 3 hours.

## Verifying a change

- `task check` makes a render of each command of the pipeline in the image. Then it compares
  the render with `render.txt`. `task render-update` accepts a change. On each pull request,
  the check workflow does the same `task check`, with no secrets.
- `task run -- task list` gets a listing of `ftp.gnu.org`, with no credentials. It is the
  one check of upstream that you can do without the vault. `.run/upstream.txt` then has
  approximately 7,200 lines, and `mirror-updated-timestamp.txt` is one of them.
- `task plan` does the read-only part of a run with the bucket of the mirror: `clock`,
  `list`, `state`, `diff` and `split`. It uses the secrets from the vault.
- The verbs of the engine, the pages and the read-back checks are lib's. To do a test of
  them, use `cd ../lib/examples/rsync && task run -- task offline`.
- Freshness: compare `curl -s https://gnu-alpha.katoptra.org/mirror-updated-timestamp.txt`
  with `date +%s`.
