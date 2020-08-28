---
title: Organizing your Development Directory
date: 2019-04-15T19:05:00.000+00:00
categories:
- development
tags:
- development
comments: true
draft: true

---
As an engineer, I like to think that I help fix problems. That's what I've tried to do most of
my life and career and I love doing so to this day. It struck me, though, that there was one
problem which has followed me around for years without due attention: the state of my development
directories.

That's not to say that they are disorganized, I've spent hours deliberating over the best way
to arrange them such that I can always find what I need, yet I often end up having to resort to
some dark incantation involving `find` to locate the project I was *certain* sat under my `Work`
folder.

No more, I've drawn the line and decided that if I can't fix the problem, automation damn well
better be able to!

I'd like to introduce you to my new, standardized (and automated), development directory structure
and the tooling I use to maintain it. With any luck, you'll find it useful and it will enable you
to save time, avoid code duplication and more easily transition between machines.

<!--more-->

## Background
I've been writing code and building systems for well over a decade at this stage and have studiously
hung onto every piece of code I've ever written (that wasn't covered by an IP clause in a contract).
That has left me with multiple millions of lines of code, hundreds of projects and a directory
structure roughly resembling the well groomed beard of our lord and saviour C'thulu. That is to say,
it scared small children and required so shortage of black magic to comprehend.

With each new job I've started, I've trialed different ways of organizing my dev directory and
some have been more successful than others. I'm sure some of these will sound familiar to those of
you who have walked a similar road, while some may cause you to grimace and flinch. You may want
to put down your coffee, fair warning...

### Unorganized
Oh sure, we've all been there, a single folder holding every single one of our projects. It works out
great at first but 2 years in and it looks like a child's bedroom after the hand-painting party with
open candy-bar.

{{< filetree >}}
- dev
    - <i class="icon-github"></i> farmtrack-ui
    - <i class="icon-github"></i> farmtrack-api
    - <i class="icon-github"></i> farmtrack-server
    - <i class="icon-github"></i> fieldsense-app
    - <i class="icon-github"></i> libfieldsense
{{</ filetree >}}

This structure basically exemplified the problem of trying to find things. Naming becomes extremely
important if for no reason other than to find related projects, grouping is almost impossible, finding
a project whose name you forgot becomes an exercise in patience and perhaps its only redeeming quality
is that it doesn't (strictly) require you to learn how to use `find`. I'd notch this experiment down as
a "Junior Wizard's First Foray into Filesystems".

### Organized by Project
Perhaps my first layout, this involved sorting my dev directory by the overarching "business" project
that the code formed a part of. This stemmed from my time in academia where organizing artifacts and
resources by the module it related to worked quite well.

{{< filetree >}}
- dev
   - farmtrack
       - <i class="icon-github"></i> farmtrack-ui
       - <i class="icon-github"></i> farmtrack-api
       - <i class="icon-github"></i> farmtrack-server
   - fieldsense
       - <i class="icon-github"></i> fieldsense-app
       - <i class="icon-github"></i> libfieldsense
{{</ filetree >}}

On the face of it, this looks like a pretty reasonable approach and indeed, it does work. In fact,
for people who do not heavily invest in source control or who are extremely project-centric, this
does offer a very nice way to quickly find everything related to a single project.

My problem came in the moment I wanted to start sharing code between projects... Do I have a copy
of the code in the folder for each project and manually keep them in sync, or do I have a separate
project for the library now? All of this led to inconsistencies and the aforementioned trouble finding
the project I was actually looking for.

Back to the drawing board...

### Organized by Language
Try not to laugh at this one, but at one stage this actually made sense. As someone who has worked with
a rather large number of languages, this made for a nice way to mentally "hook" the code I had written
to where it would be located on my filesystem.

{{< filetree >}}
- dev
   - apiblueprint
       - <i class="icon-github"></i> farmtrack-api
   - cpp
       - <i class="icon-github"></i> fieldsense-app
       - <i class="icon-github"></i> libfieldsense
   - csharp
       - <i class="icon-github"></i> antennaportal
   - go
       - <i class="icon-github"></i> sentry-go
   - typescript
       - <i class="icon-github"></i> farmtrack-ui
       - <i class="icon-github"></i> farmtrack-server
{{</ filetree >}}

Again, for a definition of "worked" this was pretty good. By keeping projects with shared languages
in the same directory tree I could avoid duplicating folders for shared libraries in most cases
and it was often easy to find a specific project I was looking for.

Unfortunately I have a habit of rewriting projects onto better frameworks and platforms as a means
of educating myself on their intricacies. This meant that one branch of a repository may be in one
language while another branch may be in another. Pair this with my rapid prototyping approach in
which "fast" languages are used for an MVP and more mature platforms are exercised once I move to
production and this whole system starts to fall apart.

### Organized by Company
My next plan was to split things up by "company" - have all code which fell under a certain intellectual
property domain housed within a single root directory.

{{< filetree >}}
- dev
   - EMSS
       - <i class="icon-github"></i> farmtrack-api
       - <i class="icon-github"></i> farmtrack-ui
       - <i class="icon-github"></i> farmtrack-server
   - SierraSoftworks
       - <i class="icon-github"></i> iridium
       - <i class="icon-github"></i> sentry-go
   - Personal
       - <i class="icon-github"></i> cv
{{</ filetree >}}

Since code doesn't naturally transition from one IP domain to another in most cases, this avoided a number
of the headaches I had encountered with the other layouts. This was probably the closest to a "perfect"
I managed to find before switching to my current layout and, in fact, automating it was the catalyst for
the way I organize things today.

## My Current Solution
Based on the idea of having projects separated by organizational boundaries and optimizing the structure
to better support automation, my current layout looks similar to the way Go manages your `$GOPATH/src`
directory.

{{< filetree >}}
- dev
  - github.com
     - spartan563
         - <i class="icon-github"></i> cv
     - sierrasoftworks
         - <i class="icon-github"></i> bender
         - <i class="icon-github"></i> blog
         - <i class="icon-github"></i> git-tool
         - <i class="icon-github"></i> iridium
{{< /filetree >}}

The beauty of this layout is, primarily, consistency. Projects land in the same directories on every
system you work with, they can easily be mapped to their project web pages, can easily have their
directory names derived, are easy to locate (or, at least, as easy as your GitHub is to navigate)
and avoid the issue of duplication by having a unique ID form their path.

This layout is also one that is instantly familiar to anybody who has used `Go` and is one that
has been proven to be easily parsable by automation tools. This enabled me to build [Git-Tool][],
a set of `PowerShell` commands which automate the management of this directory structure.

## Introducing Git-Tool

Git-Tool is a PowerShell module I've developed to assist with managing this particular repo structure.
It provides commands for creating new repositories, fetching existing ones and quickly opening local
copies by introducing native tab-completion.

For example, getting the latest version of a repo could be done with `Get-Repo sierrasoftworks/git-tool`,
while a new repo might be created by running `New-Repo spartan563/iot-test`. Finally, when opening a
shell, I need only type `Open-Repo sierrasoftworks/git-tool` to have my shell's working directory
changed to the location of that repo on my machine. Clean, simple, consistent and predictable; regardless
of which machine I'm on.

Git-Tool also supports services other than [GitHub][], including [GitLab][], [BitBucket][] and [Azure DevOps][].

### Installing Git-Tool
Git-Tool is available in the [PowerShell Gallery](https://www.powershellgallery.com/packages/Git-Tool) and
can be installed using `Install-Module Git-Tool` from any administrative PowerShell prompt.

You'll then want to configure your `$PROFILE` to ensure that `Git-Tool` is imported when your shell starts
and configured with the correct dev directory automatically. You can do this by opening up Notepad (or your
favourite editor) and pointing it at the `$PROFILE.CurrentUserAllHosts` configuration file.

{{< codeblock "Installing Git-Tool" >}}
{{< codeblock-tab "powershell" "Profile File" >}}
# notepad.exe $PROFILE.CurrentUserAllHosts

Import-Module Git-Tool

$GitTool.Directory = "C:\dev"
{{< /codeblock-tab >}}
{{< codeblock-tab "powershell" "Installation" >}}
Install-Module Git-Tool

notepad.exe $PROFILE.CurrentUserAllHosts
{{< /codeblock-tab >}}
{{< /codeblock >}}

### Creating a new repo
The process to create a new repo is incredibly straightforward, you simply need to call the `New-Repo` command
with the full name of the repo you wish to create.


{{< codeblock "Creating a Repo" >}}
{{< codeblock-tab "powershell" "Azure DevOps" >}}
New-Repo -Service dev.azure.com sierrasoftworks/opensource/bender

<# --------------------------------------------------------------------------------------------
Creating https://dev.azure.com/sierrasoftworks/opensource/_git/bender
 - Git URL:   git@ssh.dev.azure.com:v3/sierrasoftworks/opensource/bender
 - Target:    C:\dev\dev.azure.com\sierrasoftworks\opensource\bender

Running git init
Running git remote add origin
Adding .gitignore file

Name                           Value
----                           -----
Service                        dev.azure.com
Exists                         False
Namespace                      sierrasoftworks/opensource
Name                           bender
Repo                           sierrasoftworks/opensource/bender
Path                           C:\dev\dev.azure.com\sierrasoftworks\opensource\bender
GitURL                         git@ssh.dev.azure.com:v3/sierrasoftworks/opensource/bender
WebURL                         https://dev.azure.com/sierrasoftworks/opensource/_git/bender
-------------------------------------------------------------------------------------------- #>
{{< /codeblock-tab >}}
{{< codeblock-tab "powershell" "GitHub" >}}
New-Repo sierrasoftworks/git-tool -Open -GitIgnore csharp

<# -----------------------------------------------------------
Creating https://github.com/SierraSoftworks/demo
 - Git URL:   git@github.com:SierraSoftworks/demo.git
 - Target:    C:\dev\github.com\SierraSoftworks\demo

Running git init
Running git remote add origin
Adding .gitignore file

Name              Value
----              -----
Service           github.com
Exists            False
Namespace         SierraSoftworks
Name              demo
Repo              SierraSoftworks/demo
Path              C:\dev\github.com\SierraSoftworks\demo
GitURL            git@github.com:SierraSoftworks/demo.git
WebURL            https://github.com/SierraSoftworks/demo
----------------------------------------------------------- #>
{{< /codeblock-tab >}}
{{< /codeblock >}}

### Fetching an existing repo
Similarly, when you wish to fetch an existing repo, you simply call the `Get-Repo` command
with the full name of the repo. If you've got it locally already, tab completion will help
you fill in the name, otherwise it'll probably get you close by suggesting other similar
names to modify.

Of course, if you wish to also open up the repo once it has been fetched, you can pass the
`-Open` flag.

{{< codeblock "Fetching a Repo" >}}
{{< codeblock-tab "powershell" "Azure DevOps" >}}
Get-Repo -Service dev.azure.com sierrasoftworks/opensource/bender

<# --------------------------------------------------------------------------------------------
Synchronizing https://dev.azure.com/sierrasoftworks/opensource/_git/bender
 - Git URL:   git@ssh.dev.azure.com:v3/sierrasoftworks/opensource/bender
 - Target:    C:\dev\dev.azure.com\sierrasoftworks\opensource\bender
Running git pull
Already up to date.

Name                           Value
----                           -----
Service                        dev.azure.com
Exists                         False
Namespace                      sierrasoftworks/opensource
Name                           bender
Repo                           sierrasoftworks/opensource/bender
Path                           C:\dev\dev.azure.com\sierrasoftworks\opensource\bender
GitURL                         git@ssh.dev.azure.com:v3/sierrasoftworks/opensource/bender
WebURL                         https://dev.azure.com/sierrasoftworks/opensource/_git/bender
-------------------------------------------------------------------------------------------- #>
{{< /codeblock-tab >}}
{{< codeblock-tab "powershell" "GitHub" >}}
Get-Repo sierrasoftworks/git-tool -Open

<# -----------------------------------------------------------
Synchronizing https://github.com/SierraSoftworks/git-tool
 - Git URL:   git@github.com:SierraSoftworks/git-tool.git
 - Target:    C:\dev\github.com\SierraSoftworks\git-tool
Running git pull
Already up to date.

Name             Value
----             -----
Service          github.com
Exists           True
Namespace        SierraSoftworks
Name             git-tool
Repo             SierraSoftworks/git-tool
Path             C:\dev\github.com\SierraSoftworks\git-tool
GitURL           git@github.com:SierraSoftworks/git-tool.git
WebURL           https://github.com/SierraSoftworks/git-tool
----------------------------------------------------------- #>
{{< /codeblock-tab >}}
{{< /codeblock >}}

### Opening a local repo
Finally, when you want to open a repo which you have cloned or created, you'll simply 

{{< codeblock "Opening a Repo" >}}
{{< codeblock-tab "powershell" "Azure DevOps" >}}
Open-Repo -Service dev.azure.com sierrasoftworks/opensource/bender

<# --------------------------------------------------------------------------------------------
Name                           Value
----                           -----
Service                        dev.azure.com
Exists                         False
Namespace                      sierrasoftworks/opensource
Name                           bender
Repo                           sierrasoftworks/opensource/bender
Path                           C:\dev\dev.azure.com\sierrasoftworks\opensource\bender
GitURL                         git@ssh.dev.azure.com:v3/sierrasoftworks/opensource/bender
WebURL                         https://dev.azure.com/sierrasoftworks/opensource/_git/bender
-------------------------------------------------------------------------------------------- #>
{{< /codeblock-tab >}}
{{< codeblock-tab "powershell" "GitHub" >}}
Open-Repo sierrasoftworks/git-tool

<# -----------------------------------------------------------
Name             Value
----             -----
Service          github.com
Exists           True
Namespace        SierraSoftworks
Name             git-tool
Repo             SierraSoftworks/git-tool
Path             C:\dev\github.com\SierraSoftworks\git-tool
GitURL           git@github.com:SierraSoftworks/git-tool.git
WebURL           https://github.com/SierraSoftworks/git-tool
----------------------------------------------------------- #>
{{< /codeblock-tab >}}
{{< /codeblock >}}

### Everything else
There are, of course, more commands included in [Git-Tool][], things like `Get-GitIgnore` which
pulls down a `.gitignore` template from https://gitignore.io for you, or `Get-RepoInfo` which
allows you to quickly get details like the web address, Git URL and more for a given repo.

In the future I'm sure I'll extend [Git-Tool][] even further to support use cases like Gists,
organized scratch folders and more. If you've got ideas for features you'd like to see or
improvements which can be made - I'd love to hear them.

[Git-Tool]: https://github.com/sierrasoftworks/git-tool
[GitHub]: https://github.com
[GitLab]: https://gitlab.com
[BitBucket]: https://bitbucket.org 
[Azure DevOps]: https://dev.azure.com