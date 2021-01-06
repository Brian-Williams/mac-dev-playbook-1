# Python, Django, and React Development on Apple Silicon

I'm using the following steps to configure a local development environment for Python & Django and Node & React using a MacBook Pro (13-inch, M1, 2020). This is a work in progress. Obviously not everything is natively supported yet and require workarounds.


## Applications and tools I use daily

* 


## Rosetta 2 and Command Line Tools for Xcode

[Rosetta 2](https://support.apple.com/en-us/HT211861) enables a Mac with Apple silicon to use apps built for a Mac with an Intel processor. Install it with:

```sh
softwareupdate --install-rosetta  --agree-to-license
```

Also install Xcode's command line tools for `python3` and other useful libraries:

```
xcode-select --install
```


### Intel-emulated Terminal

Use the `arch` command to run commands under Rosetta 2:

```sh
> arch
arm64
> arch -x86_64 bash -c "arch"
i386
```

My current `.zshrc` configuration is incompatible using Rosetta 2, so you may not be able to launch a `zsh` shell like this:

```
❯ arch -x86_64 zsh
arch: posix_spawnp: zsh: Bad CPU type in executable
```

So I just use `bash` as noted in the first example to install packages under Rosetta 2.


## Install Homebrew Natively on Apple Silicon

Homebrew isn't officially supported yet on Apple Silicon, but you can use it. See [macOS 11 Big Sur compatibility on Apple Silicon #7857](https://github.com/Homebrew/brew/issues/7857) for more information. 

Install ``brew`` into ``/opt/homebrew`` using the Alternative Installs method:

```sh
mkdir homebrew && curl -L https://github.com/Homebrew/brew/tarball/master | tar xz --strip 1 -C homebrew
sudo mv homebrew /opt/homebrew
export PATH="/opt/homebrew/bin:$PATH"
brew update
```

## Install Intel-emulated Homebrew

Install Intel-emulated Homebrew to the default `/usr/local`:

```
arch --x86_64 /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install.sh)"
```

Add an `alias` to easily run the emulated brew:

```
❯ alias brewx86='arch --x86_64 /usr/local/Homebrew/bin/brew'
❯ which brewx86
brewx86: aliased to arch --x86_64 /usr/local/Homebrew/bin/brew
❯ brewx86 info
0 kegs, 64B
```

## Install Ansible to run Mac Dev Playbook

I use this repository to automate base configuration of my Mac development environment, which uses Ansible. Ansible fails to install because ``cryptography`` fails to find `openssl`.

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

Install `openssl` using Homebrew (ignore arm64 warnings):

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

Now install `ansible`:

```sh
export LDFLAGS="-L/opt/homebrew/opt/openssl@1.1/lib"
export CPPFLAGS="-I/opt/homebrew/opt/openssl@1.1/include"
python3 -m pip install ansible --user
```


## Mac Dev Playbook

1. Clone this repo and install galaxy packages:

   ```sh
   mkdir projects && cd projects/
   git clone git@github.com:copelco/mac-dev-playbook.git
   cd mac-dev-playbook/
   export PATH=${PATH}:~/Library/Python/3.8/bin
   ansible-galaxy install -r requirements.yml
   ```

2. Run playbook:

   ```
   ansible-playbook main.yml -i inventory -K
   ```


## Apple Silicon Workaround - Go

Go isn't officially supported on Apple Silicon yet. `direnv`, `antibody`, and other homebrew packages depend on it. These aren't dealbreakers, but it may impact your local setup.

Use this [gist](https://gist.github.com/joseph-ravenwolfe/de8de3c0f79c4684eb4505c2d072d133) to install Go `1.16beta1` manually for now:

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


## Apple Silicon Workaround - nvm

``nvm`` is available in Homebrew:

```sh
brew install nvm
```

I was able to install the latest version of node using `nvm`:

```sh
❯ nvm install v15
```

It builds from source and takes a while, but eventually produces an arm64 executable:

```sh
❯ file `which node`
/Users/colincopeland/.nvm/versions/node/v15.5.0/bin/node: Mach-O 64-bit executable arm64
❯ node -p process.arch
arm64
```

However, as noted in [nvm install node fails to install on macOS Big Sur M1 Chip #2350](https://github.com/nvm-sh/nvm/issues/2350), older versions of node require Rosetta 2.

You can install Node v12 like this:

```sh
❯ arch -x86_64 bash
❯ source /opt/homebrew/opt/nvm/nvm.sh
❯ nvm install v12
Downloading and installing node v12.20.0...
Downloading https://nodejs.org/dist/v12.20.0/node-v12.20.0-darwin-x64.tar.xz...
Computing checksum with shasum -a 256
Checksums matched!
Now using node v12.20.0 (npm v6.14.8)
```

Now you can switch seamlessly:

```sh
❯ nvm use 12
Now using node v12.20.0 (npm v6.14.8)
❯ node -p process.arch
x64
❯ nvm use default
Now using node v15.5.0 (npm v7.3.0)
❯ node -p process.arch
arm64
```

## Python

Xcode's command line tools provide several versions of Python that can run natively on Applie Silicon:

```sh
❯ python2 --version
Python 2.7.16
❯ python3 --version
Python 3.8.2
❯ file /usr/bin/python
/usr/bin/python: Mach-O universal binary with 2 architectures: [x86_64:Mach-O 64-bit executable x86_64] [arm64e:Mach-O 64-bit executable arm64e]
/usr/bin/python (for architecture x86_64):      Mach-O 64-bit executable x86_64
/usr/bin/python (for architecture arm64e):      Mach-O 64-bit executable arm64e
❯ file /usr/bin/python3
/usr/bin/python3: Mach-O universal binary with 2 architectures: [x86_64:Mach-O 64-bit executable x86_64] [arm64e:Mach-O 64-bit executable arm64e]
/usr/bin/python3 (for architecture x86_64):     Mach-O 64-bit executable x86_64
/usr/bin/python3 (for architecture arm64e):     Mach-O 64-bit executable arm64e
```

This means the following version of Python can run natively:

* 2.7 (from Xcode)
* 3.8 (from Xcode)
* 3.9 (using pyenv below)

According to [bpo-41100: Support macOS 11 and Apple Silicon #22855](https://github.com/python/cpython/pull/22855), there's work underway to backport the 3.9 fixes to 3.8 in the official CPython distribution. [Issue 41100](https://bugs.python.org/issue41100#msg382939 indicates Python 3.7 and below will never be supported:

> There are no plans to backport support to 3.7 and 3.6 which are in the security-fix-only phase of their release cycles.

Rosetta 2 is required for these versions, but I haven't investigated this path yet.


### Install pyenv from source

I like [pyenv](https://github.com/pyenv/pyenv) for Python version management. I used to use [python-virtualenv](https://github.com/pyenv/pyenv-virtualenv) to manage virtualenvs, but I think it may be [unmaintained](https://github.com/pyenv/pyenv-virtualenv/issues/374). direnv's [layout_python](https://github.com/direnv/direnv/wiki/Python) looks like a great alternative. Not only does it include `pyenv` support, but also the ability to use other Python binaries.

pyenv hasn't released a new version to be picked up with Homebrew yet, so we'll install pyenv from source:

```
git clone git@github.com:pyenv/pyenv.git ~/.pyenv
```

Add to `.zshrc`:

```sh
# pyenv git checkout
export PYENV_ROOT="$HOME/.pyenv"
export PATH="$PYENV_ROOT/bin:$PATH"
```

Install latest Python 3.9.x version:

```sh
❯ pyenv install 3.9.1
python-build: use openssl@1.1 from homebrew
python-build: use readline from homebrew
Downloading Python-3.9.1.tar.xz...
-> https://www.python.org/ftp/python/3.9.1/Python-3.9.1.tar.xz
Installing Python-3.9.1...
python-build: use readline from homebrew
python-build: use zlib from xcode sdk
Installed Python-3.9.1 to /Users/colincopeland/.pyenv/versions/3.9.1
```

This installs a Python 3.9.x `arm64` binary:

```
❯ file ~/.pyenv/versions/3.9.1/bin/python
/Users/colincopeland/.pyenv/versions/3.9.1/bin/python: Mach-O 64-bit executable arm64
```

And can easily be used with direnv in your `.envrc`:

```sh
# .envrc
layout pyenv 3.9.1
```


### Python packages requiring compilation

I found it useful to add these flags to your `.zshrc` to ensure Python packages compile properly:

```sh
# usful Python C-library compliation flags
export LDFLAGS="-L/opt/homebrew/opt/openssl@1.1/lib -L/opt/homebrew/opt/zlib/lib"
export CPPFLAGS="-I/opt/homebrew/opt/openssl@1.1/include -I/opt/homebrew/opt/zlib/include"
```


### Accessing DLLs or shared libraries

Python libraries can wrap C libraries. For example, [python-magic](https://github.com/ahupp/python-magic) is a simple wrapper around the libmagic C library. ``libmagic`` is in Homebrew, but python-magic fails to find the library (perhaps due to arm64 Homebrew being installed on a non-standard prefix: ``/opt/homebrew/``). 

I was seeing an error like this:

```python
>>> import magic
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "/Users/.../projects/.../.direnv/python-3.8.2/lib/python3.8/site-packages/magic.py", line 201, in <module>
    raise ImportError('failed to find libmagic.  Check your installation')
ImportError: failed to find libmagic.  Check your installation
```

When Homebrew installs ``libmagic``, it also runs ``brew link libmagic`` which creates the necessary symbolic links into the ``/opt/homebrew/`` paths, such as ``/opt/homebrew/lib/libmagic.dylib``. ``python-magic`` doesn't [search this path on Darwin](https://github.com/ahupp/python-magic/blob/ca14bfba7d1eeea543c9e00ea33d1487a49e68e0/magic.py#L218-L219) through [Python's ctypes](https://docs.python.org/3/library/ctypes.html#finding-shared-libraries):

```python
    platform_to_lib = {'darwin': ['/opt/local/lib/libmagic.dylib',
                                  '/usr/local/lib/libmagic.dylib'] +
```

As a workaround, I set ``DYLD_FALLBACK_LIBRARY_PATH`` in my project's `.direnv` to search this path:

```sh
export DYLD_FALLBACK_LIBRARY_PATH=/opt/homebrew/lib
```

And now ``import magic`` works for me.


## Install Intel-emulated Python 3.7

Python 3.7 isn't supported on Apple Silicon. While it can be installed via Homebrew using Rosetta 2:

```sh
arch -x86_64 bash
export PATH="/usr/local/Homebrew/bin:$PATH"
export LDFLAGS="-L$(brew --prefix openssl)/lib -L$(brew --prefix zlib)/lib"
export CPPFLAGS="-I$(brew --prefix openssl)/include -I$(brew --prefix zlib)/include"
brew install python@3.7
```

And you can run it normally using:

```sh
❯ export PATH="/usr/local/opt/python@3.7/bin:$PATH"
❯ file `which python3.7`
/usr/local/opt/python@3.7/bin/python3.7: Mach-O 64-bit executable x86_64
❯ python3.7
Python 3.7.9 (default, Nov 20 2020, 23:58:42) 
[Clang 12.0.0 (clang-1200.0.32.27)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>> 
```

I've run into issues with packages with external dependencies (such as `libffi`):

```sh
❯ brewx86 install libffi
❯ /usr/local/opt/python@3.7/bin/python3 -m pip install bcrypt
❯ /usr/local/opt/python@3.7/bin/python3 -m bcrypt
Traceback (most recent call last):
  File "/usr/local/Cellar/python@3.7/3.7.9_2/Frameworks/Python.framework/Versions/3.7/lib/python3.7/runpy.py", line 183, in _run_module_as_main
    mod_name, mod_spec, code = _get_module_details(mod_name, _Error)
  File "/usr/local/Cellar/python@3.7/3.7.9_2/Frameworks/Python.framework/Versions/3.7/lib/python3.7/runpy.py", line 142, in _get_module_details
    return _get_module_details(pkg_main_name, error)
  File "/usr/local/Cellar/python@3.7/3.7.9_2/Frameworks/Python.framework/Versions/3.7/lib/python3.7/runpy.py", line 109, in _get_module_details
    __import__(pkg_name)
  File "/usr/local/lib/python3.7/site-packages/bcrypt/__init__.py", line 25, in <module>
    from . import _bcrypt  # type: ignore
ImportError: dlopen(/usr/local/lib/python3.7/site-packages/_cffi_backend.cpython-37m-darwin.so, 2): Symbol not found: _ffi_type_double
  Referenced from: /usr/local/lib/python3.7/site-packages/_cffi_backend.cpython-37m-darwin.so
  Expected in: flat namespace
 in /usr/local/lib/python3.7/site-packages/_cffi_backend.cpython-37m-darwin.so
 ```


## Apple Silicon Workaround - Docker

Install Docker's [Apple M1 Tech Preview](https://docs.docker.com/docker-for-mac/apple-m1/).


### Kubernetes - kubectl CLI

`kubectl` on Apple Silicon is slated for [Go 1.16](https://github.com/kubernetes/kubectl/issues/988). Following [Install kubectl binary with curl on macOS](https://kubernetes.io/docs/tasks/tools/install-kubectl/#install-kubectl-binary-with-curl-on-macos), I wasn't able to run the binaries using Rosetta 2. However, Docker's [Apple M1 Tech Preview] comes with kubectl 1.19:

```sh
❯ kubectl.docker version
Client Version: version.Info{Major:"1", Minor:"19", GitVersion:"v1.19.3", GitCommit:"1e11e4a2108024935ecfcb2912226cedeafd99df", GitTreeState:"clean", BuildDate:"2020-10-14T12:50:19Z", GoVersion:"go1.15.2", Compiler:"gc", Platform:"darwin/amd64"}
```

However, you can also use homebrew to install an Intel-emulated version of kubectl:

```
arch -x86_64 bash
export PATH="/usr/local/Homebrew/bin:$PATH"
brew install kubectl
```


## Resources

* https://isapplesiliconready.com


## Initial applications

* Slack via Mac App Store
* [VS Code Insiders](https://code.visualstudio.com/insiders/)