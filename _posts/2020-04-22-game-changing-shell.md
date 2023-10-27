---
title: "Game Changing Shell"
date: "2020-04-22"
categories: 
  - "miscellaneous"
  - "play"
coverImage: "git.png"
---

A colleague of mine recently introduced me to the game changing [ohmyzsh](https://github.com/ohmyzsh/ohmyzsh) shell.
Before: ![](/images/normal.png)
After: ![](/images/git.png)

The shell has [tonnes of plugins](https://github.com/ohmyzsh/ohmyzsh/tree/master/plugins) available built in, as well as loads of community driven plugins. The aesthetic themes make long hours much easier and the plugins can really improve your efficiency at the terminal.

It was a little fiddely to set up, so here is a little semi-automated writeup on how to get started.

Terminator is by far my favourite terminal. Install it via apt along with zsh and git (required for setup)

```sudo apt-get install zsh terminator git -y```

Download ohmyzsh `sh -c "$(wget -O- https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"`
When the script has finished, just type `exit` to revert to bash.

If you need to set your default shell to zsh, do the following: [code lanuage="bash"] sudo chsh -s /bin/zsh $(whoami)```

Download & install the meslo nerd font. This include loads of icons for stuff like debian/ubuntu, gitlab, github.
```
mkdir ~/.fonts
wget https://github.com/ryanoasis/nerd-fonts/releases/download/v2.1.0/Meslo.zip -O temp.zip
unzip temp.zip -d ~/.fonts/
rm temp.zip
fc-cache -f -v
```

Download & Install the zsh-syntax-highlighting plugin
```
wget https://github.com/zsh-users/zsh-syntax-highlighting/archive/master.zip -O temp.zip
unzip temp.zip
mv zsh-syntax-highlighting-master /home/$(whoami)/.oh-my-zsh/plugins/zsh-syntax-highlighting
rm temp.zip
```

Install powerlevel10k ```git clone --depth=1 https://github.com/romkatv/powerlevel10k.git ~/.oh-my-zsh/themes/powerlevel10k```

Edit ~/.zshrc and add the following 
```
ZSH_THEME="powerlevel10k/powerlevel10k"
POWERLEVEL10K_MODE="nerdfont-complete"
```

Replace the plugins section (line ~72) with the following. add more plugins as necessary
```
plugins=(git dnf zsh-syntax-highlighting)
```

Now close the terminal and open up terminator. You'll be asked a bunch of questions on customising the terminal.
![](/images/powerlevel10k-prompt.png)
The only thing i recommend is to use the "rainbow" layout and to use unicode and not ascii.
![](/images/post-deployment.png)

If you ever want to re-run the prompt to change the layout, run the following command from anywhere: `p10k configure`

I use Visual Studio code a lot, so ive change the terminator colour to the same as VSCodes default theme. Its #1e1e1e Right click on the screen of terminator and go to preferences. Change to the profiles tab Under General, change the font to MesloLGSDZ Nerd Font Regular Under the colours tab, change the built in schemes to custom and change the colour.
![](/images/terminator-colour-change.png)

If you type in an invalid command - for example "sl" instead of "ls" the command is highlighted red instead of green. This is actually an incredibley useful feature, especially when you are developing and have a dozen different aliases set.
![](/images/syntax-highlighter.png)

If you clone a git repo, you can see at a glance what branch youre on. This is also really useful if you change into some really old clone that you forgot about. 
```
git clone https://github.com/postmanlabs/httpbin.git
```

![](/images/git.png)

On top of this, if you run git and git tab, itll show you all the possible command variants - very useful (much more than any --help suffix)

If you want to integrate the shell into VSCode, open it up, press F1 and type `Preferences : Open Settings (JSON). Then in the settings, append the following at the bottom.
```
"terminal.integrated.fontFamily": "MesloLGS NF" ,
"terminal.integrated.shell.linux": "/bin/zsh",
"terminal.integrated.rendererType": "canvas",
"terminal.integrated.lineHeight": 1.3
```

This will add in the icons and the font onto your VSCode Terminal

There are tonnes of plugins out there and im really just getting started, but im fully converted to zsh.
