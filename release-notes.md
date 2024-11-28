These instructions assume you have `$VERSION`, `$PROJECT`, and `$REPO` environment variables set in your shell (e.g. `6.1.1`, `citus`, and `citus`). With those set, code from most steps can be copy-pasted.

**After this checklist, you're still not done: open a release checklist in Enterprise and release there, too!**

# Prepare Project

  - [ ] Ensure all needed changes are in the relevant `release-x.y` branch. `git log --cherry-pick --no-merges release-x.y...master` can be helpful. Be sure to cherry-pick changes in the same order they were _merged_ to the main branch (but do not cherry-pick merge commits themselves)
  - [ ] Add a `CHANGELOG` entry in the `master` branch summarizing meaningful changes
  - [ ] Use `git cherry-pick` to add the new `CHANGELOG` entry to the `release-x.y` branch
  - [ ] Use `git tag -a -s v$VERSION` to create an annotated, signed tag for the release. Summarize the release in the one-line tag annotation (beneath 52 characters). Push the tag with `git push origin v$VERSION`
  - [ ] Visit the project's releases page (e.g. `open https://github.com/citusdata/$REPO/releases`)
    - [ ] Create a new release object for your git tag (i.e. `v$VERSION`). Leave the description blank (it will auto-fill with the tag description)
    - [ ] Click the _Source code (tar.gz)_ link for the new version to download a source tarball
    - [ ] Sign the tarball using `gpg -a -u 3F95D6C6 -b $PROJECT-$VERSION.tar.gz`
    - [ ] Back on the releases page, click the _Edit_ button for your release. Attach the signature (`$PROJECT-$VERSION.tar.gz.asc`) and click the _Update release_ button

If you lack the signing key, it's in our 1Password vault. Ask around. As for signing tags, use your own key, and ensure it's [known to GitHub](https://help.github.com/articles/adding-a-new-gpg-key-to-your-github-account/).

# Update OS Packages

  - [ ] Check out the `debian-$PROJECT` branch of the [packaging repository](https://github.com/citusdata/packaging); create a new branch for your changes
    - [ ] Add a new entry (`$VERSION.citus-1`, `stable`) to the `debian/changelog` file
    - [ ] Update the `pkglatest` variable in the `pkgvars` file to `$VERSION.citus-1`
    - [ ] Test the Debian release build locally: `citus_package -p=debian/jessie -p=debian/wheezy -p=ubuntu/xenial -p=ubuntu/trusty -p=ubuntu/precise local release 2>&1 | tee -a citus_package.log`
    - [ ] Test the Debian nightly build locally: `citus_package -p=debian/jessie -p=debian/wheezy -p=ubuntu/xenial -p=ubuntu/trusty -p=ubuntu/precise local nightly 2>&1 | tee -a citus_package.log`
    - [ ] Ensure no warnings or errors are present: `grep -Ei '(warning|\bi|\be|\bw):' citus_package.log | sort | uniq -c`. Ignore any warnings about _using a gain-root-command while being root_ or _Recognised distributions_
    - [ ] Push these changes to GitHub and open a pull request, noting any errors you found
  - [ ] Check out the `redhat-$PROJECT` branch of the [packaging repository](https://github.com/citusdata/packaging); create a new branch for your changes
    - [ ] Update the `pkglatest` variable in the `pkgvars` file to `$VERSION.citus-1`
    - [ ] Edit the `$PROJECT.spec` file, being sure to:
      - [ ] Update the `Version:` field
      - [ ] Update the `Source0:` field
      - [ ] Add a new entry (`$VERSION.citus-1`) to the `%changelog` section
    - [ ] Test the Red Hat release build locally: `citus_package -p=el/7 -p=el/6 -p=fedora/24 -p=fedora/23 -p=ol/7 -p=ol/6 local release 2>&1 | tee -a citus_package.log`
    - [ ] Test the Red Hat nightly build locally: `citus_package -p=el/7 -p=el/6 -p=fedora/24 -p=fedora/23 -p=ol/7 -p=ol/6 local nightly 2>&1 | tee -a citus_package.log`
    - [ ] Ensure no warnings or errors are present: `grep -Ei '(warning|\bi|\be|\bw):' citus_package.log | sort | uniq -c`. Ignore any errors about `--disable-dependency-tracking`
    - [ ] Push these changes to GitHub and open a pull request
  - [ ] Get changes reviewed; use the "squash" strategy to close the PR
  - [ ] Ensure both Travis builds completed successfully (new releases should be in packagecloud)

# Update Docker

  - [ ] Create a `release-$VERSION` branch in your [docker repository checkout](https://github.com/citusdata/docker), based on `develop`
  - [ ] If needed, bump the version of the base PostgreSQL image in the `FROM` instruction of the main `Dockerfile`
  - [ ] Bump the Citus version number in the `Dockerfile`, `docker-compose.yml`, and `tutum.yml` files
  - [ ] Add a new entry in the `CHANGELOG` noting that the Citus version has been bumped (and the PostgreSQL one, if applicable)
  - [ ] Locally build your image and test it standalone and in a cluster: `docker build -t citusdata/citus:$VERSION .`
  - [ ] Open _two_ pull request: one against `develop`, and one against `master`. Get one or the other reviewed, and merge both
  - [ ] Tag the latest `master` as `v$VERSION`: `git fetch && git tag -a -s v$VERSION origin/master && git push origin v$VERSION`
  - [ ] Ensure the Docker Hub builds (e.g. https://hub.docker.com/r/citusdata/citus/builds) complete successfully

# Update CloudFormation

  - [ ] Check out the `master` branch of the [cloudformation repository](https://github.com/citusdata/cloudformation)
  - [ ] Update the two places in `citus-6/citusdb.json` which reference the Citus version (search and replace)
  - [ ] Commit and push to GitHub
  - [ ] Upload the template as `citus-$VERSION.json` to S3 (in the folder `citus-deployment/aws/citus6/cloudformation`)

# Update Documentation

  - [ ] Check out the `master` branch of the [documentation repository](https://github.com/citusdata/citus_docs)
  - [ ] Bump version occurrences in `conf.py` and `installation/multi_machine_aws.rst`
  - [ ] Open a Pull Request

# Update PGXN

  - [ ] Check out the `pgxn-$PROJECT` branch of the [packaging repository](https://github.com/citusdata/packaging)
  - [ ] Bump all version occurrences in `META.json`
  - [ ] Test locally with `citus_package -p=pgxn local release`
  - [ ] Push these changes to GitHub and open a pull request
  - [ ] After merging, ensure the Travis build completed successfully (a new release should appear in PGXN eventually)

# Update PGDG

PGDG has separate teams for Red Hat and Debian builds.

## Red Hat

  - [ ] Create a new feature request in [the RPM Redmine](https://redmine.postgresql.org/projects/pgrpms/issues/new) asking their team to update the Citus version in their spec file
  - [ ] Wait for the issue to be closed and verify that the package is available in the PGDG RPM repo

## Debian

  - [ ] Check out the `debian` branch of the [packaging repository](https://github.com/citusdata/packaging)
  - [ ] Add a new entry (`$VERSION.PGDG-1`, `unstable`) to the `debian/changelog` file
  - [ ] Commit this change and push to GitHub
  - [ ] Visit the PGDG Jenkins [page for Citus](https://pgdgbuild.dus.dg-i.net/job/citus-source/) and click the _Build with Parameters_ link
  - [ ] Leave all parameters as default and press the _Build_ button
  - [ ] After success, the `citus-binaries` project will build. When it's done, the next `dput` task will upload the build artifacts to [testing](https://wiki.postgresql.org/wiki/Apt/FAQ#What_are_the_testing_distributions.3F)
  - [ ] Verify the package works, then [open an issue](https://redmine.postgresql.org/projects/pgapt/issues/new) to have it promoted from `testing`

The PGDG Jenkins site uses client-side certificates for login, which is a bit obnoxious to manage. If you don't have a certificate for logging in, [follow these instructions](https://wiki.postgresql.org/wiki/Apt/Jenkins#Jenkins_Login).
