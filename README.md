# netconnectd plugin for OctoPrint

![netconnectd plugin: Overview with list of available wifis](https://i.imgur.com/Yjmxypvl.png)

![netconnectd plugin: Configuration of secured wifi](https://i.imgur.com/NIjPBYpl.png)

This is a plugin for OctoPrint that acts as a client for the [netconnect](https://github.com/foosel/netconnectd) linux
daemon. It allows visualizing the status of the current network connection, viewing the available Wifi networks
and connecting to a Wifi network (including entering necessary network credentials).

It depends on OctoPrint version 1.2.0-dev-195 and up.

The plugin will try to automatically reconnect when switching from netconnectd's ap mode to a known network. In order
to do that, it assumes that the client host will switch to the same network as the OctoPrint instance once the access
point goes down and that the OctoPrint instance will be available as `<hostname>.local` after the switch (so the host
running OctoPrint will need to be configured with [support for .local domains](https://en.wikipedia.org/wiki/.local)). 
If the client/browser doing the configuration is running on Windows, [Bonjour for Windows](http://support.apple.com/kb/DL999) will need to be installed 
for this to seamlessly work (it comes bundled with the iTunes installed which you simply unzip to get at it but 
alternatively take a look at [this entry in the OctoPrint wiki](https://github.com/foosel/OctoPrint/wiki/Setup-on-a-Raspberry-Pi-running-Raspbian#reach-your-printer-by-typing-its-name-in-address-bar-of-your-browser---avahizeroconfbonjour-based)). 
For Linux clients, Avahi will need to be installed, including the libdnssd compatibility layer (`libavahi-compat-libdnssd1` 
in Ubuntu).

## Setup

First setup [netconnect](https://github.com/foosel/netconnectd) like described in its [README](https://github.com/foosel/netconnectd/blob/master/README.md). 
Without that daemon setup on the system serving as your OctoPrint host, the plugin won't work.

After that, just install the plugin like you would install any regular Python package from source:

    pip install https://github.com/OctoPrint/OctoPrint-Netconnectd/archive/master.zip

Make sure you use the same Python environment that you installed OctoPrint under, otherwise the plugin won't be able
to satisfy its dependencies.

Restart OctoPrint. `octoprint.log` should show you that the plugin was successfully found and loaded:

    2014-09-11 17:45:26,572 - octoprint.plugin.core - INFO - Loading plugins from ... and installed plugin packages...
    2014-09-11 17:45:26,648 - octoprint.plugin.core - INFO - Found 2 plugin(s): netconnectd client (0.1), Discovery (0.1)

## Configuration

The plugin has two configuration options that due to their sensitivity are not configurable via the web interface right
now, you'll have to edit OctoPrint's `config.yaml` directly: 

    plugins:
      netconnectd:
        # The location of the unix domain socket provided by netconnectd. Defaults to /var/run/netconnectd.sock, which
        # is also netconnectd's default value.
        socket: /var/run/netconnectd.sock
        
        # The hostname to try to reach after switching from AP mode to wifi connection. If left unset, OctoPrint will
        # automatically attempt to connect to <hostname>.local. Defaults to not set.
        hostname: someothername.local


# Notes on installing Netconnectd to Robopi

I used the netconnectd application to turn the robopi into a WAP, because it can be used to power an octoprint plugin.
https://github.com/foosel/netconnectd

This document is made up of 2 sections: installation steps and known installation issues.

# Installation steps
---

---


### Prepare the system

Install the hostapd, dnsmasq, logrotate and rfkill packages:

    sudo apt-get install hostapd dnsmasq logrotate rfkill

We don't want neither `hostapd` nor `dnsmasq` to automatically startup, so make sure their automatic start on boot is 
disabled:

    sudo update-rc.d -f hostapd remove
    sudo update-rc.d -f dnsmasq remove

You can verify that this worked by checking that there are no files left in `/etc/rc*.d` referencing those two services,
so the following to commands should return `0`:

    ls /etc/rc*.d | grep hostapd | wc -l
    ls /etc/rc*.d | grep dnsmasq | wc -l

If you are running NetworkManager (default for Ubuntu or other desktop linux distributions, usually not the case for 
Raspbian), make sure to disable its own `dnsmasq` by editing `/etc/NetworkManager/NetworkManager.conf` and commenting
out the line that says `dns=dnsmasq`, it should look something like this afterwards (note the `#` in front of the
`dns` line):

    [main]
    plugins=ifupdown,keyfile,ofono
    #dns=dnsmasq
    
    no-auto-default=00:22:68:1F:83:AF,
    
    [ifupdown]
    managed=false

You'll also need to modify `/etc/dhcp/dhclient.conf` to include a timeout setting, e.g.

    timeout 60;

Otherwise -- due to a limitation of how Debian/Ubuntu currently parses Wifi configurations in `/etc/network/interfaces` 
-- netconnectd won't be able to detect when it couldn't connect to your configured local wifi and will never start the 
access point mode. The value above will mean that it will take a maximum of 60sec before netconnectd will be notified 
by the system that the connection was unsuccessful -- you might want to lower that value even more but keep in mind that 
your wifi's DHCP server has to respond within that timeout for the connection to be considered successful.

### Check that your wifi card supports AP mode

Before you continue **make absolutely sure** that hostapd works with your wifi card/dongle! To test, create a file 
`/tmp/hostapd.conf` with the following contents:

    interface=wlan0
    driver=nl80211
    ssid=TestAP
    channel=3
    wpa=3
    wpa_passphrase=MySuperSecretPassphrase
    wpa_key_mgmt=WPA-PSK
    wpa_pairwise=TKIP CCMP
    rsn_pairwise=CCMP

Then run 

    sudo hostapd -dd /tmp/hostapd.conf

This should not show any errors but start up a new access point named "TestAP" and with passphrase 
"MySuperSecretPassphrase", verify that with a different wifi enabled device (e.g. mobile phone).

If you run into errors in this step, solve them first, e.g. by googling your wifi dongle plus "hostapd". You might need 
a custom version of hostapd (e.g. for the [Edimax EW-7811Un or other RTL8188 based cards](http://jenssegers.be/blog/43/Realtek-RTL8188-based-access-point-on-Raspberry-Pi)) 
or a custom driver. If you change anything related to `hostapd` during getting this to work, verify again afterwards
that the automatic startup of `hostapd` is still disabled and if not, disable it again (see above for infos on how
to do that).

### Install netconnectd

It's finally time to install `netconnectd`:

    cd
    git clone https://github.com/foosel/netconnectd
    cd netconnectd
    sudo python setup.py install
    sudo python setup.py install_extras

Modify `/etc/netconnectd.yaml` as necessary:
 
  * Change the passphrase/psk for your access point
  * If necessary change the interface names of your wifi and wired network interfaces
  * If your machine is **not** running NetworkManager, set `wifi > free` to `false`
  * if you **don't** want to reset the wifi interface in case of any detected errors on the driver level, set
    `wifi > kill` to `false`
 
Last, start netconnectd:

    sudo service netconnectd start

Verify that the logfile looks ok-ish:

    less /var/log/netconnectd.log

and that it's indeed running (error handling of the start up script still needs to be improved):

    netconnectcli status

Congratulations, `netconnectd` is now running and should detect when you don't have any connection available, starting the AP mode to change that.

You can control the daemon via `netconnectcli`:

  * `netconnectcli status` displays the current status (which interfaces are connected, is the AP running, etc)
  * `netconnectcli start_ap` manually starts the AP
  * `netconnectcli stop_ap` manually stops the AP
  * `netconnectcli list_wifi` shows the wifi cells currently in range
  * `netconnectcli configure_wifi <ssid> <psk>` configures the wifi connection (`<ssid>` = the wifi's SSID, `<psk>` = the wifi's passphrase)
  * `netconnectcli select_wifi` manually brings up the wifi configuration

You can always get help with `netconnectcli --help` or `netconnectcli <command> --help` for specific commands.

If everything looks alright, configure the service so that it starts at boot up:

    sudo update-rc.d netconnectd defaults 98

# Known Issues
---

---

### Summary: 

Dnsmasq still automatically starts at boot.This conflicts with netconnectd's ability to start up an access point because the IP address that netconnectd would have used to bind its run of dnsmasq is already taken up by a previously booted dnsmasq run. **(Unresolved)**

### Leads for solving:

Verify whether dnsmasq is starting at boot or there's another program thats calling dnsmasq.

http://askubuntu.com/questions/218/command-to-list-services-that-start-on-startup https://help.ubuntu.com/community/UbuntuBootupHowto
http://manpages.ubuntu.com/manpages/precise/man8/update-rc.d.8.html
http://ubuntuforums.org/showthread.php?t=1417870
