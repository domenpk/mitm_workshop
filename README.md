# Setting up a man-in-the-middle attack

## Internet through ethernet

Disconnect from current WiFi connection, and connect ethernet cable.

With your package manager, install **wireshark** and **mitmproxy**


## WiFi Access Point

### Easy option: Network Manager

In nm-applet (or nmtui) click *Edit Connections...*, *Add*

And select Wi-Fi:

![nm-applet Connection Type](https://raw.githubusercontent.com/domenpk/mitm_workshop/master/pics/nm_wifi1.png)


Name your SSID mitm-something, *Mode* should be **hotspot**.

![nm-applet Editing connection](https://raw.githubusercontent.com/domenpk/mitm_workshop/master/pics/nm_wifi2.png)


You should probably also add a WPA2 password.

Does it not work? Check if WiFi interface supports AP mode:
```
# iw list | grep -A 10 modes:  # is there "AP"?
```


### Hard option: Manual

Network Manager or similar managers must not be running.

Your upstream (internet) interface is eth0, and AP interface is wlan0 (change these if needed).
```
# upstream=eth0
# ap=wlan0
```

#### WiFi AP (hostapd)

Download [hostapd.conf](https://github.com/domenpk/mitm_workshop/blob/master/configs/hostapd.conf).
Change *interface*, *ssid* and *wpa_passphrase*.

Run
```
# hostapd ./hostapd.conf
```

SSID should now appear.

(To make it permanent, you could add this hostapd.conf to /etc/network/interfaces.)

#### Configure IP and dhcpd

I'll use subnet 192.168.11.0/24 here. Change everywhere if you intent to use something else.

```
# apnet=192.168.11
# ip addr add $apnet.1/24 dev $ap
# dnsmasq --interface=$ap --dhcp-range=$apnet.100,$apnet.199
```

#### NAT (Network Address Translation)

Just giving client the IP is not enough, you also need to set up IP forwarding, and rewriting (upstream routers don't know about your $apnet).

```
# echo 1 > /proc/sys/net/ipv4/ip_forward
# iptables -t nat -A POSTROUTING -s $apnet.0/24 ! -d $apnet.0/24 -j MASQUERADE
```

To revert settings:
```
# echo 0 > /proc/sys/net/ipv4/ip_forward
# iptables -t nat -D POSTROUTING -s $apnet.0/24 ! -d $apnet.0/24 -j MASQUERADE
```
Kill `dnsmasq`, `hostapd`.


### Hard option: Plan B, help of additional WiFi router

(Use this in case you WiFi network card doesn't support AP mode)

WiFi router is configured as a plain router, and will serve as WiFi AP. Its WAN port is connected to ethernet of your laptop (your laptop is upstream for the WiFi router), and your laptop is connected to WiFi (that's your laptop's upstream).

Interface variables are then:
```
# upstream=wlan0
# ap=eth0
```

Then follow these two sections from above:

#### Configure IP and dhcpd

#### NAT (Network Address Translation)


### Other interface options

This could also be done in other ways:
- Two WiFi cards - follow instructions above, but use second wlan interface for upstream instead of ethernet.
- One WiFi card - find a guide for manual setup online. I find this a bit unreliable. Good luck!


## wireshark

Use `wireshark`, inspect packets.


## mitmproxy

Start mitmproxy or mitmweb as transparent proxy:
```
$ mitmproxy -T --host  # older version
$ mitmproxy --mode transparent --showhost  # newer
```

### Redirect packets to mitmproxy

Redirect HTTP (TCP port 80) to mitmproxy:
```
# iptables -t nat -A PREROUTING -i $ap -p tcp --dport 80 -j REDIRECT --to-port 8080
```

If you have a firewall, you may need to allow connections to :8080.

### HTTPS

On your phone, go to <http://mitm.it/> and download the CA certificate.

Redirect HTTPS (TCP port 443) to mitmproxy:
```
# iptables -t nat -A PREROUTING -i $ap -p tcp --dport 443 -j REDIRECT --to-port 8080
```

### mitmproxy tricks

#### Scripts (oS)
- `/usr/share/doc/mitmproxy/examples/upsidedownternet.py`  Very useful for apps (quickly *visible* where content is downloaded with plain HTTP)!
- `/usr/share/doc/mitmproxy/examples/sslstrip.py`

#### HTML replacements (oR):

| Filter       | Regex     | Replacement |
| ------------ | --------- | ----------- |
| `~b </head>` | `</head>` | `<style>body {transform: scaleY(-1);}</style></head>` |


## Presentation slides:
<https://docs.google.com/presentation/d/1gxFvaWnkpqjU_eYIqxC9gFBeGavvNM4QLMiB1VmcnKI/edit?usp=sharing>

## Resources:
- <https://wiki.archlinux.org/index.php/software_access_point>
- <https://seravo.fi/2014/create-wireless-access-point-hostapd>
