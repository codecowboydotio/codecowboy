+++
title = "Mermaid and Hugo - a tale of submodules and subtrees"
date = "2022-04-03"
aliases = ["git"]
tags = ["git"]
categories = ["software", "dev"]
[ author ]
  name = "codecowboy.io"
+++

# Intro

I have decided to investigate the difference and strengths and weaknesses of using subtrees versus submodules in git.

Both have pros and cons, and in this post I explore them both.

## What prompted this?

Recently I decided to update my hugo instance to use mermaid. [Mermaid](https://mermaid-js.github.io/mermaid/#/) is a library that allows you to embed diagrams in markdown. This means that I can create diagrams simply by using markdown.

My hugo theme, was installed as a git submodule. What's more, I didn't own the theme, or have commit access to it. This meant that I could not add a new shortcode for mermaid into my theme easily.

I ended up removing the submodule and just cloning the theme directly into my codebase.

This made me think about the **best way to manage dependencies** between different projects. 

# Project Dependancies

Often, there are times when you need to manage dependencies between different projects. 

- There is no **easy** way to do this. 
- There is no **right** way to do this. 

# Monorepo

There is a pattern loosely called **monorepo** where you store everything that your project needs, including all of the dependencies in a single repository.

It looks something like this:

{{<mermaid align="left">}}
graph LR;
    A[team A/frontend] --> B[monorepo]
    C[team B/service 1] --> B
    D[team C/service 2] --> B
    E[team D/service 3] --> B
{{< /mermaid >}}

## Advantages

Using a single repository for code is simpler. Everything is in one place. There is no need to use an external package manager to manage dependencies within your project. This has huge advantages as there is no need to learn external tooling like maven.

There is no need to synchronise different projects for a particular release. This has huge advantages. Imagine the following scenario.

I have a project with multiple teams committing to it, each separate team "owns" a different folder structure - much like the diagram above.

The commits would look something like this:

{{<mermaid align="left">}}
stateDiagram-v2
    main --> frontend
    main --> service1
    main --> service2
    service2 --> feature_branch_A
    feature_branch_A --> main
    service1 --> feature_branch_A
    feature_branch_A --> release_A
    frontend --> feature_branch_B
    feature_branch_B --> main
    feature_branch_B --> release_A
    release_A --> main
    release_A --> [*]
    [*] --> main
{{< /mermaid >}}

While this is a somewhat contrived example, the idea to remember here is that each branch operates independently and is checked back into main at some point, and at some point a release is pulled from everything.

In a monorepo, this is a lot easier to manage, because you're just essentially using different tags.

## Disadvantages

Monorepos have some disadvantages - the main one (no pun intended) is size. Monorepo's can grow to be **very** large. If you're using a CI tool, this can increase the time taken to run certain tests. While it's true that certain types of tests, like unit tests, will more than likely be run locally, other types of tests, like smoke tests, or integration tests can be impacted by repospitory size. If you think of a typical CI scenario, I will make a commit, tag the code with a tag that indicates a smoke test needs to be run. Typically this will spin up a runner on my CI tool of choice, and the first thing I'll do is clone my repo. In the case of a smoke test, I may only want to test a specific service, or a very small piece of functionality, but **I still need to clone the entire repository** if I am using a monorepo.

This may not sound like a big deal, but imagine the repository is 5 or 6 gigabytes in size and takes two minutes to pull down. This means that **two minutes of my pipeline is doing nothing.** This may not sound like a lot, but in terms of **developer velocity** and **productivity** it can add up over the course of a day.

# External tooling

One way of managing dependencies in code is to use external tooling. There are a lot of different types of tooling that you can use.

A lot of different languages have different ways of managing dependcies. 

Read that again - and you'll note that **these are language dependent.** What if I'm using multiple languages as part of my project? This is an almost certainty if you're not writing everything in say javascript.

- Maven (or Gradle) if you use Java (yes people still use it)
- Npm for node apps
- Bower if you use Javascript 
- Pip and requirements.txt if you use Python
- RubyGems if you use Ruby

...and the list goes on.

In a large project that uses multiple languages, or even the same language,  you may have different versions of dependency software. 

I recently had this:

```Bash
npm WARN read-shrinkwrap This version of npm is compatible with lockfileVersion@1, but package-lock.json was generated for lockfileVersion@2. I'll try to do my best with it!
```

This was simply differences in the CI tool and my local workstation where I wrote the code. Easy enough to fix, but it does illustrate the point that even using external dependency (in this case npm) can introduce some weirdness.

# Submodules and Sub Trees

As you read the next section please keep this in mind. 

{{< notice tip >}}
A sub module is a link to a repository.

A sub tree is a copy of a repository.
{{< /notice >}}

## Sub Modules

Sub modules are part of git. A git submodule is a record within a git repository that points to a specific commit in another external repository.

This seems that it is a good way of managing a dependency in code, but it has a lot of disadvantages. 

- sub modules track specific commits
- sub modules don't track REFs or branches
- sub modules are not automatically updated when the external repository is updated

Some examples:

I can clone my Star Wars API repository.

```Bash
[root@fedora test]# git clone http://github.com/codecowboydotio/swapi-json-server
Cloning into 'swapi-json-server'...
warning: redirecting to https://github.com/codecowboydotio/swapi-json-server/
remote: Enumerating objects: 669, done.
remote: Counting objects: 100% (390/390), done.
remote: Compressing objects: 100% (196/196), done.
remote: Total 669 (delta 181), reused 290 (delta 87), pack-reused 279
Receiving objects: 100% (669/669), 7.83 MiB | 6.29 MiB/s, done.
Resolving deltas: 100% (249/249), done.
```

Then I can add another repo as a submodule.

```Bash
[root@fedora swapi-json-server]# git submodule add http://github.com/codecowboydotio/dockerfiles
Cloning into '/tmp/test/swapi-json-server/dockerfiles'...
warning: redirecting to https://github.com/codecowboydotio/dockerfiles/
remote: Enumerating objects: 89, done.
remote: Counting objects: 100% (33/33), done.
remote: Compressing objects: 100% (28/28), done.
remote: Total 89 (delta 9), reused 27 (delta 4), pack-reused 56
Receiving objects: 100% (89/89), 659.54 KiB | 3.97 MiB/s, done.
Resolving deltas: 100% (23/23), done.
```

When I look at the metadata file in both the current submodule directory and in the parent I see the following.

```Bash
[root@fedora swapi-json-server]# more .gitmodules
[submodule "dockerfiles"]
        path = dockerfiles
        url = http://github.com/codecowboydotio/dockerfiles
```

In my original repository that I cloned, there is a **.gitmodules** file that contains details about the submodules, their local location on the filesystem and the URL of the other project.

We can see that my main repository - my SWAPI API - is aware of the new submodule also, but only in that the .gitmodules and the new directory that I cloned are seen as new.

```Bash
[root@fedora swapi-json-server]# git st
On branch master
Your branch is up to date with 'origin/master'.

Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
        new file:   .gitmodules
        new file:   dockerfiles
```

I can see when I add an awesome feature to my submodule that it is tracked.

```Bash
[root@fedora dockerfiles]# touch some_awesome_feature
[root@fedora dockerfiles]# git st
On branch master
Your branch is up to date with 'origin/master'.

Untracked files:
  (use "git add <file>..." to include in what will be committed)
        some_awesome_feature

nothing added to commit but untracked files present (use "git add" to track)
```

Let's add my file - remember we are still inside the submodule at this point.

```Bash
[root@fedora dockerfiles]# git add some_awesome_feature
[root@fedora dockerfiles]# git st
On branch master
Your branch is up to date with 'origin/master'.

Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
        new file:   some_awesome_feature
```

When I transition back to my main project, I see the following:

```Bash
[root@fedora swapi-json-server]# git st
On branch master
Your branch is up to date with 'origin/master'.

Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
  (commit or discard the untracked or modified content in submodules)
        modified:   dockerfiles (modified content)
```

You can see that the main project knows that there has been a change, but that the submodule is responsible for actually tracking that change. 

Another way of showing this is via git diff.

```Bash
[root@fedora swapi-json-server]# git diff
diff --git a/dockerfiles b/dockerfiles
--- a/dockerfiles
+++ b/dockerfiles
@@ -1 +1 @@
-Subproject commit 87cae9fb73adadcdebafab6c4923fe9d9efd6920
+Subproject commit 87cae9fb73adadcdebafab6c4923fe9d9efd6920-dirty
```

You can see that I've added stuff, I've deleted stuff, but the main project points to the sub project for the detail.

### Buyer Beware

This can be a handy way of including a specific version of another project inside your own project. **BUYER BEWARE** though that it can become very challenging to manage local changes to something that it a sub module. This is especially the case where you are including a repository that you do not necessarily have commit access to.

This is the case if you are using a repository, like a HUGO theme, that you add as a submodule. I don't have write access to the theme to be able to commit changes to it. This makes modifying content in the submodule and then checking in my code to my own repository difficult.  More on this later.

## Sub Trees

Sub trees are another feature of git. They allow you to nest another repository inside a sub directory. 

{{< notice tip >}}
You may need to install git subtree as it's not installed by default.
{{< /notice >}}

```Bash
dnf -y install git-subtree
```

On fedora the command above performs the installation for me.


Let's use the same example of my Star Wars API again.

```Bash
[root@fedora test]# git clone http://github.com/codecowboydotio/swapi-json-server
Cloning into 'swapi-json-server'...
warning: redirecting to https://github.com/codecowboydotio/swapi-json-server/
remote: Enumerating objects: 669, done.
remote: Counting objects: 100% (390/390), done.
remote: Compressing objects: 100% (196/196), done.
remote: Total 669 (delta 181), reused 290 (delta 87), pack-reused 279
Receiving objects: 100% (669/669), 7.83 MiB | 4.30 MiB/s, done.
Resolving deltas: 100% (249/249), done.
```

When I change into my new directory, I can add another project as a sub tree.

```Bash
git subtree add --prefix subtree https://github.com/codecowboydotio/libp2p-experiements main --squash
```

This pulls down the second project (my libp2p experiments) and I can then see the following:

```Bash
[root@fedora swapi-json-server]# git subtree add --prefix subtree https://github.com/codecowboydotio/libp2p-experiements main --squash
git fetch https://github.com/codecowboydotio/libp2p-experiements main
remote: Enumerating objects: 133, done.
remote: Counting objects: 100% (133/133), done.
remote: Compressing objects: 100% (81/81), done.
remote: Total 133 (delta 90), reused 90 (delta 50), pack-reused 0
Receiving objects: 100% (133/133), 25.19 KiB | 1.48 MiB/s, done.
Resolving deltas: 100% (90/90), done.
From https://github.com/codecowboydotio/libp2p-experiements
 * branch            main       -> FETCH_HEAD
Added dir 'subtree'
```

I have now successfully added my second project as a completely independent sub tree. This means that both projects can be updated and maintained separately. 

If I want to update the sub tree at some point in the future because it has changed and there is new functionality that I want to get, I can do the following:

```Bash
git subtree pull --prefix subtree https://github.com/codecowboydotio/libp2p-experiements main --squash
```

This looks like this:
```Bash
[root@fedora swapi-json-server]# git subtree pull --prefix subtree https://github.com/codecowboydotio/libp2p-experiements main --squash
From https://github.com/codecowboydotio/libp2p-experiements
 * branch            main       -> FETCH_HEAD
Subtree is already at commit d75b61d41646ed05675a72e6e7cac588ed3ac770.
```

Note that the command is run from the root of the main project. If you are in another directory you will get the following message.

```Bash
[root@fedora subtree]# git subtree pull --prefix subtree https://github.com/codecowboydotio/libp2p-experiements main --squash
You need to run this command from the toplevel of the working tree.
```

## Squash

You will note that I have used the **squash** option as part of my commands when using sub tree. All this does is bundle up historical commits into a single commit. This makes managing things like sub trees a lot easier.

# Conclusion

Submodules and sub trees both have good points and bad points. Below are some of my thoughts on both of these.

## Submodule Thoughts

Submodules have a bad wrap on the internet. People **really** don't like them. There are a lot of blog posts that say "don't use them". Even though they didn't work for me, I think that they have a place. If you are using multiple repositories and you own them all. A submodule is pushed when you push code to your main repository. That is, **everything** submodule and all is pushed. 

This has a lot of benefits if you need to keep code separate, but also need to update the submodule project.

## Sub Tree Thoughts

Sub trees are good - they're close to submodules in some ways but are slightly more independent for my purposes. For example, when I push my main repository, my sub tree is not pushed. This is really good if you are a **consumer** rather than a committer of a secondary project in your repository, and you only want to receive updates.

For example, If I have another team providing me templates that I consume for a specific purpose, like security templates, or pipelines that I can pull down and run - sub trees are a fantastic way to go.

Look out for the next blog where I look at this example and work through using sub trees for security templates from an external team, and what that looks like.
