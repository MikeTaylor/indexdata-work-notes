# ReShare work on VuFind

<!-- md2toc -l 2 README.md -->
* [Sources of information](#sources-of-information)
* [The task](#the-task)
* [Technical details](#technical-details)
    * [Login to server](#login-to-server)
* [Notes](#notes)



## Sources of information

The information in this document is mostly taken from a Slack conversation at
https://indexdata.slack.com/archives/DGCAAHEJ2/p1663859084271569

The relevant Jira issues are
[PR-1279](https://openlibraryfoundation.atlassian.net/browse/PR-1279)
and
[DEVOPS-1108](https://jira.indexdata.com/browse/DEVOPS-1108).



## The task

We need to ensure that a patron is allowed to make a request from VuFind only if he[1] is authorized to do so. We can only make this determination once we know who the user is. That can happen either because of a just-in-time login in order to place a request, or when logging in for some other reason.

When a patron autheticates onto VuFind, we can determine who he is by having ReShare look him up in the local library's NCIP service. When it responds, we will know whether or not the user in question is authorized to initiate ILL, and can allow or deny the request accordingly. We can also disable (or leave enabled) the user's *ILL* button for the rest of the session; and if we determined the user's identity from a non-JIT login, we need never show a bait-and-switch *ILL* button at all.

The authentication check here is just an optimization. ReShare already prevents an unauthorized user from placing a request; the present work is to avoid the aunauthorized user from thinking he gets to place a request, only to find out later that ReShare rejected it. (XXX Awaiting [confirmation](https://openlibraryfoundation.atlassian.net/browse/PR-1279?focusedCommentId=469770).)



## Technical details


### Loging in to the server

SSH in as mike@reshare-mp.folio-dev.indexdata.com (using public/private key authentication)


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


