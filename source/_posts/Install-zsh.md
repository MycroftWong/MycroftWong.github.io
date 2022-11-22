---
title: Install zsh
date: 2022-11-22 21:02:32
tags:
---

# Install zsh

```shell
# install zsh
## CentOS
yum install zsh wget git -y

# install oh-my-zsh
sh -c "$(wget -O- https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"

# install zsh-syntax-highlighting
## Linux
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git $ZSH_CUSTOM/plugins/zsh-syntax-highlighting && sed -i's/^plugins=(/plugins=zsh-syntax-highlighting /' ~/.zshrc
## Mac
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git $ZSH_CUSTOM/plugins/zsh-syntax-highlighting && sed -i "" 's/^plugins=(/plugins=zsh-syntax-highlighting /' ~/.zshrc

# install zsh-autosuggestions
## Linux
git clone https://github.com/zsh-users/zsh-autosuggestions.git $ZSH_CUSTOM/plugins/zsh-autosuggestions && sed -i 's/^plugins=(/plugins=zsh-autosuggestions /' ~/.zshrc
## Mac
git clone https://github.com/zsh-users/zsh-autosuggestions.git $ZSH_CUSTOM/plugins/zsh-autosuggestions && sed -i "" 's/^plugins=(/plugins=zsh-autosuggestions /' ~/.zshrc
```
