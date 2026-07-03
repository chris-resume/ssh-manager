# SSH Manager

## Description
**GNU/Linux-only** command line SSH manager with extra features, written in Bash.

Each configuration entry with a remote server's credentials is referred to as a **machine**.

## Dependencies
The following tools are strictly required:

* `bash ≥5`
* `coreutils`
* `bubblewrap`
* `fzf`
* `grep`
* `jq`
* `GNU sed`
* `ssh`
* `sshpass`

The following tools are required for extra features:

* `host` — for looking up IP address and hostname.
* `ping` — for pinging.
* `rsync` — for file transfer.
* `x2goclient` — for connecting with X2Go.

## Features
* SSH connection with passwordless `sudo` without modifying the remote server; the password on the remote server is also not visible in the process name (for example in `top`).
  * The `sudo` cached credentials are updated and the user authenticated once every 30 seconds.
  * The password is visible in the process name on the local machine, however.
  * If "Bash shell" is configured to be `true`, uses Bash *rcfile* file descriptor with the undermentioned code printed out. Additionally, if "passwordless sudo" is also configured to be `true`, the following extra functions are declared, which take a username (or a path with the username as its last component) as an argument, defaulting to `root`, and also encompass the described Bash *rcfile* file descriptor:
    * `.su` — for switching to another user with `sudo -Hu`.
    * `.si` — for switching to another user with `sudo -Hiu`.
```bash
. -- ~/.bashrc ; export LANG=en_US.UTF-8 XDG_RUNTIME_DIR=/run/user/$EUID ; HISTCONTROL=ignoreboth ; set +H ; shopt -s autocd cdspell dotglob extglob ; stty -echoctl ; function /() { cd /; } ; function ..() { cd ..; } ; function ...() { cd ../..; } ; function ....() { cd ../../..; } ; function .....() { cd ../../../..; } ; function -() { cd -; } ; bind 'set completion-ignore-case on' ; bind 'set mark-symlinked-directories on' ; bind 'set show-all-if-ambiguous on' ; bind 'set colored-stats on' ; alias l=ls ; alias la='ls -A' ; alias ll='ls -lh' ; alias lo='ls -Alh' ; alias trea='tree -a' ; export LESS=-i SYSTEMD_PAGER='less -i' ; alias sudo="env LESS=-i SYSTEMD_PAGER='less -i' sudo " ; PS1='$(if [[ $? -eq 0 ]]; then printf +; else printf -; fi) '$(bash -ic 'printf %s "'\$'PS1"') ; PROMPT_DIRTRIM=3 ; if [[ -f /usr/share/bash-completion/bash_completion ]]; then . /usr/share/bash-completion/bash_completion; elif [[ -f /etc/bash_completion ]]; then . /etc/bash_completion; fi
```
* SSH authorization with specific keyfiles (if configured for a given machine, otherwise password authorization).
* Creating, editing, and deleting machines in a JSON file, with custom names.
  * Asks interactively for user input.
  * Default values can be configured for SSH port, SSH username, SSH password, SSH keyfile, and X2Go session type.
* Listing a machine's connection configuration.
* Pinging a machine.
* File transfer to or from a machine with `rsync`.
  * Requires the "Bash shell" to be `true`.
  * `rsync` is run with `-aPh -e 'ssh -p PORT'` flags, where `PORT` is the SSH port.
    * Remote `rsync` is always run without `sudo`.
  * Can transfer multiple files or directories to a remote machine, or a single one from a remote machine to the local machine.
    * User-provided remote path `~` or starting with `~/` keeps a literal leading tilde.
  * On failure, will keep retrying after 2 seconds, indefinitely.
* Passwordless X2Go session for any user (except `root`), using a temporary SSH key (deleted on session end or error).
  * Requires the "Bash shell" and "passwordless sudo" to be `true`.
  * Requires a GUI session.
  * Asks for session type.
  * Uses 1920×1080 resolution.
  * Temporary files are kept in random-named directories in the `$TMPDIR` (if set) or `/tmp` (by default) directory.
* Last used machines are cached (listed at the beginning of the list, following a single empty entry), cache can be managed (remove a single or all entries).
* Lock-based JSON file and cache editing, using a lockfile.
  * Lockfile is in one of the following environment variable directories:
    * `$SSH_MANAGER_LOCKFILE_DIR`, if set.
    * `$XDG_RUNTIME_DIR`, by default.
      * If not set, defaults to `/run/user/$EUID`.
    * Lockfile directory must exist.
  * Lockfile is named `ssh-manager.$EUID.lock`.

## Configuration
* Download the `ssh-manager` file.
* Create a dedicated directory:
```bash
mkdir -m 700 ~/.ssh-manager
```
* Place the file there.
  * Make sure it is executable:
```bash
chmod 700 ~/.ssh-manager/ssh-manager
```
* Append the following to `~/.bashrc`:
```bash
bind '"\e\\": "\C-a ~/.ssh-manager/ssh-manager \C-j"'
```

## Files
The following files are created and used in the directory of the script's immediate (not following symlinks) filename:

* `ssh-manager.defaults` -- the default values for configuring new machines and for X2Go session type.
* `ssh-manager.credentials` -- a JSON file storing machine configurations.
* `ssh-manager.cache` -- a cache of recently invoked machines.
* `ssh-manager.x2go.key` -- an SSH private key file for connecting with X2Go.
* `ssh-manager.x2go.key.pub` -- an SSH public key file for connecting with X2Go.

## Usage
* Invoke `Alt+\` in the Bash shell.
  * The script will list missing commands that need to be installed, if any.
  * Further usage info is shown in the `fzf` chooser:
    * **Enter** or **Double-Click** — Connect with SSH.
    * **Alt+Q** — Remove from cache.
    * **Alt+V** — Clear cache.
    * **Alt+W** — Edit entry.
    * **Alt+E** — Configure new entry.
    * **Alt+A** — Show credentials.
    * **Alt+D** — Look up IP address and hostname.
    * **Alt+Z** — Ping.
    * **Alt+C** — File transfer.
    * **Alt+X** — Connect with X2Go.
    * **Esc** — Abort.
  * There are 3 types of `read` user input prompts:
    * **.>** -- Expects a literal string.
    * **->** -- Expects *a single* Bash-escaped word, with tab-completion available. If it turns out to be more than one word, only the first one is kept.
    * **=>** -- Expects *any number of* Bash-escaped words, with tab-completion available.
      * The `dotglob`, `extglob`, and `failglob` shell options are set.
      * Bash-escaped words are later unescaped in disposable sandboxed environments, to contain potential code injection.
  * If run with an argument, will interactively (re-)configure a machine with a given custom name (can be different than the hostname).
    * To do so, one can type a machine's custom name in shell prompt and then invoke `Alt+\`.
* Interactive configuration asks for:
  * custom name -- arbitrary, used only in the machine chooser;
  * SSH hostname;
  * SSH port;
  * SSH username;
  * SSH password;
  * SSH keyfile (`-` is password authentication; to use a file named as such, use `./-`);
  * whether to use passwordless `sudo`;
  * whether the remote shell is Bash -- must be `true` if passwordless sudo is `true`.
* The defaults file can be used to configure default values when configuring new machines, or asking for X2Go session type in the case of `x2go_st`. It should contain `key=value` pairs, not commented out with the `#` characters, one per line. The following keys are recognized:
  * **port** — SSH port.
    * *default value:* `22`
  * **username** — SSH username.
    * *default value:* `user`
  * **password** — SSH password.
    * *default value:* `qwerty`
  * **keyfile** — SSH private key file. Must be an absolute path. Path `~` (likely useless) or starting with `~/` is considered an absolute path. `-` is password authentication.
    * *default value:* `~/.ssh/id_rsa`
  * **x2go_st** — X2Go session type, that is: value to use for the `command` key in an X2Go configuration file. It is not configurable per machine and is instead asked for each time an X2Go connection is to be made.
    * *default value:* `XFCE`

## Caveats
* When using passwordless `sudo` or X2Go:
  * The remote server's operating system must be GNU/Linux.
  * The user's login shell must be Bash. Interpreting the sent Bash command line may work in other shells, but it is not at all guaranteed.
    * This does not apply to X2Go user's login shell.
  * `sudo` must be installed on the remote server.
    * The user's `sudo` password must be cached for at least 30 seconds on the remote server (the default is 5 minutes).
* If "Bash shell" is configured to be `true`, then due to the fact that some commands are automatically executed on the remote server, it is assumed that the remote `~/.bashrc` files are trusted and the remote users' `$PATH` variables are sane.
* Literal newline characters in credentials or user input, excluding the resulting glob expansions from **=>** `read` prompts, are unsupported.
* Lockfile may become stale, for example due to SIGKILL or power loss. In that case, it must be removed manually.
