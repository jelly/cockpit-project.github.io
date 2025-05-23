---
title: Cockpit Bots
source: https://github.com/cockpit-project/bots/blob/master/README.md
---

These are automated bots and tools that work on Cockpit. This includes updating
operating system images, updating translations or NPM modules, testing PRs, and
more.

## Images

In order to test Cockpit-related projects, they are staged into an operating
system image. These images are tracked in the `images/` directory. For example,
you might want to test a scenario where Cockpit on one machine talks to FreeIPA
on another, and you want those two machines to use different images.

This is handled by passing a specific image to image-create
and other scripts that work with test machine images. Available images include:

 - `fedora-*`, `rhel-*`, `debian-*`, etc: Various operating systems for testing Cockpit related projects
 - `services`: Auxiliary network services for tests which are independent from
   the OS where Cockpit runs: FreeIPA, Samba AD, candlepin, Grafana

These well known image names are expected to contain no `.`
characters and have no file name extension.

Individual projects are expected to locally build their code into packages, and
install them as overlay on top of these pristine images, with `image-customize`
or using the [machine Python API](./machine/machine_core/).

For managing these images:

 - `image-download`: Download selected or all test images
 - `image-create`: Create test machine images from scratch (usually through
   downloading a cloud image), with common build and test dependencies for
   Cockpit projects preinstalled
 - `image-upload`: Upload a locally built test image to the official image servers

For running and debugging the images:

 - `image-customize`: Install packages, upload files, or run commands in a test
   machine image; this keeps the original image intact, and puts the changes
   into an image overlay into test/images/.
 - `vm-run`: Run a test machine image; by default this happens in an ephemeral
   overlay. You can use the `--maintain` option to write into the persistent
   overlay in test/images/ instead.
 - `vm-reset`: Remove all overlays from test/images/

## Image location

Downloaded images are stored into ~/.cache/cockpit-images/ by default. If you
want to change that, you can set `$COCKPIT_IMAGES_DATA_DIR` or the
`cockpit.bots.images-data-dir` variable with `git config` to a directory where
to store the pristine virtual machine images.  For example:

    git config cockpit.bots.images-data-dir /srv/cockpit/images

## Tests

The bots automatically run the tests as needed on pull requests
and branches. To check when and where tests will be run, use the
tests-scan tool:

    ./tests-scan -vd

#### Note on eslintrc interaction

As eslint looks for additional configurations, eslintrc.(json|yaml) files, in
parent directories, it is recommended to have `"root": true` in the eslint
configuration of any project which is using eslint and is tested through
cockpit-bots.

## Integration with GitHub

A number of machines are watching our GitHub repositories and are
executing tests for pull requests as well as making new images.

Most of this happens automatically, but you can influence their
actions with the tests-trigger utility in this directory.

### Setup

You need a GitHub token in ~/.config/cockpit-dev/github-token or from
the [GitHub CLI](https://cli.github.com/) configuration in
~/.config/gh/config.yml.  You can create one for your account at
[Developer Settings → Personal access tokens](https://github.com/settings/tokens).

When generating a new personal access token, the scopes should contain
`repo:status` and `read:org`. Note in particular, that `repo` and
`public_repo` scopes each grant full push access, and should not be used.

You need at least "Write" access to the project for triggering statuses, either
individually per repo (e.g. [cockpit](https://github.com/cockpit-project/cockpit/settings/access)
or for [all cockpit-project repos](https://github.com/orgs/cockpit-project/teams/committers).

If you'd like to download Red Hat-only internal images from S3, you'll
need to create a key file in `~/.config/cockpit-dev/s3-keys/[domain]`.
The `[domain]` can be any non-toplevel domain which contains the S3 URL
in question.  The contents of this file should be a single line
containing the "access key" and the "secret key" separated by
whitespace.

For the currently configured mirrors this means that you'd likely have the
following file:

- `~/.config/cockpit-dev/s3-keys/linodeobjects.com`

For more control, you could also use the following:

- `~/.config/cockpit-dev/s3-keys/cockpit-images.eu-central-1.linodeobjects.com`
- `~/.config/cockpit-dev/s3-keys/eu-central-1.linodeobjects.com`
- either of the above, with `us-east` instead of `eu-central`

each file would be a single line which looks like

```
EEVIDIDFSOQ0ABJ2LGTT    009rKOypIoqO44Q3VQGRyYPfugi84zANHF0pOW9f
```

The "access key" and "secret key" is unique per-developer and can be
obtained by talking to Allison.

### Test contexts

For describing tests which we want to run we use __contexts__. A context has the form:

    image[/scenario][@bots#bots_pr][@owner/project/ref]

where items have the following meaning:
- image: Name of the image on which tests should run (e.g. 'fedora-coreos').
- scenario: Name of a specific test. This is specific for each separate project and
  is passed verbatim to 'test/run' in `$TEST_SCENARIO`.
- bots_pr: Number of pull request that exists in bots repository. When specified,
  bots from this PR would be used instead of main.
- owner/project: Name of github project (e.g. 'cockpit-project/cockpit'). This part can
  be omitted when testing in the same project and no 'ref' is needed.
- ref: Reference in the project (usually branch) (e.g. 'rhel-8.2'). Default is
  the project's primary branch.

For example, context for scenario 'firefox' on 'fedora-coreos' is:

    fedora-coreos/firefox

If we want to trigger it on 'cockpit-project/cockpit':

    fedora-coreos/firefox@cockpit-project/cockpit

If we want to also not run it on the primary branch, but on 'rhel-8-0' branch:

    fedora-coreos/firefox@cockpit-project/cockpit/rhel-8-0

If we want to run tests on 'fedora-coreos' but with bots from pull request '169':

    fedora-coreos@bots#169

### Retrying a failed test

If you want to run the "fedora-coreos" testsuite again for pull
request #1234 of cockpit-project/cockpit, run tests-trigger like so:

    ./tests-trigger --repo cockpit-project/cockpit 1234 fedora-coreos

You can also invoke bots/tests/trigger from any project checkout, in which case
you don't need the explicit `--repo` -- it will default to the GitHub origin of
the current directory's project.

### Testing a pull request by a non-allowed user

If you want to run all tests on pull request #1234 that has been opened by
someone who does not have push access to the repository nor isn't in the
[allowlist](https://github.com/cockpit-project/bots/blob/main/lib/allowlist.py)
run tests-trigger with `--allow`:

    ./tests-trigger --allow [...]

Of course, you should make sure that the pull request is proper and
doesn't execute evil code during tests.

### tests-trigger with a different origin

If you need to specify --repo in tests-trigger as your remote is different from
cockpit-project/cockpit, you can set a git configuration option from which
tests-trigger reads the repo. This has to be set per cockpit project.

    git config cockpit.bots.github-repo cockpit-project/cockpit

### Refreshing a test image

Test images are refreshed automatically once per week, and even if the
last refresh has failed, the machines wait one week before trying again.

If you want the machines to refresh the fedora-coreos image immediately,
run image-trigger like so:

    ./image-trigger fedora-coreos

### Creating new images for a pull request

If as part of some new feature you need to change the content of some
or all images, you can ask the machines to create those images.

If you want to have a new fedora-coreos image for pull request #1234, add
a bullet point to that pull request's description like so, and add the
"bot" label to the pull request.

    * [ ] image-refresh fedora-coreos

The machines will post comments to the pull request about their
progress and at the end there will be links to commits with the new
images.  You can then include these commits into the pull request in
any way you like.

### Creating a new image

Creating a new image from scratch requires some
[images/scripts/](./images/scripts) files. For a new image called `tux` we need:

- `tux.bootstrap`: Download or create an initial qcow2 image which boots and has SSH
  acccess. If available, this should be the latest available cloud image,
  but it may also invoke some other VM build tool or build service.
  This script runs in the [cockpit/tasks](https://github.com/cockpit-project/cockpituous/tree/main/tasks/container)
  container, and thus can only use the tools installed there. If necessary, add
  them first.
- `tux.setup`: The setup script runs inside the downloaded test image,
  install all required build and test dependencies, and sets up an `admin`
  user.

For a new image it is recommended to follow the existing setup/bootstrap scripts,
for example the `fedora` one.

Run `./image-create -v tux` to build the image. If that succeeds, a new image
is saved in `images` as `images/tux` (a symlink to the real qcow2 file).
Test-boot it with `./vm-run tux`, and ensure you can:

 - Log in with SSH as the `admin` and `root` user with our usual test SSH key
 - Become root with `sudo` as admin works (password "foobar")

To add that image to our CI, create a PR and follow the "Creating new
images for a pull request" section. External contributors will need to ask a
Cockpit team member to create a copy of the PR and follow this workflow.

For the initial PR it is recommended to add the new image to the `_manual`
testmap of `starter-kit` or other project to prove that the created image is
functional.

### Updating CI to a new Fedora release

`TEST_OS_DEFAULT` is usually set to the latest (stable) Fedora released,
used as default OS for test VMs.

1. If this is a new image, add `_manual` test contexts for the new image to `lib/testmap.py`, and land that into `main`.
2. Create a PR that updates `TEST_OS_DEFAULT` in `lib/constants.py`, and trigger all tests for that image there.

#### Fedora CoreOS

The Fedora CoreOS image is updated to a new Fedora release out of our
control, when this occurs:

1. Update the naughty symlink `naughty/fedora-coreos` to the release
   CoreOS uses.
2. Update `OSTREE_BUILD_IMAGE` to point to the Fedora release CoreOS
   uses.

####  Pixel tests

The pixel tests used in Cockpit projects use `test/reference-image` to
determine what image to run the pixel tests on.

1. Create a PR which updates `test/reference-image`.
2. Update the pixel tests if required.