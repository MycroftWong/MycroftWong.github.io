+++
date = '2025-07-22T14:38:56+08:00'
draft = false
title = 'Install Zsh'
+++

Zsh is almost best shell nowadays, because it has oh-my-zsh :). You can configure a theme, add useful plugins, such as zsh-syntax-highlighting and zsh-autosuggestions.

Here's a simple process of installing and configuring zsh.

## Step 1: Install Zsh

### On Ubuntu/Debian
```bash
sudo apt install zsh wget git -y
```

### On macOS
```bash
brew install zsh wget git
```

After installation, set Zsh as your default shell(or not, it will automatically change to zsh while installing oh-my-zsh):
```bash
chsh -s $(which zsh)
```

Log out and log back in for the changes to take effect.

## Step 2: Install Oh-My-Zsh

Install Oh-My-Zsh using wget:

```bash
# Using wget
sh -c "$(wget -O- https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

## Step 3: Configure the 'ys' Theme

Edit your Zsh configuration file:
```bash
ZSH_THEME="ys"
```

## Step 4: Install Plugins

### Install zsh-syntax-highlighting
```bash
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git $ZSH_CUSTOM/plugins/zsh-syntax-highlighting
```

### Install zsh-autosuggestions
```bash
git clone https://github.com/zsh-users/zsh-autosuggestions.git $ZSH_CUSTOM/plugins/zsh-autosuggestions
```

## Step 5: Enable Plugins

Edit your Zsh configuration again:
```bash
plugins=(git zsh-syntax-highlighting zsh-autosuggestions)
```

Save and exit.

## Step 6: Apply Changes

Reload your Zsh configuration:
```bash
exec zsh
```

## Step 7: Verify Installation

Check that everything is working:
```bash
# Check Zsh version
zsh --version

# Check Oh-My-Zsh
omz version

# Verify plugins are working (should see colored syntax and suggestions)
echo "This should show syntax highlighting"  # Type slowly to see suggestions
```

## Complete Installation Scripts

### For Ubuntu/Debian
```bash
sudo apt install zsh wget git -y
sh -c "$(wget -O- https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
sed -i 's/robbyrussell/ys/' ~/.zshrc
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git $ZSH_CUSTOM/plugins/zsh-syntax-highlighting
sed -i 's/^plugins=(/plugins=(zsh-syntax-highlighting /' ~/.zshrc
git clone https://github.com/zsh-users/zsh-autosuggestions.git $ZSH_CUSTOM/plugins/zsh-autosuggestions
sed -i 's/^plugins=(/plugins=(zsh-autosuggestions /' ~/.zshrc
```

### For macOS
```bash
brew install zsh wget git
sh -c "$(wget -O- https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
sed -i "" 's/robbyrussell/ys/' ~/.zshrc
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git $ZSH_CUSTOM/plugins/zsh-syntax-highlighting
sed -i "" 's/^plugins=(/plugins=(zsh-syntax-highlighting /' ~/.zshrc
git clone https://github.com/zsh-users/zsh-autosuggestions.git $ZSH_CUSTOM/plugins/zsh-autosuggestions
sed -i "" 's/^plugins=(/plugins=(zsh-autosuggestions /' ~/.zshrc
```

## Install other useful tools

### 1. fzf

```bash
# Install
brew install fzf

# Enable in Zsh
echo '[ -f ~/.fzf.zsh ] && source ~/.fzf.zsh' >> ~/.zshrc
```

### 2. zoxide

```bash
# Install
brew install zoxide

# Enable in Zsh
echo 'eval "$(zoxide init zsh)"' >> ~/.zshrc
```

### 3. lsd

```bash
# Install
brew install lsd

# Create aliases
echo '\n' >> ~/.zshrc
echo "alias ls='lsd'" >> ~/.zshrc
echo "alias l='ls -l'" >> ~/.zshrc
echo "alias la='ls -a'" >> ~/.zshrc
echo "alias lla='ls -la'" >> ~/.zshrc
echo "alias lt='ls --tree'" >> ~/.zshrc
echo "alias ltr='ls -l --sort time'" >> ~/.zshrc
```
