---
description: With and without resetting the database
---

# Version management

After having created and deployed a Squid, it is possible to make improvements, changes, add new functionalities. For this reasons, it is sometimes necessary to update a Squid and make sure it is running on the latest code.

## Authentication

In order to accomplish the tasks below, the terminal window in which the commands are being launched, needs to be authenticated. To do so, refer to the Recipe on [how to obtain a deployment key](obtaining-a-deployment-key.md), and launch the authentication command:

```bash
sqd auth --key sqd_<auht_code>
```

## Redeploy the same version

If, for example, we have made changes to our project and we want to trigger a new release, while maintaining the same version and, consequently, the same endpoint, the `update` subcommand is going to do just that. It will suffice to give all these variables the right values:

* `squid_name`
* `squid_version`
* `github url`
* `github branch`

To visualize this, let's take an example used previously:

![A deployed Squid](../.gitbook/assets/SquidSaas.png)

In this case, the command to trigger a new release of the same version would be:

```bash
sqd squid:update SquidSaas@1 --source <github url>#<github branch>
```

Where the GitHub variables depend on the project itself.

{% hint style="info" %}
In case the release needs a database reset, the `--hardReset` option can be added to the previous command.&#x20;
{% endhint %}

## Deploy a new version

Deploying a new version is as easy as using the same command as releasing the first version, only difference is that, by changing the `squid_version`, a new version will be created. If we take the example used in previous pages, the following command:

```bash
sqd squid:release SquidSaas@2 --source <github url>#<github branch>
```

Will result in a new line being added to Squid web page:

![Multiple versions of the same Squid](../.gitbook/assets/SquidSaas\_v2.png)

## Listing version on the command line

To keep track of which Squids are available and which versions of a Squid are deployed, without consulting the webpage every time, you can use the `ls` command. This will list all Squids deployed for the account linked to the deployment key used to authenticate:

```bash
⇒ sqd squid ls
 Name      Description 
 ───────── ─────────── 
 SquidSaas null   
```

And when using it with the `-n` parameter to specify the Squid name, it can be used to list all its versions:

```bash
⇒ sqd squid ls -n SquidSaass
 version name artifactUrl                                           deploymentUrl                                       Status  Created at 
 ──────────── ───────────────────────────────────────────────────── ─────────────────────────────────────────────────── ─────── ────────── 
 1            https://github.com/<account>/squid-template.git#main https://app.gc.subsquid.io/beta/squidsaas/1/graphql SYNCING 1645701248 
 2            https://github.com/<account>/squid-template.git#main https://app.gc.subsquid.io/beta/squidsaas/2/graphql SYNCING 1645710152 
```