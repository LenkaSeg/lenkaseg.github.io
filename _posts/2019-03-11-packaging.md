---
layout: post
title: My notes about packaging for Fedora
comments: true
---

These personal notes serve for me, when trying not to get lost in the amount of 
guides and links there 
are and in effort of keeping some outlines and avoid asking the same questions more times.
It is very probably full of mistakes and inaccuracies. Please don't use this as a guide and 
if you find a mistake, let me know about it in the comments.

## How to make a Fedora package

When a program has enough functionality, maybe it's time to package it.
Cranc has 4 working commands. It can list PRs and issues, merge PR and create PR.
First thing to do is to create a Vagrantfile with Fedora 29. I used an example 
[Fedora Vagrantfile](https://fedoraproject.org/wiki/Vagrant) and modified it with the latest
version of Fedora and also made some adjustments.

`vagrant up`

`vagrant ssh`

## Spec file

The next thing to do is writing a spec file. I checked how a spec file should look like in 
a popular Fedora package - 
[python-requests](https://github.com/OpenMOOC/moocng/blob/master/rpmbuild/SPECS/python-requests.spec).
I was following several Fedora guides. First one was providing general info about [Packaging
and maintaining Python applications](https://fedoraproject.org/wiki/SIGs/Python#Areas_of_contribution).
There are useful links to 
[Fedora packaging guidelines](https://docs.fedoraproject.org/en-US/packaging-guidelines/)
and
[Python packaging guidelines](https://docs.fedoraproject.org/en-US/packaging-guidelines/Python/) 
and it is strongly recommended to get familiar with them.
It's also necessary to add the license. I decided to use the GPLv3 license.

## Tagging

Then it is time to create a tarball. First I tagged the last commit of `cranc` with the version tag:

`git tag -a v.0.1 -m "[tag message]"`

To see all the tags:

`git tag`

To see info about specific tag:

`git show v.0.1`

Tagged commits are stored as full objects in git database and are checksummed. About [tagging](https://git-scm.com/book/en/v2/Git-Basics-Tagging)

## Repo

Then I made a new repo on pagure called `python-cranc`.
On local repo I placed the `cranc.spec` file and `Vagrantfile`, added, commited and pushed to 
remote python-cranc repo.

## Koji install 

I followed the 
[Join the package collection maintainers](https://fedoraproject.org/wiki/Join_the_package_collection_maintainers)
guide. I created the [Bugzilla account](https://bugzilla.redhat.com/), Fedora account I already had.

I configured git like this:

`git config --global user.name "Lenka Seg"`
`git config --global user.email lenkaseg@email.com`

The next step was installing [Koji](https://fedoraproject.org/wiki/Using_the_Koji_build_system#Installing_Koji)
which is the developer client tool to request package builds and get information about the 
buildsystem. Koji builds RPM packages.

`dnf install fedora-packager` 

`dnf install fedora-review`

I added myself to the mock group:

`usermod -a -G mock [myusername]`

I changed from the root user to my user name and run

`newgrp` and 

`id` to confirm that the mock group appears in the group list.

After that I continued with the Koji build:

`fedpkg build`

This starts a build request for the branch. Hopefully it builds succesfully.
With this I get the srpm file. 

There is a useful tool for package review called rpmlint. It checks 
if the spec file passes in all the MUST items and SHOULD items and leaves a report.

`yum install -y rpmlint`

To test the rpm file:

`rpmlint [PATH]/rpmbuild/SRPMS/[RPM FILE]`

When all the complaints are solved, it's time to upload a release.

## Upload the srpm file and file a bug in Bugzilla

From pagure.io I clicked on Releases button and Upload a new release. That means to upload the
src.rpm file. The file will appear in the Release folder.

Now it's time to go to Bugzilla and file a bug. I need urls of the spec file and SRPM file. 
It was my first package and I needed to find
a sponsor. After the reviews and some changes my package was finally approved. One very 
kind sponsor took care of me, so I could request a repo and a branch.

## Requesting repo and branch

I want to be a maintainer of my package, thus I have to request a new git repo.

(from the devel directory in vagrant)
`fedpkg request-repo cranc [bugnumber]`

Requesting a branch:

`fedpkg request branch f29` or f30 or rawhide (master)

Requests were reviewed and processed pretty quickly. Now I have access to commit and 
build the package.

## Import, commit and building a package

Checkout the module from the SCM:

fedpkg clone cranc

`cd cranc`

Now it's time to test the package with Koji scratch build.

First use [kerberos auth with Fedora](https://fedoraproject.org/wiki/Infrastructure/Kerberos):

`kinit <yourfasloginname>@FEDORAPROJECT.ORG`

Confirm:

`kinit hello`

Otherwise run just `kinit`.

After that run fedpkg to import the contents of the SRPM into the SCM:

`fedpkg import PATH_TO_SRPM`

example:

`fedpkg import ../rpmbuild/SRPMS/[cranc src.rpm]`

This will import the contents of the SRPM into the SCM.

Next step in the guide is to review changes, commit with the bug number:

`git commit -m "Initial import (#[bug number])"`

`git push`

`fedpkg build`

## Update in Bodhi

[Bodhi](https://fedoraproject.org/wiki/Bodhi) is Fedora update system, serves for 
pushing updates on branches.

`fedpkg update`

## Requesting another branch

Now I need to request new branch. I requested it again with
`fedpkg request-branch f30` from the devel/cranc in vagrant 
this will return a url to pagure.io/releng and open an issue with new branch request

and it was approved.

Now from the same diectory:
- go to master
- git pull 
it shows there is a new branch available:  `* [new branch]      f30        -> origin/f3`

`git branch -f f30 origin/f30` this command fetches the new branch from the origin, which is the
remote repo rpms/cranc which was created by fedora people.

Checkout that new branch:
`git checkout f30`
`ls`
it should contain the same stuff as in the remote:
https://src.fedoraproject.org/rpms/cranc/tree/f30 
(if there are some extra things, remove them)

## Scratch build

The guide recommends to test the package making the 
[scratch build](https://fedoraproject.org/wiki/Using_the_Koji_build_system#Scratch_Builds)
It's building a package against the buildroot without actually including it in the release.

`rpmbuild -bs cranc.spec`
`koji build --scratch rawhide cranc.srpm

More info in the link.

## Import

First run git status, add and commit the files. Usually it's `.gitignore`, `cranc.spec` and
`sources`.

Run `kinit` for logging in.

Time to import!
Run:

`fedpkg import PATH_TO_SRPM`

`fedpkg import /home/vagrant/rpmbuild/SRPMS/cranc-0.2.2-3.fc29.src.rpm`

It does not matter, that I want f30 and in the path there is f29.

## Build

### For rawhide (master)

`git status`

If there are some more changes (for example in .gitignore and sources) add them, commit, push:

`git add`

`git commit -m "commit [package review bug number]"`

`git push`

and build:

`fedpkg build`

### For updating branches

`fedpkg switch-branch BRANCH (for example f30)`

`git merge master`

`git push`

`fedpkg build`

If there is another branch, repeat "To switch to a branch", import and commit again.
Always rawhide (master) has to be build first.

If the build is succesful, there is this url:
"https://koji.fedoraproject.org/koji/taskinfo?taskID=33401469" 
where the build is nicely visible in koji.

## Update to Bodhi

Fedora update system. Always submit from branches, never rawhide (master).

`fedpkg update`

Here it will be moved to testing and waiting for reviews (karma points). I set karma to 1, for 
such an obscure package as cranc I think it's appropriate. If there are no reviews, apparently 
after 7 days the package is moved to stable.


