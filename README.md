# Macsible

[![Build Status](https://travis-ci.org/phluid/macsible.svg?branch=master)](https://travis-ci.org/phluid/macsible)

## Forking and customisation overview

This repository is forked from macsible [https://github.com/macsible/macsible](https://github.com/macsible/macsible).
If you want to use macsible, I recommend checking the up-to-date original repo. You can copy the customizations done here of course.
The largest difference to the original repo is that this repo provides a ready `init.sh` script, taking care of the requirements, that you're supposed to copy-paste into a file and run (see below).
You're supposed to run `init.sh` before making any other customizations like changing shells etc., basically as the first thing after the MacOS install is done, and let Macsible then take care of all customizations.

## Requirements and init

Currently targeting MacOS High Sierra.

Save the following script as `init.sh` and run it (`chmod +x init.sh` and `./init.sh`) to setup and install the required tools.

```sh
#!/bin/sh

# Not /bin/bash, if you want to change it to bash, you need to add echo flag -e
# to most lines.

# Vars
# ------------------------------------------------------------------------------

BGreen='\e[1;32m' # Green
Color_Off='\e[0m' # Text Reset

# Functions
# ------------------------------------------------------------------------------

function setStatusMessage {
  printf " --> ${BGreen}$1${Color_Off}\n" 1>&2
}

fancyEcho() {
  local fmt="$1"; shift

  # shellcheck disable=SC2059
  printf "\n$fmt\n" "$@"
}

# Check whether a command exists - returns 0 if it does, 1 if it does not
function commandExists {
  if command -v $1 >/dev/null 2>&1
  then
    return 0
  else
    return 1
  fi
}

# Script
# ------------------------------------------------------------------------------

set -e

setStatusMessage "Checking system updates"
fancyEcho "Have you updated the system? For that: Apple Icon on the right corner > App Store... > Updates"
echo ""
read -n 1 -p "I solemnly swear that I have updated the system [y/n]: " prompt_have_updated
echo ""

if [ "$prompt_have_updated" != "y" ]; then
  exit
fi

echo "\n\n"

setStatusMessage "Checking xcode install"
xcodePath=`xcode-select -p`
if [ "$?" = "0" ]; then
  fancyEcho "Detected xcode cli tools as already installed at \"$xcodePath\", skipping install"
else
  fancyEcho "Enabling xcode cli tools. If xcode is not installed, you'll be prompted for download and install. Click \"Install\" and then \"Done\"."
  xcode-select --install
fi

echo "\n\n"


setStatusMessage "Checking SSH keys"
mkdir -p "$HOME/.ssh"
ssh_keyfile_basic="$HOME/.ssh/id_ed25519"
ssh_keyfile_github="$HOME/.ssh/id_ed25519_github"
if [ ! -f "$ssh_keyfile_basic" ]; then
  fancyEcho "Generating an basic ssh-key for this computer, please give a comment e.g. \"$USER@gmail.com\""
  read -p "Comment for basic ssh key: " prompt_basic_ssh_comment
  echo ""
  ssh-keygen -t ed25519 -C "$prompt_basic_ssh_comment" -f "$ssh_keyfile_basic"
fi
if [ ! -f "$ssh_keyfile_github" ]; then
  fancyEcho "Generating a github specific ssh-key for this computer, please give a comment e.g. \"$USER@gmail.com\""
  read -p "Comment for github ssh key: " prompt_github_ssh_comment
  echo ""
  ssh-keygen -t ed25519 -C "$prompt_github_ssh_comment" -f "$ssh_keyfile_github"

  ssh_keyfile_github_pub="${ssh_keyfile_github}.pub"
  pbcopy < "$ssh_keyfile_github_pub"
  fancyEcho "Please login to GitHub (on Safari) and upload the newly created pubkey. It has been copied to your clipboard."
  fancyEcho "https://github.com/settings/keys"
  fancyEcho "You can re-copy the pubkey to your clipboard by running"
  fancyEcho "    pbcopy < \"$ssh_keyfile_github_pub\""

  prompt_have_uploaded_github_pubkey="n"
  while [ "$prompt_have_uploaded_github_pubkey" != "y" ]; do
    read -n 1 -p "I have uploaded the pubkey to GitHub [y/n]: " prompt_have_uploaded_github_pubkey
    echo ""
  done
else
  ssh_keyfile_github_pub="${ssh_keyfile_github}.pub"
  fancyEcho "Have you uploaded the github specific pubkey to GitHub?"
  fancyEcho "If not, you can re-copy the pubkey to your clipboard by running"
  fancyEcho "    pbcopy < \"$ssh_keyfile_github_pub\""
  fancyEcho "and upload it at https://github.com/settings/keys"
  prompt_have_uploaded_github_pubkey="n"
  while [ "$prompt_have_uploaded_github_pubkey" != "y" ]; do
    read -n 1 -p "I have uploaded the pubkey to GitHub [y/n]: " prompt_have_uploaded_github_pubkey
    echo ""
  done
fi

fancyEcho "Making sure ~/.ssh/config contains the GitHub settings."
ssh_config_file="$HOME/.ssh/config"
ssh_config_github_line="IdentityFile $HOME/.ssh/id_ed25519_github"
grep -qF -- "$ssh_config_github_line" "$ssh_config_file" || echo "Host github github.com\nHostName github.com\nUser git\n$ssh_config_github_line" >> "$ssh_config_file"

echo "\n\n"


# Download and install Miniconda
setStatusMessage "Checking miniconda install"
miniconda_dir="$HOME/miniconda3"
miniconda_installer="$HOME/tmp/Miniconda_Install.sh"
mkdir -p "${HOME}/tmp"
if [ ! -d $miniconda_dir ]; then
  fancyEcho "Downloading miniconda installer, saved to $miniconda_installer"
  set +e
  curl "https://repo.continuum.io/miniconda/Miniconda3-latest-MacOSX-x86_64.sh" -o "$miniconda_installer"
  if [ $? -ne 0 ]; then
      curl "http://repo.continuum.io/miniconda/Miniconda3-latest-MacOSX-x86_64.sh" -o "$miniconda_installer"
  fi
  set -e
  fancyEcho "Installing miniconda to $miniconda_dir"
  bash $HOME/tmp/Miniconda_Install.sh -b -p "$miniconda_dir"
  conda clean -iltp --yes
  rm "$miniconda_installer"

  fancyEcho "Setting miniconda path"
  export PATH="${miniconda_dir}/bin:$PATH"
  miniconda_path_line="export PATH=\"$HOME/miniconda3/bin:\$PATH\""
  bash_config_file="$HOME/.bash_profile"
  # check the file, if the line is already there, don't append
  grep -qF -- "$miniconda_path_line" "$bash_config_file" || echo "$miniconda_path_line" >> "$bash_config_file"
else
  fancyEcho "Detected miniconda at $miniconda_dir, skipping install."
fi

echo "\n\n"


setStatusMessage "Checking python environment"
# Install Ansible
if ! commandExists ansible; then
  fancyEcho "Installing Ansible to miniconda root"
  pip install ansible --quiet
fi

# Confirm installed versions
pip --version
ansible --version

echo "\n\n"

setStatusMessage "Install script done"
fancyEcho "All done."
fancyEcho "NOTE: YOU NEED TO OPEN A NEW TERMINAL FOR MINICONDA TO BE ACTIVE"
fancyEcho "Next, please open a new terminal and then clone your macsible repo and run it."
echo ""
```

## Usage

After running the code above, you need to open a new terminal for the installed required tools to be enabled.
Then `git clone` this repo, `cd` to it and run

```
bash create_fork_files.sh
```

which will create the following files based on the examples found in the src directory:

- config.yml
- config.local.yml
- mac-custom.yml
- requirements.yml

### Download externally sourced roles

Remotely sourced Ansible roles can be added to requirements.yml. Before running the playbook you'll need to download any Ansible roles specified in requirements.yml by running the following command:

```
ansible-galaxy install -r requirements.yml --force
```

### Configure

Default variables can be overridden in config.yml.

config.local.yml can be used to override config.yml which can be useful when you need to use different values for just a few variables on a specific system. By default config.local.yml is ignored by git.

### Run the Ansible playbook

The primary Ansible playbook file is called macsible.yml and can be run using the command below (asks for sudo password). Note that running macsible.yml will in turn run your customised mac-custom.yml playbook.

```
ansible-playbook macsible.yml -K
```

To run only certain tags (e.g. `firefox` and `dev_apps`):

```
ansible-playbook macsible.yml -K -t "firefox,dev_apps"
```
