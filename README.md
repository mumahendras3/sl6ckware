# sl6ckware
## Using s6 + s6-rc as init system in Slackware

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
- rc.local
- rc.6 (for all oneshots down scripts)

### Create the -log services
- haveged
- udevd
- cgmanagerd
- klogd
- syslogd
- udevd
- system-dbusd
- networkmanagerd
- cupsd
- crond
- atd
- ntpd
- acpid
- bluetoothd
- consolekitd

### Create bundles
- runlevel 1 (single user)
- runlevel 4 (graphical login)

### Create additional services
- acpid-socket (create a socket and pass it as acpid stdin for faster startup and reliable readiness indicator)
