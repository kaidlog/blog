---
title: Building a CLI in Bash
date: 2016-12-12T08:43:55.000+00:00
tags:
- development
- github
- bash
- opensource
draft: true

---
{{< alert info >}}
If you're just looking to hop straight to the final project, you'll want
to check out [SierraSoftworks/bash-cli](https://github.com/SierraSoftworks/bash-cli) on GitHub.
{{< /alert >}}

Anybody who has worked in the ops space as probably built up a veritable
library of scripts which they use to manage everything from deployments
to [brewing you coffee](https://github.com/NARKOZ/hacker-scripts).

Unfortunately, this tends to make finding the script you're after
and its usage information a pain, you'll either end up `grep`-ing
a README file, or praying that the script has a `help` feature built
in.

Neither approach is conducive to a productive workflow for you or
those who will (inevitably) replace you. Even if you do end up adding
help functionality to all your scripts, it's probably a rather significant
chunk of your script code that is dedicated to docs...

After a project I was working on started reaching that point, I decided
to put together a tool which should help minimize both the development
workload around building well documented scripts, as well as the usage
complexity related to them.

<!--more-->

## What makes a good CLI?
I'm sure, if you're reading this post, that you've had experiences with
both good and bad CLIs; but have you ever analyzed what makes a good
CLI?

Before I started building [bash-cli][] I sat down and fleshed out a couple
of features I felt defined good quality command line interfaces.

#### 1. Accessible on your `$PATH`
A command line tool which requires you to first locate its directoy on
disk and then run `./scripts/my-app a b c` will always fall far short of
a simple `my-app a b c` in terms of usability, especially if the scripts
themselves are `cwd` dependent...

Making sure that the scripts this tool exposed could be made available
through your `$PATH` without causing unintentional name conflicts was
one of the core design aspects here. 

#### 2. Provides script usage information
Any command line tool worth its salt is required to show you how to
use its various commands. The moment someone has to open up the docs
to find out how to perform an action, you have failed.

This is most commonly provided when invalid arguments are provided to
a script, or when the contextual help is triggered.

#### 3. Provides contextual help
Contextual help is a very important component of practical command
line tools. The help information should give the user any information
required to make a decision on whether the command meets their requirements
or not.

#### 4. Exits non-zero on failures
From an automation perspective, a command which exits with an exit-code
of `0` when it has failed is beyond infuriating - it immediately prevents
you from being able to run things like `my-app test a && my-app exec a b c`
reliably.

Any tool which wraps scripts should ensure that it surfaces the script's exit
codes reliably, ensuring that any tools using them can continue to do so
without issue.

#### 5. Installs quickly and easily
Finally, a good CLI should be simple and easy to install. If you are required
to pull down a compiler, runtime dependencies or build things then the
barrier to entry is already raised far above what your average user is
willing to do.

Ideally, a one-liner with no install requirements should be all it takes
to setup your CLI.

## What options exist already?
There are a good number of options out there for building command line
tools, in every language under the sun. Some of the best ones I've had
experience with include Go's brilliant [urfave/cli][] and JavaScripts's
[commander][].

{{< codeblock "Command Lines" >}}
    {{< codeblock-tab js "JavaScript" >}}
    import * as program from "commander";
    program
        .version("0.0.1")
        .option("-p, --peppers", "Add peppers")
        .option('-P, --pineapple', 'Add pineapple')
        .option('-b, --bbq-sauce', 'Add bbq sauce')
        .option('-c, --cheese [type]', 'Add the specified type of cheese [marble]', 'marble')
        .parse(process.argv);

    console.log("You ordered a pizza with:");
    program.peppers && console.log("  - peppers");
    program.pineapple && console.log("  - pineapple");
    program.bbqSauce && console.log("  - bbq-sauce");
    console.log(`  - ${program.cheese} cheese`);
    {{< /codeblock-tab >}}
    {{< codeblock-tab go "Go" >}}
    package main
    import (
        "fmt"
        "os"

        "gopkg.in/urfave/cli.v2"
    )

    func main() {
        app := cli.NewApp()
        app.Name = "Pizza"
        app.Usage = "Order a pizza"
        app.Flags = []cli.Flag{
            cli.BoolFlag{ Name: "peppers, p" },
            cli.BoolFlag{ Name: "pineapple, P" },
            cli.BoolFlag{ Name: "bbq-sauce, b" },
            cli.StringFlag{ Name: "cheese, c" },
        }
        app.Action = func(c *cli.Context) error {
            fmt.Println("You ordered a pizza with:")
            if c.Bool("peppers") {
                fmt.Println("  - peppers")
            }
            if c.Bool("pineapple") {
                fmt.Println("  - pineapple")
            }
            if c.Bool("bbq-sauce") {
                fmt.Println("  - bbq-sauce")
            }
            
            fmt.Printf("  - %s cheese\n", c.String("cheese"))
        }
    }
    {{< /codeblock-tab >}}
{{< /codeblock >}}

The problem with both of these solutions is that they require a runtime
or compiler, breaking our [5th rule of good CLIs](#5-installs-quickly-and-easily).
Both also mean that you need to build your entire set of scripts on top of
JavaScript or Go, something I'm sure nobody wants to do unless it's absolutely
necessary.

So what can we do instead?

## Command Proxies
The notion of a command proxy is rather analagous to a reverse proxy in the web
world. It's an application which runs other apps based on the arguments it recieves,
passing through arguments to the child application.

Adopting this approach has a number of benefits as it enables you to standardize
the way scripts are accessed, provide wrapping functionality and only requires
the proxy script to be available on the `$PATH` in order to function.

As the solve purpose of a proxy is to forward requests to another service, or script
as the case may be, the language and framework on which it is implemented is almost
a non-issue. This potentially enables us to write the proxy in anything from Go to
Python, [JavaScript to Java](http://stackoverflow.com/a/245073). Of course, we're
building something for Bash scripts, intended to be used by people who write Bash
scripts on a daily basis, it makes perfect sense that we use C++ for this role...

Jokes aside, we'll be using Bash as it makes for a portable, understandable and
customizable framework wihch any of our users can modify with ease.

## Designing Bash CLI
I'm a strong proponent of only writing code once you've defined your problem domain
and figured out the solution. In Bash CLI's case, our two problems are how we
meet the [5 features which make a good CLI](#what-makes-a-good-cli) and how we
make the lives of developers using our framework as easy as possible.

Many frameworks seem to forget that their primary users aren't the people who
will use the products built on them, but the people who use the framework to build
those products. Features and end-user facing components, while important, will
not drive adoption of your framework in quite the same way that a great developer
experience will.

With that in mind, we want Bash CLI's developer experience to be simple enough
that documentation becomes extraneous rather than necessary. To accomplish that,
let's create a directory structure which we'd like to work with and flesh out
the commands we'd like to use to interact with it.

{{< codeblock "Bash CLI Design" >}}
    {{< codeblock-tab sh "Commands" >}}
    # Print out the help information for the application
    proxy help

    # Print out the help information for "cmd"
    proxy help cmd

    # Print out the help information for the group "group"
    proxy help group

    # Print out the help information for "group cmd"
    proxy help group cmd

    # Run the command "cmd" and pass the argument "arg"
    proxy cmd "arg"

    # Print out the help information for the group "group"
    proxy group

    # Run the command "group cmd" and pass it 3 arguments
    proxy group cmd "arg" "arg" arg
    {{< /codeblock-tab >}}
    {{< codeblock-tab "" "Directory Structure" >}}
    +--+-group # Command group "group"
    |  |
    |  +- .help # Help for command group 
    |  |  
    |  +- cmd # Command script for "group cmd"
    |  |
    |  +- cmd.usage # Usage details for "group cmd"
    |  |
    |  +- cmd.help # Help file for "group cmd"
    |
    +- .author # Author information for this app
    |
    +- .bash_cli # Bash CLI API versioning marker
    |
    +- .name # Name of the application
    |
    +- .help # Help information for this application
    |
    +- cmd # Command script for "cmd"
    |
    +- cmd.usage # Usage information for "cmd"
    |
    +- cmd.help # Help file for "cmd"
    {{< /codeblock-tab >}}
{{< /codeblock >}}

Now that we've got that out of the way, how do we make it work?

### Finding the Script
Well the best place to start would be figuring out which script we're
expected to run, given an arguments list like `proxy group cmd`.
To do that, we'll iterate over our command line arguments, looking
for directories until we find the right script.

```sh
#!/bin/bash

CMD_FILE=`pwd`
CMD_ARG_START=1
while [[ -d "$CMD_FILE" && $CMD_ARG_START -le $# ]]; do

    # If the user provides help as the last argument on a directory, then
    # show them the help for that directory rather than continuing
    if [[ "${!CMD_ARG_START}" == "help" ]]; then
        "$ROOT_DIR/help" $0 ${@:1:$(($CMD_ARG_START-1))}
        exit 3
    fi

    CMD_FILE="$CMD_FILE/${!CMD_ARG_START}"
    CMD_ARG_START=$(($CMD_ARG_START+1))
done

# Print out the command and its arguments
echo "Command: $CMD_FILE"
echo "Arguments: ${@:$CMD_ARG_START}"
```

This works brilliantly, but wasn't the point of this excercise that
we didn't need to know our script's directory? Having it look in the
current directory (`CMD_FILE="$(pwd)"`) isn't going to help anybody.

### Find the Scripts Directory
The solution we've opted to go for is that you have the proxy reside
somewhere it can always find the scripts. You'd then either add that
folder to your `$PATH`, or better yet, `symlink` the proxy itself into
your `$PATH`.

To do this, we want to determine the true path of the proxy's file.
The simplest approach is to use the `readline -f` command like this:

```sh
#!/bin/bash

TRUE_SCRIPT=`readline -f "$0"`
ROOT_DIR=`dirname "$TRUE_SCRIPT"` 
```

Unfortunately, the first time you run this on a Mac you'll find that
the `-f` flag doesn't work... That's unfortunate, so how do we fix it?

There's a couple of hacky approaches, but the best is to use something
which is available almost everywhere, `perl`.

```sh
#!/bin/bash

TRUE_SCRIPT=`perl -e 'use Cwd "abs_path"; print abs_path(shift)' $0`
ROOT_DIR=`dirname "$TRUE_SCRIPT"` 
```

### Execute the Script
Now that we've got the script directory and can find the script based
on that, we'll want to actually run the script. Bash makes doing so
super easy, so using our variables from
[Finding the Script](#finding-the-script), we'll run the command file
and pass the remaining arguments to it.

```sh
"$CMD_FILE" "${@:$CMD_ARG_START}"
```

We accomplish this using the `$@` variable, which spreads the arguments
received by the running script as well as the `${VAR:OFFSET}` slice operation
which will take everything from `VAR` at the given `OFFSET` and onwards.

Combining these two with `${@:2}` will return everything after, and including,
the second argument to the script. Finally, we use our argument offset which
was computed when looking for the command file to determine where to start.

[bash-cli]: https://github.com/SierraSoftworks/bash-cli
[urfave/cli]: https://github.com/urfave/cli
[commander]: https://www.npmjs.com/package/commander

*[CLI]: Command Line Interface