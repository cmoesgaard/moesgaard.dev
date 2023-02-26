---
title: Managing dotfiles with chezmoi
date: 2023-02-26T16:27:07.392Z
---
Between home and work I usually have to deal with a couple of different computers in my life. I try to keep the configuration of each machine as identical as possible, just to make it easier for myself. 

My setup is fairly customized to my own workflow with small tweaks and shortcuts added over the years. It's a continuous work in progress, and I add new tweaks to it regularly. The big issue is keeping the configuration synchronized between my machines.

Keeping your dotfiles synchronized using a Git repo is of course pretty much a no brainer, but this isn't a trivial problem to solve.

* The config files in the repo should be symlinked to their respective locations in your home folder. This doesn't happen automatically, but can be solved using tools like [GNU Stow](https://www.gnu.org/software/stow/)
* Your different computers will still have machine-specific configuration. E.g. a different email configured for Git between the two machines.

I've eventually settled on using [`chezmoi`](https://www.chezmoi.io/) to solve this problem, [as can be see in my dotfiles repo](https://github.com/cmoesgaard/dotfiles).

`chezmoi` manages your configuration files without symlinks, but by keeping a copy of your configuration files' desired contents and locations, and then does its best to make your actual configuration files look like the state in your repo. 

This requires you to use `chezmoi` to edit and update the files, e.g. 

```bash
$ chezmoi edit $FILE
```

This extra layer of indirection allows you to use templates to define your configuration files, in cases where the actual file will differ between machines.

`chezmoi` also makes it easy to get up and running on a new machine or after a OS reinstall, as you can point it towards a public Git repo it should use as its initial state.

```bash
$ chezmoi init https://github.com/$GITHUB_USERNAME/dotfiles.git
```