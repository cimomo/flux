---
title: Using Weave Flux
menu_order: 40
---

All of the features of Flux are accessible from within
[Weave Cloud](https://cloud.weave.works).

However, `fluxctl` provides an equivalent API that can be used from
the command line. The `--help` for `fluxctl` is described below.

```sh
fluxctl helps you deploy your code.

Workflow:
  fluxctl list-controllers                                           # Which controllers are running?
  fluxctl list-images --controller=deployment/foo                    # Which images are running/available?
  fluxctl release --controller=deployment/foo --update-image=bar:v2  # Release new version.

Usage:
  fluxctl [command]

Available Commands:
  automate         Turn on automatic deployment for a controller.
  deautomate       Turn off automatic deployment for a controller.
  help             Help about any command
  identity         Display SSH public key
  list-controllers List controllers currently running on the platform.
  list-images      Show the deployed and available images for a controller.
  lock             Lock a controller, so it cannot be deployed.
  policy           Manage policies for a controller.
  release          Release a new version of a controller.
  save             save controller definitions to local files in platform-native format
  unlock           Unlock a controller, so it can be deployed.
  version          Output the version of fluxctl

Flags:
  -h, --help           help for fluxctl
  -t, --token string   Weave Cloud controller token; you can also set the environment variable WEAVE_CLOUD_TOKEN or FLUX_SERVICE_TOKEN
  -u, --url string     base URL of the flux controller; you can also set the environment variable FLUX_URL (default "https://cloud.weave.works/api/flux")

Use "fluxctl [command] --help" for more information about a command.
```

# What is a Controller?

This term refers to any cluster resource responsible for the creation of
containers from versioned images - in Kubernetes these are workloads such as
Deployments, DaemonSets, StatefulSets and CronJobs.

# Viewing Controllers

The first thing to do is to check whether Flux can see any running
controllers. To do this, use the `list-controllers` subcommand:

```sh
$ fluxctl list-controllers
CONTROLLER                     CONTAINER   IMAGE                                         RELEASE  POLICY
default:deployment/helloworld  helloworld  quay.io/weaveworks/helloworld:master-a000001  ready
                               sidecar     quay.io/weaveworks/sidecar:master-a000002
```

Note that the actual images running will depend on your cluster.

# Inspecting the Version of a Container

Once we have a list of controllers, we can begin to inspect which versions
of the image are running.

```sh
$ fluxctl list-images --controller default:deployment/helloworld
CONTROLLER                     CONTAINER   IMAGE                          CREATED
default:deployment/helloworld  helloworld  quay.io/weaveworks/helloworld
                                           |   master-9a16ff945b9e        20 Jul 16 13:19 UTC
                                           |   master-b31c617a0fe3        20 Jul 16 13:19 UTC
                                           |   master-a000002             12 Jul 16 17:17 UTC
                                           '-> master-a000001             12 Jul 16 17:16 UTC
                               sidecar     quay.io/weaveworks/sidecar
                                           '-> master-a000002             23 Aug 16 10:05 UTC
                                               master-a000001             23 Aug 16 09:53 UTC
```

The arrows will point to the version that is currently running
alongside a list of other versions and their timestamps.

# Releasing a Controller

We can now go ahead and update a controller with the `release` subcommand.
This will check whether each controller needs to be updated, and if so,
write the new configuration to the repository.

```sh
$ fluxctl release --controller=default:deployment/helloworld --user=phil --message="New version" --update-all-images
Submitting release ...
Commit pushed: 7dc025c
Applied 7dc025c61fdbbfc2c32f792ad61e6ff52cf0590a
CONTROLLER                     STATUS   UPDATES
default:deployment/helloworld  success  helloworld: quay.io/weaveworks/helloworld:master-a000001 -> master-9a16ff945b9e

$ fluxctl list-images --controller default:deployment/helloworld
CONTROLLER                     CONTAINER   IMAGE                          CREATED
default:deployment/helloworld  helloworld  quay.io/weaveworks/helloworld
                                           '-> master-9a16ff945b9e        20 Jul 16 13:19 UTC
                                               master-b31c617a0fe3        20 Jul 16 13:19 UTC
                                               master-a000002             12 Jul 16 17:17 UTC
                                               master-a000001             12 Jul 16 17:16 UTC
                               sidecar     quay.io/weaveworks/sidecar
                                           '-> master-a000002             23 Aug 16 10:05 UTC
                                               master-a000001             23 Aug 16 09:53 UTC
```

# Turning on Automation

Automation can be easily controlled from within
[Weave Cloud](https://cloud.weave.works) by selecting the "Automate"
button when inspecting a controller. But we can also do this from `fluxctl`
with the `automate` subcommand.

```sh
$ fluxctl automate --controller=default:deployment/helloworld
Commit pushed: af4bf73
CONTROLLER                     STATUS   UPDATES
default:deployment/helloworld  success

$ fluxctl list-controllers --namespace=default
CONTROLLER                     CONTAINER   IMAGE                                             RELEASE  POLICY
default:deployment/helloworld  helloworld  quay.io/weaveworks/helloworld:master-9a16ff945b9e ready    automated
                               sidecar     quay.io/weaveworks/sidecar:master-a000002
```

We can see that the `list-controllers` subcommand reports that the
helloworld application is automated. Flux will now automatically
deploy a new version of a controller whenever one is available and commit
the new configuration to the version control system.

# Turning off Automation

Turning off automation is performed with the `deautomate` command:

```sh
$ fluxctl deautomate --controller=default:deployment/helloworld
Commit pushed: a54ef2c
CONTROLLER                     STATUS   UPDATES
default:deployment/helloworld  success

$ fluxctl list-controllers --namespace=default
CONTROLLER                     CONTAINER   IMAGE                                             RELEASE  POLICY
default:deployment/helloworld  helloworld  quay.io/weaveworks/helloworld:master-9a16ff945b9e ready
                               sidecar     quay.io/weaveworks/sidecar:master-a000002
```

We can see that the controller is no longer automated.

# Rolling back a Controller

Rolling back can be achieved by combining:

- [`deautomate`](#turning-off-automation) to prevent Flux from automatically updating to newer versions, and
- [`release`](#releasing-a-controller) to deploy the version you want to roll back to.

```sh
$ fluxctl list-images --controller default:deployment/helloworld
CONTROLLER                     CONTAINER   IMAGE                          CREATED
default:deployment/helloworld  helloworld  quay.io/weaveworks/helloworld
                                           '-> master-9a16ff945b9e        20 Jul 16 13:19 UTC
                                               master-b31c617a0fe3        20 Jul 16 13:19 UTC
                                               master-a000002             12 Jul 16 17:17 UTC
                                               master-a000001             12 Jul 16 17:16 UTC
                               sidecar     quay.io/weaveworks/sidecar
                                           '-> master-a000002             23 Aug 16 10:05 UTC
                                               master-a000001             23 Aug 16 09:53 UTC

$ fluxctl deautomate --controller=default:deployment/helloworld
Commit pushed: c07f317
CONTROLLER                     STATUS   UPDATES
default:deployment/helloworld  success

$ fluxctl release --controller=default:deployment/helloworld --update-image=quay.io/weaveworks/helloworld:master-a000001
Submitting release ...
Commit pushed: 33ce4e3
Applied 33ce4e38048f4b787c583e64505485a13c8a7836
CONTROLLER                     STATUS   UPDATES
default:deployment/helloworld  success  helloworld: quay.io/weaveworks/helloworld:master-9a16ff945b9e -> master-a000001

$ fluxctl list-images --controller default:deployment/helloworld
CONTROLLER                     CONTAINER   IMAGE                          CREATED
default:deployment/helloworld  helloworld  quay.io/weaveworks/helloworld
                                           |   master-9a16ff945b9e        20 Jul 16 13:19 UTC
                                           |   master-b31c617a0fe3        20 Jul 16 13:19 UTC
                                           |   master-a000002             12 Jul 16 17:17 UTC
                                           '-> master-a000001             12 Jul 16 17:16 UTC
                               sidecar     quay.io/weaveworks/sidecar
                                           '-> master-a000002             23 Aug 16 10:05 UTC
                                               master-a000001             23 Aug 16 09:53 UTC
```

# Locking a Controller

Locking a controller will stop manual or automated releases to that
controller. Changes made in the file will still be synced.

```sh
$ fluxctl lock --controller=deployment/helloworld
Commit pushed: d726722
CONTROLLER                     STATUS   UPDATES
default:deployment/helloworld  success
```

# Unlocking a Controller

Unlocking a controller allows it to have manual or automated releases
(again).

```sh
$ fluxctl unlock --controller=deployment/helloworld
Commit pushed: 708b63a
CONTROLLER                     STATUS   UPDATES
default:deployment/helloworld  success
```

# Recording user and message with the triggered action

Issuing a deployment change results in a version control change/git
commit, keeping the history of the actions. The Flux daemon can be
started with several flags that impact the commit information:

| flag              | purpose                       | default |
|-------------------|-------------------------------|------------|
| git-user          | committer name                | Weave Flux |
| git-email         | committer name                | support@weave.works |
| git-set-author    | override the commit author    | false |

Actions triggered by a user through the Weave Cloud UI or the CLI `fluxctl`
tool, can have the commit author information customized. This is handy for providing extra context in the
notifications and history. Whether the customization is possible, depends on the Flux daemon (fluxd)
`git-set-author` flag. If set, the commit author will be customized in the following way:

# Image Tag Filtering

When building images it is often useful to tag build images by the branch that they were built against for example:

```
quay.io/weaveworks/helloworld:master-9a16ff945b9e
```

Indicates that the "helloworld" image was built against master commit "9a16ff945b9e". 

When automation is turned on flux will, by default, use whatever is the latest image on a given repository. If you want to only auto-update your image against a certain subset of tags then you can do that using tag filtering.

So for example, if you want to only update the "helloworld" image to tags that were built against the "prod" branch then you could do the following:

```
fluxctl policy --controller=default:deployment/helloworld --tag-all='prod-*'
```

If your pod contains multiple containers then you tag each container individually:

```
fluxctl policy --controller=default:deployment/helloworld --tag='helloworld=prod-*' --tag='sidecar=prod-*'
``` 

## Actions triggered through Weave Cloud

Weave Cloud UI sends user parameter, value of which is the username (email) of the user logged into
Weave Cloud.

## Actions triggered through `fluxctl`

`fluxctl` provides the following flags for the message and author customization:

  -m, --message string      message associated with the action
      --user    string      user who triggered the action

Commit customization

    1. Commit message

       fluxctl --message="Message providing more context for the action" .....

    2. Committer

        Committer information can be overriden with the appropriate fluxd flags:

        --git-user
        --git-email

        See [site/daemon.md] for more information.

    3. Commit author

        The default for the author is the committer information, which can be overriden,
        in the following manner:

        a) Default override uses user's git configuration, ie user.name
           and user.email (.gitconfig) to set the commit author.
           If the user has neither user.name nor for
           user.email set up, the committer information will be used. If only one
           is set up, that will be used.

        b) This can be further overriden by the use of the fluxctl --user flag.

        Examples

        a) fluxctl --user="Jane Doe <jane@doe.com>" ......
            This will always succeed as git expects a new author in the format
            "some_string <some_other_string>".

        b) fluxctl --user="Jane Doe" .......
            This form will succeed if there is already a repo commit, done by
            Jane Doe.

        c) fluxctl --user="jane@doe.com" .......
            This form will succeed if there is already a repo commit, done by
            jane@doe.com.

## Errors due to author customization

In case of no prior commit by the specified author, an error will be reported
for b) and c):

git commit: fatal: --author 'unknown' is not 'Name <email>' and matches
no existing author
