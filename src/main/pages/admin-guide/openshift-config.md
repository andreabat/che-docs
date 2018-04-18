---
title: "Configuration: OpenShift"
keywords: openshift, configuration
tags: [installation, openshift]
sidebar: user_sidebar
permalink: openshift-config.html
folder: setup-openshift
---

## Admin Guide

This page describes OpenShift specific configuration. Refer to [OpenShift Admin Guide][openshift-admin-guide] to general information that works both for OS and K8S.

## How It Works

Che server is configured via environment variables passed to Che deployment. You can do it when initially deploying Che (See: [Installation Single User][openshift-single-user], [Multi-User][openshift-multi-user]) or afterwards, by editing Che deployment.

There are multiple ways to edit Che deployment to add new or edit existing envs:

* `OC_EDITOR="nano" oc edit dc/che` opens Che deployment yaml in nano editor (VIM is used by default)
* manually in OpenShift web console > deployments > Che > Environment
* `oc set env dc/che KEY=VALUE KEY1=VALUE1` updates Che deployment with new envs or modifies values of existing ones


## What Can Be Configured?

You can find deployment env or config map in yaml files. However, they do not reference a complete list of environment variables that Che server will respect.

Here is a [complete](https://github.com/eclipse/che/tree/master/assembly/assembly-wsmaster-war/src/main/webapp/WEB-INF/classes/che) list of all properties that are configurable for Che server.

You can manually convert properties into envs, just make sure to follow [instructions on properties page](properties.html#properties-and-environment-variables)

## HTTPS Mode

To enable https for server and workspace routes, follow instructions in [setup docs single user][openshift-single-user] and [multi-user][openshift-multi-user]. To migrate an existing Che deployment to https, do the following:

<span style="color:red;">IMPORTANT!</span> Self-signed certificates aren't acceptable.

1. Update Che deployment with `PROTOCOL=https, WS_PROTOCOL=wss, TLS=true`
2. Manually edit or recreate routes for Che and Keycloak `oc apply -f https`
3. Once done, go to `https://keycloak-${NAMESPACE}.${ROUTING_SUFFIX}`, log in to admin console.
Default credentials are `admin:admin`.
Go to Clients, `che-public` client and edit **Valid Redirect URIs** and **Web Origins** URLs so that they use **https** protocol.
You do not need to do that if you initially deploy Che with https support.

## Private Docker Registries

Refer to [OpenShift documentation](https://docs.openshift.com/container-platform/3.7/security/registries.html)

## Enable ssh and sudo

By default, pods are run with an arbitrary user that has a randomly generated UID (the range is defined in OpenShift config file). This security constrain has several consequences for Eclipse Che users:

* installers for language servers will fail since most of them require `sudo`
* no way to run any sudo commands in a running workspace

It is possible to allow root access which in its turn allows running system services and change file/directory [permissions](#filesystem-permissions). You can change this behavior. See [OpenShift Documentation for details](https://docs.openshift.com/container-platform/3.6/admin_guide/manage_scc.html#enable-images-to-run-with-user-in-the-dockerfile).

You may also configure some services to bind to ports below `1024`, say, apache2. Here's an example of enabling it for [Apache2](https://github.com/eclipse/che-dockerfiles/blob/master/recipes/php/Dockerfile#L49) in a PHP image.

**How to Get a Shell in a Pod?**

Since OpenShift routes do not support ssh protocol, once cannot run sshd (or equivalent) in a pod and ssh into it. However, OpenShift itself provides a few alternatives (only for users who can authenticate as a user that has deployed Che):

* `oc rsh ${POD_NAME}` (you can get running pods with `oc`). Note that this is a remote shell, not an ssh connection
* in an OpenShift **web console, projects > ws-namespace > pods > pod details > Terminal**.

Once Che server is able to create OpenShift objects on behalf of a current user, rsh will be available for all users. You may follow GitHub [issue](https://github.com/eclipse/che/issues/8178) to get updates.

## Filesystem Permissions

As said above, pods in OpenShift are started with an arbitrary user with a dynamic UID that is generated for each namespace individually. As a result, a user in an OpenShift pod does not have write permissions for files and directories unless root group (UID - `0`) has write permissions for those (an arbitrary user in OpenShift belongs to root group). All Che ready to go stacks are optimized to run well on OpenShift. See an example from a [base image](https://github.com/eclipse/che-dockerfiles/blob/master/recipes/stack-base/centos/Dockerfile#L45-L48). What happens there is that a root group has write permissions for `/projects` (where workspace projects are located), a user home directory and some other dirs.


## Multi-User: Using Own Keycloak and PSQL

Out of the box Che is deployed together with Keycloak and Postgres pods, and all three services are properly configured to be able to communicate. However, it does not matter for Che what Keycloak server and Postgres DB to use, as long as those have compatible versions and meet certain requirements.

Follow instructions on deploying multi-user [Che without Keycloak or Postgres or both][openshift-multi-user].

***Che Server and Keycloak***

Keycloak server URL is retrieved from the `CHE_KEYCLOAK_AUTH__SERVER__URL` environment variable. A new installation of Che will use its own Keycloak server running in a Docker container pre-configured to communicate with Che server. Realm and client are mandatory environment variables. By default Keycloak environment variables are:

```
CHE_KEYCLOAK_AUTH__SERVER__URL=http://${KC_ROUTE}:5050/auth
CHE_KEYCLOAK_REALM=che
CHE_KEYCLOAK_CLIENT__ID=che-public
```

You can use your own Keycloak server. Create a new realm and a public client. A few things to keep in mind:

* It must be a public client
* `redirectUris` should be `${CHE_SERVER_ROUTE}/*`. If no or incorrect `redirectUris` are provided or the one used is not in the list of `redirectUris`, Keycloak will display an error saying that redirect_uri param is invalid.
* `webOrigins` should be  either`${CHE_SERVER_ROUTE}` or `*`. If no or incorrect `webOrigins` are provided, Keycloak script won't be injected into a page because of CORS error.


***Using an alternate OIDC provider instead of Keycloak***

Instead using a Keycloak server, Che now provides a limited support for alternate authentication servers compatible with the [OpenId Connect specification](http://openid.net/specs/openid-connect-core-1_0.html).

Some limitations restrict the alternate OIDC providers that can be used with Eclipse Che. Supported providers should:
- implement access tokens as JWT tokens including at least the following claims:
    - `exp`: the expiration time (https://tools.ietf.org/html/rfc7519#section-4.1.4)
    - `sub`: the subject (https://tools.ietf.org/html/rfc7519#section-4.1.2)
- allow redirect Urls with wildcards at the end
- provide an endpoint that returns the [OpenID Provider Configuration information](http://openid.net/specs/openid-connect-discovery-1_0.html#ProviderConfig). According to the specification, this endpoint should end with sub-path `/.well-known/openid-configuration`.

When using an alternate OIDC provider, the following Keycloak environment variables should be set to `NULL`:

```
CHE_KEYCLOAK_AUTH__SERVER__URL=NULL
CHE_KEYCLOAK_REALM=NULL
```

Instead, you should set the folowing environement variables:

```
CHE_KEYCLOAK_CLIENT__ID=<client id provided by the OIDC provider>
CHE_KEYCLOAK_OIDC__PROVIDER=<base URL of the OIDC provider that provides a configuration endpoint at `/.well-known/openid-configuration` sub-path>
```

If the optional [`nonce` OpenId request parameter](http://openid.net/specs/openid-connect-core-1_0.html#AuthRequest) is not supported, the following environment variable should be added:

```
CHE_KEYCLOAK.USE__NONCE=FALSE
```

***Che Server and PostgreSQL***

Che server uses the below defaults to connect to PostgreSQL to store info related to users, user preferences and workspaces:

```
CHE_JDBC_USERNAME=pgche
CHE_JDBC_PASSWORD=pgchepassword
CHE_JDBC_DATABASE=dbche
CHE_JDBC_URL=jdbc:postgresql://postgres:5432/dbche
CHE_JDBC_DRIVER__CLASS__NAME=org.postgresql.Driver
CHE_JDBC_MAX__TOTAL=20
CHE_JDBC_MAX__IDLE=10
CHE_JDBC_MAX__WAIT__MILLIS=-1
```

Che currently uses version 9.6.


***Keycloak and PostgreSQL***

Database URL, port, database name, user and password are defined as environment variables in Keycloak pod. Defaults are:

```
POSTGRES_PORT_5432_TCP_ADDR=postgres
POSTGRES_PORT_5432_TCP_PORT=5432
POSTGRES_DATABASE=keycloak
POSTGRES_USER=keycloak
POSTGRES_PASSWORD=keycloak
```

## Development Mode

After you have built your [custom assembly][assemblies], execute `build.sh` [script](https://github.com/eclipse/che/tree/master/dockerfiles/che). You can then tag it, either push to MiniShift or a public Docker registry, and reference in your Che deployment as `CHE_IMAGE_REPO` and `CHE_IMAGE_TAG`. Alternatively, you may make sure the image is available locally and change pull policy to `IfNotPresent` in che deployment.

## Che Workspace Termination Grace Period

Info about changing workspace termination grace period can be found in the following [section](kubernetes-config.html#che-workspace-termination-grace-period) of the Che Kubernetes config document.

{% include links.html %}