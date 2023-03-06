WARNING: This is the htop debian source package example. It is created only for learning and testing, **no one else should be using it.**

---

# Purpose

This document explains the steps needed to update the existing debian package.

For this example we are going to update the **htop** package by:

1. Adding a preinstall script to print the message just before the package is installed.
2. Adding a test script in the source and install it as part of package installation, the script name will be `testing.sh` and it will be installed in `/usr/bin`.

# Env Setup

## Software installation

Install required packages

```sh
sudo apt install gnupg pbuilder ubuntu-dev-tools apt-file debhelper
```

# Setup build/dev env

**pbuilder** can be used to build the binary packages in an isolated env without touching the host system. Build env is created for each ubuntu release.

Create a new env:

```sh
pbuilder-dist create kinetic
```

# Account setup

THe existing and the updated package will be managed via launchpad, for this we need to create the launchpad account, setup GPG keys for package signing and ssh for uploading the packages.

## GPG keys:

- `gpp --full-generate-key`
- `gpg --send-keys --keyserver keyserver.ubuntu.com <id>`

## SSH

- `ssh-keygen -t rsa`
- `cat ~/.ssh/id_rsa.pub`

## Launchpad

- Create account @ https://launchpad.net/
- Import GPG key: `gpg --fingerprint <email>`
- Import SSH key: `cat ~/.ssh/id_rsa.pub`
- Create a new PPA repository

For detailed instructions please refer to https://packaging.ubuntu.com/html/getting-set-up.html.

# Updating/Fixing a bug in a package

1. Install all required packages as explained in [Env Setup](#env-setup)

2. Find package from a file

- To download the correct source, first we need to find the associated binary package and from that we can also find the source package using the following steps.
  - First update apt-file cache: `sudo apt-file update`
  - Find the binary package: `apt-file find /usr/bin/htop`
  - Find the associated source package
    (**you may have to enable source repos in the sources.list**): `apt-cache showsrc htop`

3. Get the source package

   - Download the package source using: `pull-lp-source htop kinetic`

4. [Optional Step] Add source to git repo

   - Create a repo in github/gitlab
   - Upload the initial version to git

   ```sh
     git init
     touch README.md
     git add .
     git commit -m "Initial commit: Unchanged version from launchpad."
     git remote add origin <git URL>
     git push -u origin master
   ```

5. Add preinst script to log a message during package installation

   - go to the package source: `cd htop-3.2.1/debian`
   - add a file names preinst : `touch preinst`
   - update permissions : `chmod 755 preinst`
   - add script contents as needed
   - add **#DEBHELPER#** placeholder before the script exits.

6. Add a script testing.sh in the source tree, here I used scripts directory

   - go to the required directory: `cd htop-3.2.1/scripts`
   - create a new script: `touch testing.sh`
   - update permissions : `chmod 755 preinst`
   - add script contents as needed

7. Update install script to install **testing.sh** when the package is installed

   - go to debian scripts directory: `cd htop-3.2.1/debian`
   - add following line in the file named `install`

   ```sh
   scripts/testing.sh usr/bin
   ```

8. [Optional step] Commit all changes in git

   - `git add .`
   - `git commit -m "Added preinst and testing.sh"`
   - `git push`

9. Commit changes to source package (git is duplication of the efforts, it is not really need in normal workflow)

   - `dpkg-source --commit`
   - add a patch name and commit message as required

10. Update package commit history

    - open the commit log with: `dch -i`
    - change the target from "UNRELEASED" to 'kinetic' (or as applicable)
    - Add other comments as needed

11. [Optional Step] Above steps create new artifacts, commit them in git

    - `git add .`
    - `git commit -m "Updated dpkg metadata and changelog"`
    - `git push`

12. Build the updated source package

    - build signed source package: `debuild -S -d -k<gpg key id>`
    - verify there are no lint errors/warnings
    - the source build will create two important files
    - \*.dsc: file needed to build the binary package
    - \*.change: file need to upload the source package in lanuchpad
    - these files are created in the parent directory of the source

13. Build the update binary package

    - binary packages are build in an isolated build env with **pbuiler**
    - create env for required release: `pbuilder-dist kinetic create`
    - build the binary deb package: `pbuilder-dist kinetic build ../htop_3.2.1-1ubuntu1.dsc`
    - the binary package will be avaiable in: `~/pbuilder/kinetic_result/htop_3.2.1-1ubuntu1_amd64.deb`

14. Basic test

    - Check package contents: `dpkg --contents htop_3.2.1-1ubuntu1_amd64.deb`
    - It should list **/usr/bin/testing.sh**
    - Install the package: `dpkg -i /htop_3.2.1-1ubuntu1_amd64.deb`
    - Verify that pre-install script is called
    - Verify that htop and testing.sh is present in /usr/bin

15. Upload to PPA

    - Ensure that PPA is setup as explained in [Launchpad](#launchpad)
    - Upload the package in PPA: `dput ppa:dzambare/dpkgtest ../htop_3.2.1-1ubuntu1_source.changes`
    - Wait for email confirmation of pacakge acceptance
    - Wait for binary package build is available 
    - In case the upload is rejected fix the issue, update the package and upload new version

16. Install from PPA

    - Add the PPA: `sudo add-apt-repository ppa:dzambare/dpkgtest`
    - Verify that the updated version of htop package is available:

      ```sh
      $ apt-cache show htop | grep Version
      Version: 3.2.1-1ubuntu1
      Version: 3.2.1-1
      ```

    - Install the updated version

17. [Optional Step] Commit final artifacts to git.

    - `git add .`
    - `git commit -m "Artifacts for version 3.2.1-1ubuntu1"`
    - `git push`

---

# Common Issues

This section documents the issues faced during implementation and suggested fixes.

## 1 debuild failure because 'dh' not found

```sh
$ debuild -S -d -uc -us

<output stripped>

 dpkg-source --before-build .
 debian/rules clean
dh clean
make: dh: No such file or directory
make: *** [debian/rules:24: clean] Error 127
dpkg-buildpackage: error: debian/rules clean subprocess returned exit status 2
debuild: fatal error at line 1182:
dpkg-buildpackage -us -uc -ui -S -d failed
```

### Solution

Install debhelper package

```sh
sudo apt-get install debhelper
```

## 2 'dput' failure because of GPG signing

The default build command is not able to sign packages, dput reports following error:

```sh
dput ppa:dzambare/dpkgtest ../htop_3.2.1-1ppa1_source.changes
D: Splitting host argument out of  ppa:dzambare/dpkgtest.
D: Setting host argument.
Checking signature on .changes
gpg: ../htop_3.2.1-1ppa1_source.changes: error 58: gpgme_op_verify
gpgme_op_verify: GPGME: No data
```

### Solution

Option 1: Append the key GPG key id to debuild command (ensure there is no space between -k and the key)

```sh
debuild -S -d -k<GPG key id>
```

Option 2: Sign the .change and .dsc file manually

```sh
debsign -k <GPG Key id> htop_3.2.1-1ppa2_source.changes
```

## 3. Linting errors

#### 1. bad-distribution-in-changes-file

While building a package using debuild the linter reported the following error. The existing change logs have the target field as unstable but it didn't work for this change.

```sh
E: htop changes: bad-distribution-in-changes-file unstable
```

### Solution:

Edit the change log with `dch -i` and update the target field to one of the valid ubuntu releases like "kinetic".

#### 2. maintainer-script-lacks-debhelper-token

While buildig a package using debuild the linter reported the following error.

```sh
W: htop source: maintainer-script-lacks-debhelper-token [debian/preinst]
```

### Solution:

The special debian scripts need a placeholder for automated changes to be done by the tooling. To fix this add **#DEBHELPER#** in the script just before exit.

## 4. PPA addition results in 403 forbidden error

After creating the PPA adding the PPA reported 403 error.

### Solution

If the launchpad is showing everything correctly, wait for a couple of hours.
