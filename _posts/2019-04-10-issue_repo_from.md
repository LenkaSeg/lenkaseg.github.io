---
layout: post
title: The parent/fork repo check for allowed pull requests
comments: true
---

While trying to confirm a certain bug running the command for creating new PR from fork
to upstream:

`cranc create pr --title new_readme --repo carrot --branch_from readme --branch_to master`
I ran into this problem:

~~~
raise APIError(output["error"])
libpagure.exceptions.APIError: Pull-Request have been deactivated for this project
~~~

The issue was probably caused by not allowing to open a PR on the target repo from the
fork repo. (Pingou mentions that the current API does not allow to specify the 
`repo_from` and the PR is being open on the fork, which is not allowed by default).
The solution should be just to add the `repo_from` option to the API.

I wanted to test the behavior locally. For that I used the Pagure local intance which
 I'm running manually from a Fedora container and virtual environment with python3.6.

On the web interface I created the carrot repo, forked it and got an error while trying 
to clone the fork with ssh keys, even though it was possible to add the ssh key.

Nevermind, I tried the old proven way of cloning it manually into the locally stored
repos in pagure/lcl:

In `lcl/repos/`:
 
`sudo mkdir clones`

`cd clones`

`sudo chown myname:myname -R /home/myname/mypath/pagure/lcl/`

'git clone ../repos/carrot.git/`

`cd carrot`

`ls -l`

Here make some file, README.md for example and write something deeply meaningful in it.

`vim README.md`

`git checkout -b readme`

`git add README.md`

`git commit -m "motivational_quote"`

`git remote add origin /home/myname/pagure/lcl/repos/forks/foo/carrot.git`

`git push origin readme`

Yeah, but I got an error here: 

~~~
remote: Traceback (most recent call last):
remote:   File "hooks/pre-receive", line 31, in <module>
remote:     import pagure.lib
remote: ModuleNotFoundError: No module named 'pagure'
To /home/mylovelyname/pagure/lcl/repos/clones/../carrot.git/
 ! [remote rejected] readme -> readme (pre-receive hook declined)
error: failed to push some refs to '/home/mylovelyname/pagure/lcl/repos/clones/../carrot.git/'
~~~

What could go possibly wrong? What is with this pre-recieve hook and why the module can't
be found?

I was running the pagure instance in a virtual environment with pagure3.6, as was 
recommended in the pagure README. But this hook was running out of this environment.

I deactivated the venv. Checked the python version:

`python`

answer: 3.6 That looks good.

I checked the pre-recieve hook:

`vim ../../carrot.git/hooks/pre-receive`

And here I can see this: `#!/usr/bin/env python3`. But what python version is the python3?

`python3 --version`

answer: python 3.7.2. That should not be. There is also this command to check where
the link points:

`readlink /usr/bin/python3`

answer: python3.7

That was the problem. The python3 links to a version of python which is not recommended.
To solve it and make the python3 point to python3.6 remove this link:

`sudo rm /usr/bin/python3`

And run ln -s to create the link between python3 and the version of python I really want:

`sudo ln -s /usr/bin/python3.6 /usr/bin/python3`

Note to self: Always use the -s option (soft or symbolic), otherwise the link will be hard link.
I just want to be sure the link is only a way to access the file, but they do not mess with 
each other. More info in the 
[link](https://www.linuxnix.com/soft-link-vs-hard-link-in-linuxnix/)

Now, when I run:

`/usr/bin/python3 --version`

I get: Python 3.6.8 and that's what I wanted. Let's try if the branch gets pushed:

`git push origin readme`

and it did. Ok, branch is pushed to the fork. Now it's time to use cranc to open a pull
request on the upstream.

Run [cranc](https://pagure.io/cranc):
    Install and create the virtual environment:
    
    `pip3 install virtualenv`
    
    `virtualenv ~/venvs/cranc-env`

    `source ~/venvs/cranc-env/bin`

    Retrieve the sources:

    `git clone https://pagure.io.cranc.git`

    `cd cranc`

    Install the dependencies:

    `pip install -r requirements.txt`

    Run the setup file:

    `python setup.py develop`

    Run the command for opening a pull request from a fork:

    `cranc create pr --title "new readme" --repo carrot --branch_from readme branch_to master`

And I run into error:

~~~
    PAGURE.log_debug(True)
AttributeError: 'Pagure' object has no attribute 'log_debug'
~~~

Looks like an error in Libpagure, where the log_debug is supposed to be, 
but it isn't found. I checked if the libpagure was correctly installed with 
the rest of requirements and if it is in the correct place in python3.6 site-packages - 
and everything seemed fine. The problem finally was that the log_debug was a recently 
added function, but the updated version of libpagure was not released.

The solution is to uninstall the libpagure and make a change in `requirements.txt`. 
Instead of libpagure write this:

`-e git+https://pagure.io/libpagure.git#egg=libpagure`

Source [here](https://stackoverflow.com/questions/16584552/how-to-state-in-requirements-txt-a-direct-github-source).
This link points to a direct source libpagure with the last commited changes. Like this
the libpagure will be always on the latest version, even without the necessity of release.
I also asked Cverna, the libpagure maintainer, if he could release the new version of 
libpagure and he did almost immediately. Anyways, for the future possible changes it will
be better to leave this link in requirements.

I installed the requirements.txt with pip again and ran the cranc command for opening
a pull request.
Now it works and I'm able to reproduce locally the error from the beginning.

In `pagure/api/__init__.py` I can see that the error corresponds with 
`EPULLREQUESTSDISABLED` error message. In the `pagure/api/fork.py` in the 
`def api_pull_request_create` there is this exception once, but there are some decorators
which have to be checked too. To find the exact point where the exception pops I put 
logs everywhere, after every piece of code. And I found out, that the line it fails on 
the line 1343: `_check_pull_request(repo)`. This function is defined in `utils.py`.

And I see that the pull requests are not allowed for this repo, that's why the exception.
The repo is probably not the repo I want. It's probably pointing at the fork.

Just a couple of lines below there is a piece of code which tries to find a parent of 
a repo:

~~~
parent = repo
if repo.parent:
    parent = repo.parent
~~~

I just move it a bit upper, after this line:

~~~
repo = _get_repo(repo, username, namespace)
~~~

and I change the repo, which is being checked for if the pull requests are allowed, to
parent:

~~~
_check_pull_request(parent)
~~~

Now the entire part looks like this:

~~~
parent = repo
if repo.parent:
    parent = repo.parent

_check_pull_request(parent)
_check_token(repo)
~~~

That is all and the pull requests are now opened on the parent repo, and the parent repo
is checked to have the pull requests allowed.
