# ReShare work on VuFind

<!-- md2toc -l 2 README.md -->
* [Background](#background)
    * [Sources of information](#sources-of-information)
    * [Who's who?](#whos-who)
    * [The task](#the-task)
* [Development information](#development-information)
    * [The development UI](#the-development-ui)
    * [Accessing the development server](#accessing-the-development-server)
    * [What's on the development server](#whats-on-the-development-server)
    * [How to do the work](#how-to-do-the-work)
    * [Patron identity](#patron-identity)
    * [Contacting ReShare](#contacting-reshare)
    * [Accessing the patron-validation service](#accessing-the-patron-validation-service)
* [Notes](#notes)
* [TODO](#todo)



## Background


### Sources of information

The information in this document is mostly taken from a Slack conversation at
https://indexdata.slack.com/archives/DGCAAHEJ2/p1663859084271569

The relevant Jira issues are
[PR-1279](https://openlibraryfoundation.atlassian.net/browse/PR-1279)
and
[DEVOPS-1108](https://jira.indexdata.com/browse/DEVOPS-1108).


### Who's who?

* IPLC is the Ivies Plus Library Consortium.
* Borrow Direct is the branding of their ILL service.
* ReShare is implementing Borrow Direct for IPLC.


### The task

We need to ensure that a patron is allowed to make a request from VuFind only if he[1] is authorized to do so. We can only make this determination once we know who the patron is. That can happen either because of a just-in-time login in order to place a request, or when logging in for some other reason.

When a patron autheticates onto VuFind, we can determine who he is by having ReShare look him up in the local library's NCIP service. When it responds, we will know what patron profile the patron belongs to, and therefore whether or not the patron is authorized to initiate ILL, and can allow or deny the request accordingly. We can also disable (or leave enabled) the patron's *ILL* button for the rest of the session; and if we determined the patron's identity from a non-JIT login, we need never show a bait-and-switch *ILL* button at all.

The authentication check here is just an optimization. ReShare already prevents an unauthorized patron from placing a request; the present work is to avoid the aunauthorized patron from thinking he gets to place a request, only to find out later that ReShare rejected it. (XXX Awaiting [confirmation](https://openlibraryfoundation.atlassian.net/browse/PR-1279?focusedCommentId=469770).)



## Development information


### The development UI

The production UI is https://borrowdirect.reshare.indexdata.com/

It should be shadowed by a development UI at https://vufind.folio-dev.indexdata.com/iplc

You can create an account to login as directly from the UI. But at some point we'll need to use a username that gets a valid NCIP response from some library's NCIP server. To find this, we can probably look at an existing tenant see what some previous requesters were, or ask Debra. She can point to an appropriate system to test on.


### Accessing the development server

`ssh mike@vufind-flo.folio-dev.indexdata.com`
(using public/private key authentication)

(The hostname appears to specific to FLO, the Fenway Library Organization, but that is just a historical accident regarding A and CNAME entries in DNS. It is in reality our VuFind server for all our clients.)


### What's on the development server

```
export VUFIND_HOME=/usr/local/vufind-8
export VUFIND_LOCAL_DIR=/usr/local/vufind-8/iplc
```
The rest of the customers currently run VuFind v7, and their configurations are in `/usr/local/vufind`. We can ignore these.

`/etc/apache2/conf-enabled/vufind-iplc.conf`
controls the site.
It loads up the ReShare and IPLC modules.

> **NOTE.**
> This file is a link to `/usr/local/vufind-8/iplc/httpd-vufind.conf`
> but all git operation fail in the directory hosting that file, saying
> `detected dubious ownership in repository at '/usr/local/vufind-8'`
> -- why?

The Apache2 configuration file loads VuFind modules `iplc` and `ReShare`. We will mostly want to work in the latter, so that the results can be shared among different ReShare customers. (The only thing the ReShare module does at this point is override the default OpenURL behavior to provide stuff you need for the listener.)

Source code where the current work is to be done can be found at https://github.com/indexdata/vufind-iplc
and is in `/opt/vufind-iplc` on the server.
Note that this does _not_ include `/usr/local/vufind-8/iplc` which is configuration, full of private information that needs to be templated out.


### How to do the work

The [VuFind developer documentation](https://vufind.org/wiki/development) is handy. The How to/cookbook session in there is nice. And somewhere in there is documention on how to extend classes from the existing module in your local module. Documentation on extending/overriding a class with code generators is at https://vufind.org/wiki/development:code_generators

If adding new classes, it will be necessary to hand-edit `ReShare/config/module.config.php`. You can peek at the VuFind modules config for a more complete example

Changes made in relevant directories are immmediately reflected in the website -- there is no need for `apachectl graceful` or similar:
* `/usr/local/vufind-8/iplc` -- IPLC-specific code
* `/usr/local/vufind-8/module/ReShare` (symbolic link) -- the ReShare module 
* `/usr/local/vufind-8/themes/BorrowDirect` -- visual identity, inherits from the `bootstrap3` theme

When trying to figure out what needs to be overridden the simplest way to get information is just to edit the VuFind module and temporarily edit files putting in `die(var_dump($somevar))` get the relevant variables on screen.


### Patron identity

When a patron logs in, part of the process is a Where Are You From? dropdown, so the patron identity contsists of an [institution, user-within-institition] pair. We will need to inspect the logged-in-patron data structure to see where the relevant information is stored, but there is talk of a `college` field that might point to the institition. This is needed to determine the ReShare instance that we need to communicate with.

(The list of patron profiles can be found in [**Settings** --> **Resource Sharing** --> **Host LMS patron profiles**](https://east.reshare-dev.indexdata.com/settings/rs/lmsprofiles), but we probably don't need the new code to know about these, only about what ReShare returns from its patron-validation call.)


### Contacting ReShare

The big missing piece is how we contact ReShare: how we find the right WSAPI URL, and how we authenticate. Somewhere we need to store a map from `college` strings to [okapiUrl, userName, passWord] triples.

This information is conveniently already configured into [the OpenURL listener](https://github.com/openlibraryenvironment/listener-openurl/blob/e07fa771910855018d2d4e32d4589fea12aaf465/config/openurl.json#L11-L24). We could make it available to VuFind by checking out a copy of [the relevant configuration in our Kubernetes configuration](https://github.com/indexdata/eks-folio-us-east-1-1/blob/master/iplc-prod/listener-openurl/listener-configmap.yaml), although checking out that whole repository feels like overkill.

The JSON config is embedded in a YAML file. To deal with it properly, it would be necessary to run a YAML parser. But in reality, `sed -n '/openurl.json/,$p' | sed 1d` will do it. We can make a git post-pull hook that extracts the `config.json` from the YAML file, so it's available in a form that the PHP plugin can easily make use of.


### Accessing the patron-validation service

`GET ${OKAPI_URL}/rs/patron/${PATRON_ID}/canCreateRequest`

As noted in
[PR-1077](https://openlibraryfoundation.atlassian.net/browse/PR-1077).
Chas has provided
[an example script](https://github.com/openlibraryenvironment/mod-rs/blob/master/okapi-scripts/canRequest.sh)
demonstrating how to use this service.



## Notes

1. Or she, here and elsewhere[2].
2. Which sounds like a bad Beatles song.



## TODO

```
I thought /usr/local/vufind-8/iplc was a symlink to a separate git module, but it doesn't seem to be.


Ian Hardy
:palm_tree:  18:22
only the theme and module:
New
18:22
lrwxrwxrwx 1 root root 36 Sep 22 15:53 /usr/local/vufind-8/themes/BorrowDirect -> /opt/vufind-iplc/themes/BorrowDirect
lrwxrwxrwx 1 root root 28 Sep 22 15:52 /usr/local/vufind-8/module/iplc -> /opt/vufind-iplc/module/iplc
```
