---
layout: post
title: Modifying expectations
comments: true
---

## Modifying expectations

This is our original project timeline as my mentor suggested when I was applying for the 
internship:

*1st half of Dec:

Investigate and document the current state of the tooling (what is available for 
GitHub/Gerrit/..), define the minimal set of operations to support (for example: 
list pull requests, checkout a certain pull-request) and choose the way to implement them.

2nd half of Dec:

Implement initial set of operations.

1st half of Jan:

Package it for Fedora

2nd half of Jan:

Document the tool, publish it and start the discussion with user community

1st half of Feb:

Add functionality (check ci-status of a PR, list approvals,.. ) and iterate on community feedback

2nd half of Feb:

Integrate with fedpkg tool

If time remains:

Stretch goal: extend the tool to support other Code-Review systems (GitLab, GitHub, Bitbucket..)*


The first goal probably could be considered as done. I checked how the command line tools look like
for GitHub, Gerrit or Kubernetes and we talked with Aleksandra about the structure and 
grammar of the commands. The set of operations is defined in the `commands.md` file in the 
docs.

In the meantime I started to implement the initial set of operations. First command was
`cranc get prs` followed by `cranc get issues`.
When I tried to implement `cranc merge pr` I ran into one problem. While the function was working
well when tested in pagure.io, I could not test it locally for the missing gitolite/ssh support.
Is spend more than a week trying to figure out how to run the gitolite. First I made it part of the 
worker container, but it was timing out. Then I tried to make it run in a separate container,
but in the middle of my efforts came the Devconf.cz conference in Brno. There I met Pingou personally
and he told me that all this was not necessary, because repos are stored locally in the `lcl` 
directory :)

This is how to clone a repo in local instance of Pagure:

In `lcl/repos/` 
`sudo mkdir clones`
`cd clones`
`sudo chown [user]:[user] -R [path]/pagure/lcl/`
'git clone ../repos/repos/test.git/`
`cd test`
`ls -l`

So now I am able to clone a repo from localhost, add a change, commit, 
add a remote, push and open a pull request in the upstream or on the fork.

 
## The function which creates new pull request.

This function was missing in Libpagure, so I wrote it.
First I needed to find a place in Pagure where the api pull requests are created,
trying something like this:

`git grep pull_request` in api directory.

In api/fork.py is a function called `api_pull_request_create`, which seems to be
what I was looking for. It takes 1 mandatory and 2 optional arguments.
There are four possible urls, depending on if fork or namespace are provided.
Creating the correct url is a job of `create_basic_url` in Libpagure.
New pull request needs 3 arguments (title, branch_to, branch_from) and 1 is optional
(initial_comment).

The idea is to find a suitable place in Libpagure and make a function, which 
communicates with this one in `fork.py`. I named the new function 
`create_pull_request` and placed it just above the function listing pull requests.

Of course, it is necessary to use a local copy of Libpagure, so it can be edited.
The function is creating the appropriate url (using the `create_basic_url` helper 
function and passing the payload to `_call_api` function, where the pull request
is supposed to get posted.

After that I created a function in `cranc.py` called `create_pr`, which is 
used by the command `cranc create pr`. The function takes four arguments, all of them
are mandatory:

* --title
* --branch_to
* --branh_from
* --repo

The command in all its beauty will look like this:

`cranc create pr --title [title] --branch_to [branch_to] --branch_from [branch_from]
--repo [repo]`

which is bit long, maybe in the future it would not be a bad idea to invent some shortcuts.

The function creates a Pagure instance in Libpagure with arguments `pagure_token`
and `pagure_repository`, and optional `fork_username`. To Libpagure's
`create_pull_request it passes the arguments title, branch_to, branch_from and repo.

However, it was not working. I kept getting this exception:

*libpagure.exceptions.APIError: Invalid or expired token. 
Please visit https://pagure.io/settings#api-keys to get or renew your API token.*

I made sure the api_token I was using is really the correct one.

## So I started debugging. 

The function which creates a new pull request is `def api_pull_request_create` in fork.py.
I set logging level to DEBUG and added logging message just in the beginning of the function 
with the priority INFO, so it would get executed. Then I ran docker instance of Pagure and
the `cranc create pr` with all the arguments with one testing commit I had prepared.
The message was not printed. It means that the program doesn't even get there.

I checked the decorators then. And one of the decorators was this one: 
`@api_login_required(acls=["pull_request_create"])`. 
Considering, that the error message
was about invalid token, this could probably be related.

It this point I had to learn about how decorators work, because I didn't have it very clear.
This [guide](https://gist.github.com/Zearin/2f40b7b9cfc51132851a) was very helful.

After investigating and writing the log messages all over the fork.py, __init__.py, utils.py and
models.py, I saw there was no intersection between the expected and provided ACLs for the 
user API token:

```
Token acls: set([u'create_project', u'commit_flag', u'update_watch_status', u'fork_project', u'modify_project'])

Requested acls: set([u'pull_request_create'])
```

The solution was to add the pull_request_create to the access control list (ACL). The right place 
to do that is in `pagure/default_config` file: USER_ACLS and CROSS_PROJECT_ACLS.

Now it's possible to open a pull request with the user token.

It's also necessary to check if the project token can open a pull request for the same project,
but not for a different one. Here popped another problem. While opening the pull request from 
the fork, it was opened in the fork and not in the upstream.

Also, while trying to open a pull request with the correct project token I was getting an 
Invalid/Expired token error. I added some logs into `pagure/api/fork.py: api_pull_request_create` 
to see what is happening and I realized that the project token id and parent id are not the 
same. I went to the docker container of posgresql to see what is wrong in there:

`docker exec -it dev_postgres_1 /bin/bash`

I saw in the `projects` table that there are several projects of the same name, with the same 
owner but different id. I deleted all those duplicated projects and the problem was solved. 
But a different one arised. Now the pull request was open, but on the fork, not on the upstream 
as I wanted to.

There is a piece of code in the ```pagure/api/fork.py: api_pull_request_create```:

```
parent = repo
if repo.parent:
    parent = repo.parent
```

which is responsible for finding a parent of a fork. Moving this upper, before checking if the
project token and parent project match:

```   
 if flask.g.token.project and parent != flask.g.token.project:
     raise pagure.exceptions.APIError(401, error_code=APIERROR.EINVALIDTOK)
````

seems to solve the problem. Before it was comparing the upstream project token with the 
fork and the IDs were different.

I made a pull request in Pagure.io and if I add the tests, maybe it will get merged. There will be 
four working commands of the CLI and we will be finally able to move to the next step 
of the project timeline, which is packaging!

Only with one month of delay :)
