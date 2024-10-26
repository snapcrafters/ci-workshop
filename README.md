# CI workshop

This guide explains how the snapcrafters' automation works and how you can add it to your snap.

## Prerequisites

* Your `snapcraft.yaml` should explicitly state which [architectures](https://snapcraft.io/docs/architectures) it should be built on.
* Your `snapcraft.yaml` should have **hardcoded versions** of the app and upstream dependencies. This is to make our builds more reproducible. That way, we can rebuild a snap without immediately pulling in the latest version of an app. This also makes it very clear what version of an app will be built from a given `snapcraft.yaml` file.
* The main branch of your snap's repository should be called `candidate`. The name of the branch should reflect the name of the track you're releasing to.

## Complete flow of actions

There are basically two flows. The first is a set of workflows to release new updated versions of your snap, using the following actions.

* Every night, [`sync-version-with-upstream.yml`](https://github.com/snapcrafters/signal-desktop/blob/candidate/.github/workflows/sync-version-with-upstream.yml) checks if a new version of your application or any dependencies are available. If there is a new version available, this workflow updates `snapcraft.yaml` and commits this directly to the repository. This synchronization script has to be customized for each snap.
* After every new commit to the `candidate` branch, [`release-to-candidate.yaml`](https://github.com/snapcrafters/signal-desktop/blob/candidate/.github/workflows/release-to-candidate.yaml) runs.
  * It builds the snap and pushes it to the `candidate` channel in the Snap Store.
  * It creates a "call for testing" issue asking people to test this version.
  * It runs the snap in a VM and takes two screenshots.
* When a maintainer replies in the call for testing, the [`promote-to-stable.yml`](https://github.com/snapcrafters/signal-desktop/blob/candidate/.github/workflows/promote-to-stable.yml) checks if this comment is a command. If it is a command, this workflow publishes the snap to the given channels.

The second flow is one workflow to build, test and review pull requests.

* Every time someone creates a new PR, [`pull-request.yml`](https://github.com/snapcrafters/signal-desktop/blob/candidate/.github/workflows/pull-request.yml) runs. It builds the snap and checks if it passes automated reviews.

The actual common logic for all these actions is in our [`ci` repository](https://github.com/snapcrafters/ci)

## How to add the automation code to your snap

Clone your repository, download your clone locally, and start updating your snap.

1. If your snap always downloads the latest version of the app and its dependencies, you need to update `snapcraft.yaml` first. Remove the code that gets the latest version, and change it by code that has the version hardcoded. For example, by using the top-level `version` keyword, and/or by using the `source-tag` keywords for parts.
1. Most snaps can simply copy `pull-request.yml`, `release-to-candidate.yaml`, and `promote-to-stable.yml`. These workflows should work for any snap by default.
1. The next step is to copy and update `sync-version-with-upstream.yml`. You need to update the `update-script` part of this workflow. This is a bash script that updates your snapcraft.yaml file to use the latest versions.
   1. The first step in this script is to get the latest version of the application from an online source. This can be GitHub, Gitlab, or any other website. For example:

      * Get the latest non-beta signal-desktop version from a **GitHub Release**.

        ```shell
        URL="https://api.github.com/repos/signalapp/Signal-Desktop/releases"
        VERSION=$(curl -sL ${URL} |  jq .[].tag_name -r | grep -v beta | sort -r | head -n 1  | tr -d 'v')
        ```

      * Get the latest version of dav1d from a **GitLab Release**:

        ```shell
        VERSION=$(curl -s https://code.videolan.org/api/v4/projects/198/releases/ | jq '.[0].tag_name' -r)
        ```

      * Get the latest version from a **Git Branch Name**:

        ```shell
        VERSION=$(git -c 'versionsort.suffix=-' ls-remote --refs --sort='v:refname' --heads https://git.videolan.org/git/ffmpeg.git | tail --lines=1 | cut --delimiter='/' --fields=3,4)
        ```

      * Get the latest version from a **Git Tag**:

        ```shell
        VERSION=$(git -c 'versionsort.suffix=-' ls-remote --refs --sort='v:refname' --tags https://code.videolan.org/videolan/dav1d.git | tail --lines=1 | cut --delimiter='/' --fields=3)
        ```


      * Get the latest version of Discord from their **deb repository**.

        ```shell
        DEB_API="https://discord.com/api/download?platform=linux&format=deb"
        DEB_URL=$(curl -w "%{url_effective}\n" -I -L -s -S "${DEB_API}" -o /dev/null)
        VERSION=$(echo "${DEB_URL}" | cut -d'/' -f6)
        ```

   1. The next step is to update the version in your `snapcraft.yaml` file. For example:

      * This updates **the main version** of your snap.

        ```shell
        sed -i 's/^\(version: \).*$/\1'"\"$VERSION\""'/' snap/snapcraft.yaml
        ```

      * This updates the **`source-tag`** property of the `nv-codec-headers` part.

        ```shell
        yq -i ".parts.nv-codec-headers.source-tag = \"$VERSION\"" snap/snapcraft.yaml
        ```

        > Note: using `yq` is often easier than `sed`, but it can change some whitespace in your snap.

   If your `snapcraft.yaml` contains multiple libraries or dependencies with specific version, your script should gather the latest version of each one and update `snapcraft.yaml` accordingly.

   Some applications require specific versions for their dependencies. In that case, you need to do a little bit more work.

   For example [signal-desktop downloads](https://github.com/snapcrafters/signal-desktop/blob/candidate/.github/workflows/sync-version-with-upstream.yml) the upstream `packages.json` and extracts the versions of dependencies from that file.

## How to set up credentials

The secrets used by the GitHub actions are stored in an Environment, in order to add a second layer of protection.

Each repository has one environment: "Candidate Branch". This environment is locked to the `candidate` branch.

![image](https://github.com/snapcrafters/.github/assets/1492981/8073d05c-af0c-4a26-a00d-3320437fa034)


It has four secrets:

* `SNAPCRAFTERS_BOT_COMMIT` is a GitHub token for `snapcrafters-bot` that has write access to **only** that single snap's repository and the `ci-screenshots` repository. You create it on GitHub in Settings > Developer settings > Fine-grained Personal Access Tokens.
  * Name: `snapcrafters/<repo> commit`
  * Resource owner: `snapcrafters`
  * Repository access: "Only select repositories" > `<repo>` and `ci-screenshots`
  * Permissions: Contents > "Access: Read and write".

* `LP_BUILD_SECRET` is a Launchpad token that can be used to build snaps on Launchpad. You generate this by running `snapcraft remote-build` on your local machine, login using the `snapcrafters-bot` account, and copy the token with the following command.

  ```shell
  cat ~/.local/share/snapcraft/provider/launchpad/credentials
  ```

* `SNAP_STORE_CANDIDATE` is a Snap Store token that is only allowed to push a snap to the `candidate` channel. You can generate it with the following command.

  ```shell
  SNAP_NAME=mattermost-desktop
  snapcraft export-login --snaps=${SNAP_NAME} \
     --acls package_access,package_push,package_update,package_release \
     --channels candidate \
     --expires 2024-12-07 \
     SNAP_STORE_CANDIDATE
  ```
 
* `SNAP_STORE_STABLE` is a Snap Store token that is only allowed to promote a snap from `candidate` to the `stable` channel. You can generate it with the following command.

  ```shell
  SNAP_NAME=mattermost-desktop
  snapcraft export-login --snaps=${SNAP_NAME} \
    --acls package_access,package_release \
    --channels stable \
    --expires 2024-12-07 \
    SNAP_STORE_STABLE
  ```
