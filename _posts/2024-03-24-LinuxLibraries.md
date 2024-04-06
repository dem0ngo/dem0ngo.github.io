---
title: Running multiple library versions on Linux
date: 2024-03-25 12:30:00
categories:
  - Guide
  - Linux
tags:
  - arch
  - pacman
---
> In light of [recent events](https://en.wikipedia.org/wiki/XZ_Utils_backdoor) emphasizing how libraries can be used maliciously, I'd recommend following this guide with caution. If you aren't absolutely certain that the older package is safe, just wait for official updates!
{: .prompt-danger }

Recently I decided to start using Linux as my daily driver. I found that using Kali virtual machines were just too slow and clunky for the things I'm trying do whether that's learning how to use Burpsuite or practicing in a lab environment like Hack the Box. 

After doing some research I settled on Arch *specifically* so that I can say, "I use Arch by the way." Jokes aside, I loved the customization and the challenge it offered and so I spent more than a few hours configuring it just how I liked.  One of the other reasons I liked it was because of its package manager "pacman". It has a huge number of packages avaialable officially and even more on the user submitted repository which means most programs just work when I try and install them. The key word there is **most**.

The other day I was working in Neovim when suddenly I noticed an annoying little error with my language servers that depended on NPM: 

![Pyright Error](/assets/img/lib_versions/library_error_pyright.png)

After some investigating I learned that this can happen for a few reasons, but the two most common are:
1. You did a partial-update of your system which caused there to be a mismatch between the packages you have and the packages that are available.
2. A package updated to a new version, changed the file names (specifically in the case of some libraries), and broke another package's dependency on the older library version. 

In my case, the issue was the latter. My Python Language Server Protocol, Pyright, was dependent on the shared library that International Components for Unicode (ICU) 73 provided, but when I updated my system and it replaced that library with ICU 74, it caused Pyright to break. 

With that explained, lets get into the specific problem and solution.

## The Problem
There is a mismatch between the version of a library your system is running and the one that a package depends on which results in an error like this:
```
node: error while loading shared libraries: libicui18n.so.73: cannot open shared object file: No such file or directory
```

## The Solution
### Acquire the correct library version
Find the package you need on the Arch Repository. In this example we will be going with ICU since that is the package that holds the `libicui18n.so.73` library. Then click on "View Changes" under the "Package Actions" section.

![Arch Package](/assets/img/lib_versions/arch_package.png)

Locate the commit that you need and select "Browse Files".

![Locate Commit](/assets/img/lib_versions/locate_commit.png)

Click on the "Code" dropdown and download the source code in whatever format you prefer.
> DO NOT try and use `git clone` as the urls will be for the newest commit instead of the commit you are looking at!
{: .prompt-warning }

![Download Commit](/assets/img/lib_versions/download_commit.png)

### Prepare the files for building
Extract the files to a location of your choosing. I decided to make a directory called `.alternate_packages` in my home directory so that I have a unified place to store these alternate package versions if I ever need more in the future.

Move into the newly extracted directory and make sure you see a file called `PKGBUILD`.  Now, run the `makepkg` command and you will see the package being built in your terminal.
> If you get errors during the build command you may need to do a bit of research or use `man makepkg` to find and set flags to work around those errors/failures
{: .prompt-tip }

### Set up library for use
Now that the library has been successfully built, you're almost done. But before that, we need to tell the system how to find the files. If you were to try and use a package that needs one of these libraries before doing anything else then you would get the same error as before. 

To fix this you'll have to set up the appropriate environment variable and append it to the paths already being specified there. Depending on your terminal shell you'll want to open the configuration file for editing so that we can make sure this variable is set up correctly each time you start a new session.

I'm using Zsh so the command I run is:
```
nvim ~/.zshrc
```

> Check which shell is being used by running `ps -p $$` and looking in the last column of the output. If you aren't using zsh there is a good chance you are using bash. In that case the command will look like the following: `nvim ~/.bashrc`
{: .prompt-tip}

Once you're editing the correct configuration file you can add the following line anywhere you like (generally the bottom of the file is a good option):
```
export LD_LIBRARY_PATH=~/PACKAGES_FOLDER/PACKAGE_NAME/DIR/TO/LIB/FILES:$LD_LIBRARY_PATH
```

In this example it would be:
```
export LD_LIBRARY_PATH=~/.alternate_packages/icu_73/pkg/icu/usr/lib:$LD_LIBRARY_PATH`
```

![Zsh Config](/assets/img/lib_versions/zsh_config.png)

This command tells your shell to look for the library in the location specified and to **append** that to the current environment variable rather than replacing it. 
>If you were to remove everything after the directory, starting with `:` this would overwrite the environment variable instead. Generally, this isn't a good idea.
{: .prompt-danger}

Finally, save the file and either close and reopen your terminal or source the changes with `source ~/.zshrc`

Congrats! Try running a package that needed the old library and it should run as expected this time! 
