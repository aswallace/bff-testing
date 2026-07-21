---
title: "Testing a markdown schema"
date: 2026-07-13
author: "Anya Wallace"
doi: 10.1038/s41592-026-03130-w
publication: https://doi.org/10.1038/s41592-026-03130-w
provenance_url: https://docs.google.com/spreadsheets/d/e/2PACX-1vTzcU7vJdglvA0d4ypI51bAwkU5DD-4erGQIiUuaFA4pf_3z-j2-wN47-uqu1UWkqSNdLkWxpHvjGZ5/pub?output=csv
dataset_url: https://docs.google.com/spreadsheets/d/e/2PACX-1vSgzDzX5_3RepRdCxx1H-q34jckZcZYiir0pu9rjwVfTl5QyfdqdpaBR_QLAT5E6SwDDF-ucDiKt6eJ/pub?output=csv 
something else: "foo"
another thing: "bar"
---

# Introduction
This is a larger markdown file which contains the entire bff readme

Project layout
==============

This project is structured as a multi-package repository. There are three packages:
1. `packages/core`,
2. `packages/desktop`, and
3. `packages/web`.

### Core
`core` is the primary source of the application. It primarily exports a React component: the BioFile Finder application. It also exports a 
function for instantiating the Redux store to be used with the application, as well as some other interfaces and constants that are helpful 
within the packages that render the application.


### Making use of "core"
Both `desktop` and `web` depend on `core`. They are responsible for rendering the React component exported by `core` and wiring it together 
with its Redux store. They may optionally define services that implement interfaces defined within `core`. These select interfaces are made 
available for implementation outside of `core` because they are identified as being "platform-dependent," meaning we may accomplish 
implementing the interfaces differently within Electron than within a traditional web browser--or we may opt to not implement a service within 
one of those platforms altogether.


### Why the split
The reasoning behind the split in packages is simple: it allows the application to be distributed both as a desktop
application (internal-facing, more richly featured), which is the published artifact of `desktop`, and as a web application (external-facing, 
feature-limited), which is the published artifact of `web`.

Setup and workflow
==================

To get started with a _typical_ frontend project, one would clone the repository and run `npm install`. To add a
dependency, one would run `npm install [package-name]`. But because the source code for this project is [split across
three subpackages](01-project-layout.md), this works slightly differently in this project.

To help with the management of three interconnected packages, this project makes use of
[npm workspaces](https://docs.npmjs.com/cli/v8/using-npm/workspaces). The two most important features we get from using `npm workspaces` are:
1) the deduplication of dependencies; and
2) handling of multipackage versioning.


### System requirements
1. NodeJS version 24.x (use `nvm` or similar)
2. NPM version 11.x


### Initial setup
```bash
$ git clone git@github.com:AllenInstitute/biofile-finder.git
$ cd biofile-finder
$ npm ci  # or, `npm install` if you want to pick up dependency updates; you need to commit your package-lock.json afterwards, though.
```

**N.b: There is no need to go into a subpackage and run `npm install`.**


### Periodic workspace maintenance
Once setup locally, if remote has progressed beyond your local repo, you should periodically refresh your workspace:
```bash
$ git pull
$ git clean -Xfd
$ npm ci
```


### Developing
In many cases, you'll need to make changes to the `core` package, and you'll want to see how those changes play out in the context of a
rendered application, like `desktop` or `web`.
To make that happen:
1. To start the desktop version, run `npm --prefix packages/desktop run start`. Otherwise to start the web version, run `npm --prefix packages/web run start`.

### Testing
Most components in the project have associated unit tests; to run the full suite, run `npm run test`.
The unit tests for each of the packages can also be run independently, using `npm run test:core`, `npm run test:desktop`, 
or `npm run test:web`, respectively.

To run the linter, use `npm run lint`, and for typechecking, use `npm run typeCheck`.

### Adding a dependency to to either the `desktop` or `web` subpackages
In the case that you need to add either a dependency or devDependency to either the `desktop` or `web` subpackages within this
project, use the `--workspace` option of `npm install`.

A dependency should be added to one of these subpackages instead of in the project (monorepo) root in the case that it only
affects that particular subpackage, such as its build process or something about its particular runtime.


Some examples:
1. Add `lolcatjs` as a dependency of `packages/desktop`:
```
npm install --save --workspace packages/desktop lolcatjs
```
2. Add `lolcatjs` as a shared dependency within monorepo:
```
npm install --save lolcatjs
```

Using a local `file-explorer-service`
=====================================

Some features need to be developed across this codebase and the `file-explorer-service`. And in some other cases, it can
be helpful to do manual testing via the frontend of a feature or bugfix done within `file-explorer-service`. In each of
these situations, there is a mechanism built into the BioFile Finder for using a locally running version of the
`file-explorer-service`.

Instructions:
1. Run `file-explorer-service` in your favorite way such that it is accessible from
`http://localhost:9081/file-explorer-service`. Note that it must be running without SSL, accessible through `localhost`,
and running on port `9081`. If you run `file-explorer-service` on one computer (e.g., an in-office workstation) but have
the frontend running on another computer (e.g., your laptop), you can make the service available through `localhost` by
making use of port forwarding (e.g.: `ssh -L 9081:localhost:9081 dev-aics-gmp-001.corp.alleninstitute.org -N -f`).
2. From within the running Electron application, under the "Data Source" menu bar option, select "Localhost."

Versioning and deployment/publishing
====================================


The strategy for versioning and deployment/publishing within this project is complicated by the fact that there are two distinct
types of artifacts produced. That is, `packages/desktop` produces platform-specific executables for linux, macOS, and Windows,
while `packages/web` produces a static website intended to be dumped into an S3 website bucket (or otherwise sit
behind a web server).

# Web

1) Make sure all your changes are committed and merged into `main`.
2) Navigate to the GH workflow actions and select "Run workflow" along with the branch `main` & `staging`.
   ![image](./assets/WorkflowButton.png)
3) Review deploy action & approve if it looks good.
   ![image](./assets/DeployReview.png)
4) Test out changes on [staging](https://staging.biofile-finder.allencell.org/app).
5) Navigate to the GH workflow actions and select "Run workflow" along with the branch `main` & `production`.
6) Review deploy action & approve if it looks good.
7) Test out changes on [production](https://biofile-finder.allencell.org/app)

# Desktop

The following captures the steps of a release of this project to desktop:

1) Make sure all your changes are committed and merged into main.
2) Make sure branch is clean:
    ```bash
    git checkout main
    git stash
    git pull
    ```
3) Determine version bump type, choose one of `patch`, `minor`, or `major`, depending on the scale of the changes including in this version bump. This will create a version like `v<version>`. See [below for a guideline to which version to increment to](#versioning-information).
    ```bash
    # You need to choose one of 'patch', 'minor', or 'major'
    export VERSION_BUMP_TYPE=<one of patch, minor, or major>
    ```
4) Create tag and push new version to GitHub like so:
    ```bash
    npm --no-commit-hooks version --workspace packages/desktop $VERSION_BUMP_TYPE -m "v%s"
    ```
    Verify that the `"version"` property in all `package.json` files matches the new version number; otherwise, update them to match. 
5) Wait for a [GitHub Action](https://github.com/AllenInstitute/biofile-finder/actions) to automatically create new platform-specific
builds of `packages/desktop`, prepare a draft Github release, and upload the builds as release artifacts to that release.
6) [Update the GitHub release](https://github.com/AllenInstitute/biofile-finder/releases) once the Github action in Step 4 is finished, manually edit the Github release which was drafted as part of Step 4. Format its release name with the date (consistent with other release names), add a description of the changes, and optionally
mark whether the release is "pre-release." If it is marked as "pre-release," it will not be accessible for download through the
Github pages site.

### Development Builds for Specific Branches

You can generate development builds from any branch using the [manual-build](https://github.com/AllenInstitute/biofile-finder/actions/workflows/manual-build.yml) GitHub Action. Select the branch from the dropdown in the GitHub Action interface and choose the appropriate runner environment from the following options:
```
ubuntu-latest for Linux builds
windows-latest for Windows builds
macOS-latest for ARM builds
macOS-13 for x86 builds
```
Once you trigger the workflow, the process will generate a zipped artifact containing all the build artifacts. Download this artifact from the workflow, unzip it to access the build. Please note that these zipped artifacts are available for 7 days after which they are automatically deleted.

Code signing
============

The build/release process makes use of built-in code signing hooks provided by
[electron-builder](https://www.electron.build/code-signing#how-to-disable-code-signing-during-the-build-process-on-macos).
Refer to that documentation and other widely-available information for a general introduction.

### Windows distribution
The certificate and its associated password used to sign the Windows artifact are stored as
[repo-level secrets](https://github.com/AllenInstitute/biofile-finder/settings/secrets/actions) in Github.
The certificate is base64 encoded and stored in the `CSC_LINK` secret.
The certificate password is stored in the `CSC_KEY_PASSWORD` secret.

Both the code signing certificate and its password are additionally stored in
ansible-platform in `inventory/group_vars/desktop_apps` for posterity.

### Mac distribution
This macOS artifact is not yet code signed.

### Linux distribution
N/A

Monitoring, metrics, and event tracking
=======================================

This project makes use of [`@aics/frontend-insights`](https://aicsbitbucket.corp.alleninstitute.org/projects/SW/repos/frontend-insights/browse/packages/frontend-insights) 
to abstract how application monitoring, metrics, and user event tracking are done, and where that data is sent.

### User event tracking

#### On Web

We use [Microsoft Clarity](https://clarity.microsoft.com/projects). New developers will need to request access from an existing user.

#### On Desktop
[Amplitude](https://amplitude.com/) is currently used as our user event tracking platform; it is plugged into `@aics/frontend-insights` via 
[`@aics/frontend-insights-plugin-amplitude-node`](https://aicsbitbucket.corp.alleninstitute.org/projects/SW/repos/frontend-insights/browse/packages/frontend-insights-plugin-amplitude-node).

In order to test a new usage of user events (e.g., to ensure intended event properties are set), create an `.env` file in 
`packages/desktop`, following the example of `packages/desktop/.env.example`. Set the `AMPLITUDE_API_KEY` to the API key for the 
`fms-file-explorer-test` project in Amplitude: https://analytics.amplitude.com/allencell/settings/projects/308551/general. If you 
do not have access to Amplitude yet, ask an Amplitude administrator.
