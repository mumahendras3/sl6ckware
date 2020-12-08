# sl6ckware - Using s6 + s6-rc + s6-linux-init as the init system on Slackware

## NOTE
* This repo is intended to be used on Slackware-current (for now) so some adjustments need to be done to some of the s6-rc service definition source directories in order to make it work on Slackware 14.2.
* Before using this repo, read the [s6](https://skarnet.org/software/s6/), [s6-rc](https://skarnet.org/software/s6-rc/), and [s6-linux-init](https://skarnet.org/software/s6-linux-init/) home page first to have a better understanding of what are they and what features or advantages that they provide compared to other init systems. Or for a quick introduction in video format, see [this video](https://www.youtube.com/watch?v=gZqIEstv5lM).

## REQUIREMENTS
1. skalibs
2. execline
3. s6
4. s6-rc
5. s6-linux-init

All of them are available on [SBo](https://slackbuilds.org/).

## SETUP
### Compiling a new s6-rc service database
1. Clone this repo.

2. Create a copy of the `source` directory from this repo and put it somewhere else in your system (e.g. `/etc/s6/rc/source`).

3. Edit these two files (all paths below are relative to the `source` directory that you just copied):
    1. `bundles/sysinit/contents`: this file contains a list of services that are needed to initialize the system, just like the `rc.S` script. In this file you can (for example) uncomment `font` service to load a custom screen font for the virtual console (see `rc.font/font/up` for instructions on how to load your desired custom screen font).
    2. `bundles/multi-user/contents`: this file contains a list of services that will be started when entering one of the multi user runlevels (runlevel 2, 3, 4, and 5 by default), just like the `rc.M` script. In this file you can (for example) uncomment `cpufreq` and `bluetooth` service to set the default cpu governor and enable bluetooth support, respectively, at boot time.

4. Feel free to edit all other files inside the `source` directory beside those two files above.

5. Use `s6-rc-compile` (more details are available in `/usr/doc/s6-rc-<version>/doc/s6-rc-compile.html`) to compile a new s6-rc service database from your `source` directory that will be used by s6-rc at runtime to manage services:

    ```bash
    s6-rc-compile /etc/s6/rc/compiled /etc/s6/rc/source/*
    ```

    Explanation:
    * `/etc/s6/rc/compiled`: tell `s6-rc-compile` to put the compiled database data in `/etc/s6/rc/compiled` directory (it must not exist beforehand). Of course you can change `/etc/s6/rc/compiled` to other places that you want.
    * `/etc/s6/rc/source/*`: tell `s6-rc-compile` to search for service definition source directories in all subdirectories of `/etc/s6/rc/source` directory (notice the trailing asterisk). Of course you should change `/etc/s6/rc/source` to the path of your own copy of `source` directory.

### Setting up s6-linux-init
1. Run the `s6-linux-init-maker` command (more details are available in `/usr/doc/s6-linux-init-<version>/doc/s6-linux-init-maker.html`) to create all necessary files that the system needs to be able to properly boot and bring up a full s6 infrastructure. Below is the `s6-linux-init-maker` invocation that I use for my system (feel free to change them and add any other options that suit your needs):

    ```bash
    s6-linux-init-maker -c /etc/s6/init/current -u adm -G "agetty 38400 tty12 linux" -1 -p "/usr/bin:/usr/sbin:/bin:/sbin" -t 2 -D 4 /etc/s6/init/current
    ```

    Explanation:
    * `-c /etc/s6/init/current`: tell `s6-linux-init` to look inside `/etc/s6/init/current` at system startup for all the necessary files it needs in order to boot the system properly. We will reference this directory as *basedir*.
    * `-u adm`: the catch-all logger will run as the `adm` user. The catch-all logger is a service that is started early at boot time to log all system startup and shutdown time messages (except for kernel messages) to the logging directory `/run/uncaught-logs` (or `tmpfsdir/uncaught-logs` where *tmpfsdir* is the argument that was given to the `--tmpfsdir=` configure option at s6-linux-init build time). This is made possible by using `s6-log` which takes inputs from stdin and processes them based on the logging script given to it (like adding timestamps, deselect certain lines, etc.). See `/usr/doc/s6-<version>/doc/s6-log.html` for more details on `s6-log`.
    * `-G "agetty 38400 tty12 linux"`: start an early getty service by executing the given command line (`agetty 38400 tty12 linux`). The command line `agetty 38400 tty12 linux` will start an `agetty` instance that connects to port `/dev/tty12` with baud rate `38400` and `TERM=linux`.
    * `-1`: make it so that all the messages that are sent to the catch-all logger are also copied to `/dev/console` (without timestamps) so those messages will also appear on the computer screen just like Slackware's default init system but better since those messages can still be accessed even after the system has finished booting.
    * `-p "/usr/bin:/usr/sbin:/bin:/sbin"`: set the initial value of the `PATH` environment variable to `/usr/bin:/usr/sbin:/bin:/sbin`. All s6-rc service definitions in this repo do not use absolute path when invoking a program so at least these directories should be added to the initial `PATH` value.
    * `-t 2`: set the catch-all logger timestamp format to [ISO 8601 format](http://www.iso.org/iso/home/standards/iso8601.htm).
    * `-D 4`: set the default runlevel to runlevel `4`.
    * `/etc/s6/init/current`: tell `s6-linux-init-maker` to put the generated files inside `/etc/s6/init/current` directory (this directory must not exist beforehand). You can change this argument to other places that you want (e.g. `/tmp/init-current`) but don't forget to move/copy the generated files back to the declared *basedir* (`/etc/s6/init/current` in my case). Use `cp -a` when copying the generated files since there are fifos, files with precise uid/gid permissions, and files with non-standard access rights, so be sure to copy it verbatim.

2. Edit `rc.init`, `rc.shutdown`, and `runlevel` script inside `basedir/scripts` to incorporate below changes:
    1. `rc.shutdown`: uncomment `exec s6-rc -v2 -bda change` since we're going to use s6-rc as the service manager.
    2. `rc.init`:
        1. uncomment and change `s6-rc-init /run/service` to `s6-rc-init -c /etc/s6/rc/compiled /run/service`. Change `/etc/s6/rc/compiled` to the path of your new s6-rc service database directory that you have compiled before. Also if you use different *tmpfsdir* when compiling s6-linux-init, change `/run` to your *tmpfsdir*.
        2. uncomment and change `exec /etc/s6-linux-init/current/scripts/runlevel "$rl"` to `s6-rc -v2 -up change "$rl"; exec s6-rc -v1 -u change gettys`. The `s6-rc -v2 -up change "$rl"` part will bring the system up to the desired runlevel and the `exec s6-rc -v1 -u change gettys` part will start all configured gettys (from tty1 to tty6, just like Slackware's default behaviour). By running `s6-rc -v2 -up change "$rl"` first and then followed by `exec s6-rc -v1 -u change gettys`, the login prompt at tty1 will not be covered by boot time messages that also appear on tty1.
    3. `runlevel`: uncomment and change `exec s6-rc -v2 -up change "$1"` to `exec s6-rc -v2 -up change gettys "$1"`. This will make sure the already running gettys are not stopped when changing from one runlevel to other runlevel.

3. Lastly, make a backup copy of `/sbin/halt`, `/sbin/init`, `/sbin/poweroff`, `/sbin/reboot`, `/sbin/shutdown`, and `/sbin/telinit`. Then, copy all scripts inside `basedir/bin` to `/sbin` (or alternatively, you can just create symbolic links that point to each scripts inside `basedir/bin`).

## TIPS
* The default init system and the s6-based init system can actually coexist by copying/linking the scripts inside `basedir/bin` to `/sbin` but using different names (e.g. instead of `init`, rename it to `init.s6`). Then, when you want to boot to the s6-based init system, pass `init=/sbin/init.s6` (for example) to the kernel as a command line argument. This is mainly useful for testing the s6-based init system before using it as the main init system.
* It's a good idea to use a time-based naming scheme for the compiled s6-rc service database and then create a symbolic link with generic name that points to the service database that you want to use by default at system startup. For example, each time you compile a new database, you run `s6-rc-compile compiled-$(date +%y%m%d-%R:%S) source/*`. Then, if you want to make it the default service database, make a symbolic link named `compiled` that points to it. With this approach, you don't have to modify `basedir/scripts/rc.init` every time you change your default service database.
* If you use symbolic link for the default service database (like above), you probably want to add the `-d` option to `s6-rc-init` invocation in `basedir/scripts/rc.init`. The `-d` option will tell `s6-rc-init` to dereference the argument that is given to the `-c` option (`/etc/s6/rc/compiled` in my example above) and use that instead. The symbolic link will then not be used by s6-rc and thus can be updated freely without affecting the currently running system. Without the `-d` option, you have to run `s6-rc-update another_compiled_database_absolute_path` first before updating the symbolic link.

## QUICK USER GUIDE
* Start a service: `s6-rc change service_name`
* Stop a service: `s6-rc -d change service_name`
* Restart a longrun service: `s6-svc -wr /run/service/service_name`
* List all active services: `s6-rc -a list`
* List all nonactive services: `s6-rc -da list`
* Switch to another compiled service database: `s6-rc-update another_compiled_database_absolute_path`

## SERVICES NOT YET CONVERTED
Below rc scripts are not yet converted to s6-rc service definition:
- rc.cgconfig
- rc.cgred
- rc.sysvinit
- rc.serial
- rc.M (genpowerd, accton, quota)
- rc.pcmcia
- rc.inet1
- rc.inet2
- rc.autofs
- rc.alsa
- rc.mysqld
- rc.httpd
- rc.icecc-scheduler
- rc.iceccd
- rc.yp
- rc.6 (genpowerd, pppd, accton, quota)
