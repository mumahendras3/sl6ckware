# Sl6ckware

Using s6 + s6-rc + s6-linux-init as the init system on Slackware.

## Note

- This repo is intended to be used on Slackware-current ("current" branch) and
  Slackware 15.0 ("15.0" branch).

- Before using this repo, read the [s6][], [s6-rc][], and [s6-linux-init][] home
  page first to have a better understanding of what they are and what features
  or advantages that they provide compared to other init systems. Or for a quick
  introduction in video format, see [this video][intro-video].

[s6]: https://skarnet.org/software/s6/
[s6-rc]: https://skarnet.org/software/s6-rc/
[s6-linux-init]: https://skarnet.org/software/s6-linux-init/
[intro-video]: https://www.youtube.com/watch?v=gZqIEstv5lM

## Requirements

- skalibs
- execline
- s6
- s6-rc
- s6-linux-init

All of them are available on [SBo][sbo].

[sbo]: https://slackbuilds.org/

## Setup

### Compiling a new s6-rc service database

1. Clone this repo.

2. Create a copy of the `source` directory from this repo and put it somewhere
   else in your system (e.g., `/etc/s6/rc/source`).

3. Edit these two files (all paths below are relative to the `source` directory
   that you just copied):

   - `bundles/S/contents`: this file contains a list of services that are needed
     to initialize the system, just like the `rc.S` script. In this file you can
     (for example) uncomment `font` service to load a custom screen font for the
     virtual console (see `rc.font/font/up` for instructions on how to load your
     desired custom screen font).

   - `bundles/M/contents`: this file contains a list of services that will be
     started when entering one of the multiuser runlevels (runlevel 2, 3, 4, and
     5 by default), just like the `rc.M` script. In this file you can (for
     example) uncomment `cpufreq` and `bluetooth` service to set the default CPU
     governor and enable Bluetooth support, respectively, at boot time.

4. Feel free to edit all other files inside the `source` directory beside those
   two files.

5. Use `s6-rc-compile` (more details are available in
   `/usr/doc/s6-rc-<version>/doc/s6-rc-compile.html`) to compile a new s6-rc
   service database from your `source` directory that will be used by s6-rc at
   runtime to manage services:

   ```sh
   s6-rc-compile /etc/s6/rc/compiled /etc/s6/rc/source/*
   ```

   - `/etc/s6/rc/compiled`: tell `s6-rc-compile` to put the compiled database
     data in the directory `/etc/s6/rc/compiled` (it must not exist beforehand).
     Of course, you can change `/etc/s6/rc/compiled` to other places that you
     want.

   - `/etc/s6/rc/source/*`: tell `s6-rc-compile` to search for service
     definition source directories in all subdirectories of `/etc/s6/rc/source`
     (notice the trailing asterisk). Of course, you should change
     `/etc/s6/rc/source` to the path of your own copy of `source` directory.

### Setting up s6-linux-init

1. Use `s6-linux-init-maker` (more details are available in
   `/usr/doc/s6-linux-init-<version>/doc/s6-linux-init-maker.html`) to create
   all the necessary files that the system needs to be able to properly boot and
   bring up a full s6 infrastructure. Below is the `s6-linux-init-maker`
   invocation that I use for my system (feel free to change them and add any
   other options that suit your needs):

   ```sh
   s6-linux-init-maker \
       -f ~/Documents/sl6ckware/skel \
       -c /etc/s6/init/current \
       -u adm \
       -G "agetty 38400 tty12 linux" \
       -1 \
       -p "/bin:/sbin:/usr/bin:/usr/sbin" \
       -t 2 \
       -D 4 \
       /etc/s6/init/current
   ```

   - `-f ~/Documents/sl6ckware/skel`: tell `s6-linux-init` to use the skeleton
     scripts provided by this repo (i.e., `skel/rc.init`, `skel/rc.shutdown`,
     `skel/rc.shutdown.final`, and `skel/runlevel`) instead of the default
     s6-linux-init skeleton scripts (don't forget to change
     `~/Documents/sl6ckware` to the path where you clone this repo into).

   - `-c /etc/s6/init/current`: tell `s6-linux-init` to look inside
     `/etc/s6/init/current` at system startup for all the necessary files it
     needs in order to boot the system properly. We will reference this
     directory as *basedir*.

   - `-u adm`: the catch-all logger will run as the `adm` user. The catch-all
     logger is a service that is started early at boot time to log all system
     startup and shutdown time messages (except for kernel messages) to the
     logging directory `/run/uncaught-logs` (or `tmpfsdir/uncaught-logs` where
     *tmpfsdir* is the argument that was given to the `--tmpfsdir=` configure
     option at s6-linux-init build time). This is made possible by using
     `s6-log` which takes inputs from stdin and processes them based on the
     logging script given to it (like adding timestamps, deselect certain lines,
     etc.). See `/usr/doc/s6-<version>/doc/s6-log.html` for more details on
     `s6-log`.

   - `-G "agetty 38400 tty12 linux"`: start an early getty service by executing
     the given command line (`agetty 38400 tty12 linux`). The command line
     `agetty 38400 tty12 linux` will start an `agetty` instance that connects to
     port `/dev/tty12` with baud rate `38400` and `TERM=linux`.

   - `-1`: make it so that all the messages that are sent to the catch-all
     logger are also copied to `/dev/console` (without timestamps). This will
     make those messages also appear on the computer screen just like
     Slackware's default init system but better, since those messages can still
     be accessed later after the system has finished booting.

   - `-p "/bin:/sbin:/usr/bin:/usr/sbin"`: set the initial value of the `PATH`
     environment variable to `/bin:/sbin:/usr/bin:/usr/sbin`. Almost all s6-rc
     service definitions in this repo do not use absolute path when invoking a
     program so at least these directories should be added to the initial `PATH`
     value.

   - `-t 2`: use the [ISO 8601 format][iso8601] for the catch-all logger's
     timestamp format.

   - `-D 4`: set the default runlevel to runlevel `4`.

   - `/etc/s6/init/current`: tell `s6-linux-init-maker` to put the generated
     files inside `/etc/s6/init/current` (this directory must not exist
     beforehand). You can change this argument to other places that you want
     (e.g., `/tmp/init-current`) but don't forget to move/copy the generated
     files back to the declared *basedir* (`/etc/s6/init/current` in my case).
     Use `cp -a` when copying the generated files since there are FIFOs, files
     with precise UID/GID permissions, and files with non-standard access
     rights, so be sure to copy it verbatim.

2. Edit `bin/init` inside your *basedir* and change `s6-linux-init` to
   `/sbin/s6-linux-init` since the `PATH` environment variable is undefined at
   early boot time. This is not necessary if you add `/sbin` to the default
   executable search path when compiling skalibs (see its [SBo README
   file][ssrf] for more details on how to do that).

3. Lastly, make a backup copy of `/sbin/halt`, `/sbin/init`, `/sbin/poweroff`,
   `/sbin/reboot`, `/sbin/shutdown`, and `/sbin/telinit`. Then, copy all scripts
   inside your *basedir*'s `bin` directory to `/sbin` (or alternatively, you can
   just create symbolic links inside `/sbin` that point to each script).

[iso8601]: http://www.iso.org/iso/home/standards/iso8601.htm
[ssrf]: https://slackbuilds.org/slackbuilds/15.0/libraries/skalibs/README

## Tips

- The default init system and the s6-based init system can actually coexist by
  using different names when copying/linking the aforementioned scripts (e.g.,
  instead of `/sbin/init`, use `/sbin/init.s6`). Then, when you want to use the
  s6-based init system, pass `init=/sbin/init.s6` (following the example before)
  to the kernel as a command line argument. This is mainly useful for testing
  the s6-based init system before using it as the main init system.

- Use a symbolic link to manage the default service database that s6-rc uses at
  system startup. For example, each time you compile a new database, you run:

  ```sh
  s6-rc-compile "compiled-$(date +%y%m%d-%R:%S)" source/*
  ```

  Then, if you want to make it the default service database, make a symbolic
  link named `compiled` that points to it. With this approach, you don't have to
  modify your *basedir*'s `scripts/rc.init` script every time you change your
  default service database.

- If you applied the previous tip, you probably want to modify your *basedir*'s
  `scripts/rc.init` script and add the `-d` option to the `s6-rc-init`
  invocation. The `-d` option will tell `s6-rc-init` to dereference the argument
  that is given to the `-c` option (`/etc/s6/rc/compiled` in my example above)
  and use that instead. The symbolic link will then not be used by s6-rc and
  thus can be updated freely without affecting the currently running system.
  Without the `-d` option, you have to run `s6-rc-update
  <another_compiled_database_absolute_path>` first before updating the symbolic
  link.

## Quick user guide

- Start a service:

  ```sh
  s6-rc -u change <service_name>
  ```

- Stop a service:

  ```sh
  s6-rc -d change <service_name>
  ```

- Restart a longrun service:

  ```sh
  s6-svc -r /run/service/<service_name>
  ```

- List all active services:

  ```sh
  s6-rc -a list
  ```

- List all nonactive services:

  ```sh
  s6-rc -da list
  ```

- Switch to another compiled service database:

  ```sh
  s6-rc-update <another_compiled_database_absolute_path>
  ```

## Per-user supervision tree

To have a per-user supervision tree running under the system's
supervision tree, we need to create a dedicated system service that runs
an instance of `s6-svscan` as a normal user. Each user wanting to have a
personal supervision tree will need to have this kind of service created
for them. Luckily, there is the `s6-usertree-maker` program that we can
use to easily create it.

### Creating your user services

Before creating a per-user supervision tree service (using
`s6-usertree-maker`) that will run as a system service, you need to
create your own user services first and test them. If you don't need
dependencies handling between your services, you don't need s6-rc. Just
create a new scan directory containing the service directories that you
have made and put it somewhere accessible by your user, like
`/home/<your_username>/.config/s6/service`. Try running `s6-svscan
<your_scan_directory>` directly on your terminal to test out the
services you have made. If everything is working correctly, continue to
the next section. Personally, I use s6-rc to manage all my user services
and I don't create a scan directory myself. Rather, I let the per-user
supervision tree service creates an empty scan directory
`/run/user/${UID}/s6/service` at runtime, as you will see later in the
next section. Some of the programs that I run under this per-user
supervision tree are `ssh-agent`, `dbus-daemon`, `pipewire`,
`pipewire-media-session`, and `pipewire-pulse`. See my s6-rc source
definition directories for these programs
[here](https://github.com/mumahendras3/configs-scripts/tree/master/s6/rc/source).

### Creating the per-user supervision tree service

The `s6-usertree-maker` program provides many options that you can use
to customize the generated per-user supervision tree service, see
`/usr/doc/s6-<version>/doc/s6-usertree-maker.html` for more details. As
an example, this is what I use on my system:

```sh
s6-usertree-maker \
    -d '/run/user/${UID}/s6/service' \
    -p '/usr/local/bin:/usr/bin:/bin' \
    -E env \
    -e XDG_CACHE_HOME \
    -e XDG_CONFIG_HOME \
    -e XDG_DATA_HOME \
    -e XDG_RUNTIME_DIR \
    -e XDG_STATE_HOME \
    -r usertree-${USER}-srv/usertree-${USER}-log/usertree-${USER} \
    -n 4 \
    -s 500000 \
    -t 2 \
    $USER /var/log/usertree-${USER} /etc/s6/rc/source/usertrees
```

Explanation:

- `-d '/run/user/${UID}/s6/service'`: the per-user supervision tree will
  be run on the directory `/run/user/${UID}/s6/service`, which is
  subject to variable substitution (notice that I use single quote here
  to prevent `${UID}` from being substituted by the shell).

- `-p '/usr/local/bin:/usr/bin:/bin'`: the per-user supervision tree
  will be run with a PATH environment variable set to
  `/usr/local/bin:/usr/bin:/bin`, which is also subject to variable
  substitution (although, in this case, no variable is actually being
  used). You might want to add something like `${HOME}/.local/bin` to
  that list if you have some programs in your home directory.

- `-E env`: the per-user supervision tree will be run with the
  environment variables defined in the directory `env`, which will be
  read via s6-envdir without options. Since it's not using an absolute
  path, this `env` directory will need to be located inside the
  generated service directory. If you have more than one user
  supervision trees, you might want to set it to something like
  `/etc/user-env` so all of them can share the same set of environment
  variables.

- `-e XDG_CACHE_HOME`, `-e XDG_CONFIG_HOME`, `-e XDG_DATA_HOME`, `-e
  XDG_RUNTIME_DIR`, `-e XDG_STATE_HOME`: perform variable substitution
  on `XDG_CACHE_HOME`, `XDG_CONFIG_HOME`, `XDG_DATA_HOME`,
  `XDG_RUNTIME_DIR`, and `XDG_STATE_HOME`. That is, if these environment
  variables contain `${USER}`, `${HOME}`, `${UID}`, `${GID}`, and
  `${GIDLIST}`, they will be substituted with their respective values.

- `-r usertree-${USER}-srv/usertree-${USER}-log/usertree-${USER}`:
  create s6-rc source definition directories instead of a regular s6
  service directory, where the per-user supervision tree service will be
  named `usertree-<your_username>-srv`, its logger will be named
  `usertree-<your_username>-log`, and a bundle containing both of them
  will be named `usertree-<your_username>`.

- `-n 4`, `-s 500000`, `-t 2`: use `n4 s500000 T` as the control
  directives for the `s6-log` instance that will log the per-user
  supervision tree service.

- `$USER`: the per-user supervision tree will be run as the user
  `<your_username>`.

- `/var/log/usertree-${USER}`: the per-user supervision tree logs will
  be saved in the directory `/var/log/usertree-<your_username>`.

- `/etc/s6/rc/source/usertrees`: put the generated s6-rc source
  definition directories in the directory `/etc/s6/rc/source/usertrees`.

After running that command, you can inspect the generated s6-rc source
definition directories and modify them if necessary. As an example, on
my system I have to add this small change to the generated
`usertree-<your_username>-srv/run` script:

```diff
--- a/usertree-<your_username>-srv/run     2022-08-18 00:18:42.986512403 +0700
+++ b/usertree-<your_username>-srv/run     2022-08-18 01:35:12.639851358 +0700
@@ -28,4 +28,6 @@
 export "XDG_RUNTIME_DIR" ${"XDG_RUNTIME_DIR"}
 export "XDG_STATE_HOME" ${"XDG_STATE_HOME"}
 export PATH "/usr/local/bin:/usr/bin:/bin"
+# Creating an empty scan directory since all services will be handled by s6-rc
+if { mkdir -p "/run/user/${UID}/s6/service" }
 s6-svscan -d3 -- "/run/user/${UID}/s6/service"
```

Lastly, recompile your system's s6-rc service database to include these
new services.

### Starting the per-user supervision tree service

Probably the best way to start this service is by dynamically starting
it upon the user's first log-in and stopping it upon the user's last
log-out. To achieve this, I use the PAM module `pam_exec` that executes
two scripts: `pam-start-usertree` when a user logs in and
`pam-stop-usertree` when a user logs out. Here are the diffs that I have
applied to `/etc/pam.d/login` and `/etc/pam.d/sddm` on my system:

- `/etc/pam.d/login`:

  ```diff
  --- a/etc/pam.d/login   2020-06-19 04:01:47.000000000 +0700
  +++ b/etc/pam.d/login   2022-08-17 16:58:49.716526301 +0700
  @@ -17,4 +17,18 @@
   session         include         postlogin
   session         required        pam_loginuid.so
   -session        optional        pam_ck_connector.so nox11
  +# Skip the pam-stop-usertree script for non-regular users
  +session         [success=ok default=1] pam_succeed_if.so quiet uid >= 1000 \
  +                                                         uid <= 60000
  +# Running the pam-stop-usertree script before $XDG_RUNTIME_DIR is deleted by
  +# pam_elogind
  +session         optional        pam_exec.so type=close_session \
  +                                            /usr/local/sbin/pam-stop-usertree
   -session        optional        pam_elogind.so
  +# Skip the pam-start-usertree script for non-regular users
  +session         [success=ok default=1] pam_succeed_if.so quiet uid >= 1000 \
  +                                                         uid <= 60000
  +# Running the pam-start-usertree script after $XDG_RUNTIME_DIR is set and
  +# created by pam_elogind
  +session         optional        pam_exec.so type=open_session \
  +                                            /usr/local/sbin/pam-start-usertree
  ```

- `/etc/pam.d/sddm`:

  ```diff
  --- a/etc/pam.d/sddm    2022-01-16 12:12:19.000000000 +0700
  +++ b/etc/pam.d/sddm    2022-08-17 16:58:49.716526301 +0700
  @@ -20,7 +20,21 @@
   session    substack     system-auth
   session    required     pam_loginuid.so
   -session   optional     pam_ck_connector.so      nox11
  +# Skip the pam-stop-usertree script for non-regular users
  +session    [success=ok default=1] pam_succeed_if.so quiet uid >= 1000 \
  +                                                    uid <= 60000
  +# Running the pam-stop-usertree script before $XDG_RUNTIME_DIR is deleted by
  +# pam_elogind
  +session    optional     pam_exec.so type=close_session \
  +                                    /usr/local/sbin/pam-stop-usertree
   -session   optional     pam_elogind.so
  +# Skip the pam-start-usertree script for non-regular users
  +session    [success=ok default=1] pam_succeed_if.so quiet uid >= 1000 \
  +                                                    uid <= 60000
  +# Running the pam-start-usertree script after $XDG_RUNTIME_DIR is set and
  +# created by pam_elogind
  +session    optional     pam_exec.so type=open_session \
  +                                    /usr/local/sbin/pam-start-usertree
   -session   optional     pam_gnome_keyring.so     auto_start
   -session   optional     pam_kwallet5.so          auto_start
   session    include      postlogin
  ```

As for the scripts themselves, here are what I use:

- `pam-start-usertree`:

  ```sh
  #!/bin/sh -e
  #
  # Start s6-based user services upon first login
  #
  
  # Setting $PATH first since it's probably undefined
  export PATH="/usr/bin:/bin:/usr/sbin:/sbin"
  
  # Defining some important constants that will be used later
  SYSTEM_SCANDIR="$(xargs -0a /proc/1/cmdline | grep -o ' /.*$' | cut -c 2-)"
  USERTREE_SVNAME="usertree-$PAM_USER"
  USERTREE_SVDIR="${SYSTEM_SCANDIR}/${USERTREE_SVNAME}-srv"
  S6RC_TIMEOUT="5000"
  USER_SCANDIR="${XDG_RUNTIME_DIR}/s6/service"
  USER_S6RC_LIVE="${XDG_RUNTIME_DIR}/s6/rc"
  USER_S6RC_BUNDLE="${XDG_SESSION_TYPE}-session" # unspecified/tty/x11/wayland/mir
  USER_S6RC_COMPILED="/home/${PAM_USER}/.config/s6/rc/compiled"
  
  # Aborting execution when not invoked by pam_exec
  if [ -z "$PAM_SERVICE" ]; then
      echo "We are not invoked by pam_exec, aborting..."
      exit 1
  fi
  
  # Exiting immediately if PID 1 is not s6-svscan
  grep -Fqwz s6-svscan /proc/1/cmdline || exit 0
  
  # Exiting immediately if the user doesn't have a supervision tree configured
  [ -d "$USERTREE_SVDIR" ] || exit 0
  
  # Checking if this is not the user's first session
  if [ $(loginctl -p Sessions --value show-user "$PAM_USER" | wc -w) -gt 1 ]; then
      # Exiting immediately if the user doesn't use s6-rc
      [ -d "$USER_S6RC_LIVE" ] || exit 0
      # Starting the appropriate user configured bundle (based on this session's
      # type) again in case the current and last sessions' type are different
      exec s6-setuidgid "$PAM_USER" \
          s6-rc -bl "$USER_S6RC_LIVE" -t "$S6RC_TIMEOUT" \
              change "$USER_S6RC_BUNDLE"
  fi
  
  # Starting the user's supervision tree
  s6-rc -bt "$S6RC_TIMEOUT" change "$USERTREE_SVNAME"
  
  # No need to continue further if the user doesn't use s6-rc
  [ -d "$USER_S6RC_COMPILED" ] || exit 0
  
  # Initializing s6-rc for the user's supervision tree
  s6-setuidgid "$PAM_USER" \
      s6-rc-init -dc "$USER_S6RC_COMPILED" -l "$USER_S6RC_LIVE" \
          -t "$S6RC_TIMEOUT" "$USER_SCANDIR"
  
  # Starting the appropriate user configured bundle (based on this session's type)
  exec s6-setuidgid "$PAM_USER" \
      s6-rc -l "$USER_S6RC_LIVE" -t "$S6RC_TIMEOUT" change "$USER_S6RC_BUNDLE"
  ```

- `pam-stop-usertree`:

  ```sh
  #!/bin/sh
  #
  # Stop s6-based user services upon last logout
  #
  
  # Setting $PATH first since it's probably undefined
  export PATH="/usr/bin:/bin:/usr/sbin:/sbin"
  
  # Defining some important constants that will be used later
  SYSTEM_SCANDIR="$(xargs -0a /proc/1/cmdline | grep -o ' /.*$' | cut -c 2-)"
  USERTREE_SVNAME="usertree-$PAM_USER"
  USERTREE_SVDIR="${SYSTEM_SCANDIR}/${USERTREE_SVNAME}-srv"
  S6RC_TIMEOUT="5000"
  USER_S6RC_LIVE="${XDG_RUNTIME_DIR}/s6/rc"
  
  # Aborting execution when not invoked by pam_exec
  if [ -z "$PAM_SERVICE" ]; then
      echo "We are not invoked by pam_exec, aborting..."
      exit 1
  fi
  
  # Exiting immediately if PID 1 is not s6-svscan
  grep -Fqwz s6-svscan /proc/1/cmdline || exit 0
  
  # Exiting immediately if the user doesn't have a supervision tree configured
  [ -d "$USERTREE_SVDIR" ] || exit 0
  
  # Exiting immediately if this is not the user's last session
  [ $(loginctl -p Sessions --value show-user "$PAM_USER" | wc -w) -gt 1 ] && \
      exit 0
  
  # Using s6-rc to bring down all the user's services (if applicable)
  [ -d "$USER_S6RC_LIVE" ] && s6-setuidgid "$PAM_USER" \
      s6-rc -Dal "$USER_S6RC_LIVE" -t "$S6RC_TIMEOUT" change
  
  # Stopping the user's supervision tree
  exec s6-rc -bdt "$S6RC_TIMEOUT" change "$USERTREE_SVNAME"
  ```

These scripts might be overkill for most people, since they also handle
more than just starting and stopping the per-user supervision tree
service. The `pam-start-usertree` script handles initializing s6-rc for
the user (if it's used) and starting different user s6-rc bundles
depending on the session type (logging in to a graphical or TTY session
will start a different set of services). Then, upon the user's last
log-out, the `pam-stop-usertree` script handles stopping all s6-rc
managed user services that are still active.

## Services not yet converted

Below rc scripts are not yet converted to s6-rc service definition:

- `rc.sendmail`
- `rc.6` (`pppd` related stuff)

## Notice of non-affiliation and disclaimer

Sl6ckware is not affiliated, associated, endorsed by, or in any way
officially connected with Slackware. The official Slackware website can
be found at <http://www.slackware.com/>.
