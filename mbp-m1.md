# Dev Environment Setup Notes on Apple Silicon
Last updated: Dec. 27, 2020

I used the following steps to configure a local development environment for Python & Django and Node & React using a MacBook Pro M1. Not everything is natively supported yet and require workarounds.


## Resources

* https://isapplesiliconready.com


## Initial applications

* Slack via Mac App Store
* [VS Code Insiders](https://code.visualstudio.com/insiders/)


## Homebrew

Homebrew isn't officially supported yet on Apple Silicon. See [macOS 11 Big Sur compatibility on Apple Silicon #7857](https://github.com/Homebrew/brew/issues/7857) for more information.

Many packages will fail to compile without necessary libraries . For example, Ansible fails to install because ``cryptography`` fails to find `openssl`.

```
  ...
  build/temp.macosx-10.14.6-arm64-3.8/_openssl.c:575:10: fatal error: 'openssl/opensslv.h' file not found
  #include <openssl/opensslv.h>
           ^~~~~~~~~~~~~~~~~~~~
  1 error generated.
  
      =============================DEBUG ASSISTANCE=============================
      If you are seeing a compilation error please try the following steps to
      successfully install cryptography:
      1) Upgrade to the latest pip and try again. This will fix errors for most
         users. See: https://pip.pypa.io/en/stable/installing/#upgrading-pip
      2) Read https://cryptography.io/en/latest/installation.html for specific
         instructions for your platform.
      3) Check our frequently asked questions for more information:
         https://cryptography.io/en/latest/faq.html
      =============================DEBUG ASSISTANCE=============================
  
  error: command 'clang' failed with exit status 1
  ----------------------------------------
  ERROR: Failed building wheel for cryptography
  Running setup.py clean for cryptography
Failed to build cryptography
```

1. Run ``python3`` to trigger xcode command line tools to install

```
colincopeland@MacBook-Pro ~ % python3
xcode-select: note: no developer tools were found at '/Applications/Xcode.app', requesting install. Choose an option in the dialog to download the command line developer tools.
```

Or just do `xcode-select --install`?

2. Install ``brew`` into ``/opt/homebrew`` using the Alternative Installs method:

```sh
mkdir homebrew && curl -L https://github.com/Homebrew/brew/tarball/master | tar xz --strip 1 -C homebrew
sudo mv homebrew /opt/homebrew
export PATH="/opt/homebrew/bin:$PATH"
brew update
```

4. Install base package like `curl` which will also install `openssl`. Ignore homebrew warnings:

```
colincopeland@MacBook-Pro bin % brew install -s curl
Warning: You are running macOS on a arm64 CPU architecture.
We do not provide support for this (yet).
Reinstall Homebrew under Rosetta 2 until we support it.
You will encounter build failures with some formulae.
Please create pull requests instead of asking for help on Homebrew's GitHub,
Twitter or any other official channels. You are responsible for resolving
any issues you experience while you are running this
unsupported configuration.

==> Downloading https://homebrew.bintray.com/bottles/pkg-config-0.29.2_3.arm64_big_sur.bottle.tar.gz
...
```

## Mac Dev Playbook

1. Now install ansible:

```sh
export LDFLAGS="-L/opt/homebrew/opt/openssl@1.1/lib"
export CPPFLAGS="-I/opt/homebrew/opt/openssl@1.1/include"
python3 -m pip install ansible --user
```

2. Clone this repo and install galaxy packages:

```sh
mkdir projects && cd projects/
git clone git@github.com:copelco/mac-dev-playbook.git
cd mac-dev-playbook/
export PATH=${PATH}:~/Library/Python/3.8/bin
ansible-galaxy install -r requirements.yml
```

3. Run playbook:

```
ansible-playbook main.yml --connection=local -i inventory -K
```


## Apple Silicon Workaround - Go

Go doesn't work on Apple Silicon yet. direnv, antibody, and other homebrew packages depend on it. These aren't dealbreakers, but it may impact your local setup.

Use this [gist](https://gist.github.com/joseph-ravenwolfe/de8de3c0f79c4684eb4505c2d072d133) to install Go 1.16beta1 manually for now:


1. Install [go1.16beta1.darwin-arm64.pkg](https://golang.org/dl/#go1.16beta1).
2. Run `mkdir /opt/homebrew/Cellar/go`
3. Create a symlink to the Go v1.16 pkg installation with `ln -s /usr/local/go /opt/homebrew/Cellar/go/1.16`
4. Run `brew link go`
5. Ensure `/opt/homebrew/bin` is listed ahead of `/usr/local/bin` in your $PATH
6. Open a new shell and test by running `go version` which should report `go1.16beta1 darwin/arm64`
7. Running `which go` should show that it is coming from Homebrew as `/opt/homebrew/bin/go`

Installing packages from Homebrew that depend on Go will require checking the dependencies of the package using `brew info <package_name>` and installing each of the dependencies (except for Go) manually by running `brew install <dependency_name>`.

After installing all non-Go dependencies, we should be able to run `brew install <package_name> --ignore-dependencies` to install the package while using the ARM version of Go linked from Homebrew.


### Example

```sh
# Attempt to install Direnv which relies on Go
> brew info direnv
...
==> Dependencies
Build: go ✘
> brew install direnv --ignore-dependencies
🍺  /opt/homebrew/Cellar/direnv/2.25.2: 10 files, 8.4MB, built in 4 seconds
```


## Apple Silicon Workaround - Docker

Install Docker's [Apple M1 Tech Preview](https://docs.docker.com/docker-for-mac/apple-m1/).