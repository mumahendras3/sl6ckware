# sl6ckware
## Using s6 + s6-rc as init system in Slackware

## NOTE
The **master** branch is for **Slackware -current**. Use the **14.2** branch for **Slackware 14.2**.

## TODO
### Converting to s6-rc service definitions
- rc.cgconfig
- rc.cgred
- rc.sysvinit
- rc.serial
- rc.M (genpowerd, accton, quota)
- rc.K
- rc.pcmcia
- rc.inet1
- rc.inet2
- rc.autofs
- rc.pulseaudio
- rc.dnsmasq
- rc.snmpd
- rc.atalk
- rc.smartd
- rc.saslauthd
- rc.postfix
- rc.alsa
- rc.keymap
- rc.mysqld
- rc.httpd
- rc.openldap
- rc.dovecot
- rc.samba
- rc.icecc-scheduler
- rc.iceccd
- rc.yp
- rc.6 (genpowerd, pppd, accton, quota)
- rc.4

### Create bundles
- runlevel 4 (graphical login)

### Others
- Make sure s6-linux-init-shutdownd writes a shutdown record to wtmp
- Do some renames to better match Slackware's rc scripts naming and
  and also some restructuring in the repo files to simplify things
