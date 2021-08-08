Copy `templates/openvpn-vpnname-srv` and `templates/openvpn-vpnname-log`
to this directory and replace all instances of `vpnname` (including the
services name) with the name of your VPN (or anything really). We will
use `myvpn` here as an example.

```sh
cp -r templates/openvpn-vpnname-srv source/rc.openvpn/openvpn-myvpn-srv
cp -r templates/openvpn-vpnname-log source/rc.openvpn/openvpn-myvpn-log
cd source/rc.openvpn
sed -i 's/vpnname/myvpn/g' $(grep -rl vpnname openvpn-myvpn-*)
```

Next, edit `openvpn-myvpn-srv/env/CONF_PATH` and make sure it points to
the location of your VPN's OpenVPN config file.

Lastly, add `openvpn-myvpn` (without the `-srv` or `-log` suffix) to
`source/bundles/openvpn/contents` if you want to start it at boot time.

```sh
echo openvpn-myvpn >> source/bundles/openvpn/contents
```
