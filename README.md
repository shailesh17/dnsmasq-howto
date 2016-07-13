# dnsmasq-howto - Setting up dnsmasq for Local Web Development Testing
This HOWTO walks you through installing dnsmasq on MacOS and using it to redirect development sites and other machines on your network to this DNS server running on your local machine.

Most web developers update `/etc/hosts` file to direct traffic for myapp.local to 127.0.0.1. This works well, but is only used for apps running on that machine. If you need other devices on your network to hit a running service on your machine, you cannot easily have them look up your IP everytime. 

Installing a local DNS server like dnsmasq and configuring your system and the other devices to use that server can centralize the configuration needed for development.

We'll cover:
  1. Installing dnsmasq on MacOS
  2. Configuring dnsmasq to respond to all .local requests with 127.0.0.1.
  3. Configuring MacOS to send all .local requests requests to dnsmasq.
  4. Configuring a static Virtual IP for your machine on your network
  5. Configuring your MacOS system to use your new DNS server
  6. Configuring your router to send DNS requests to your machine

WARNING: these instructions show you how to install new system software and change your system configuration. Like all such changes, you should not proceed unless you are confident you have understood them and that you can reverse the changes if needed.

## MacOS Instructions

### Install MacOS Command Line Tools
 * Download the most recent version of *Command Line Tools for Xcode" from the [Apple Developer site](https://developer.apple.com/downloads/index.action) and install it.

### Install Homebrew
 * Install the [Homebrew](http://brew.sh/) package manager on your system. This is needed to install the latest version of dnsmasq.

### Install dnsmasq
```
# Update your homebrew installation
brew up
# Install dnsmasq
brew install dnsmasq
```
### Configure dnsmasq
First time setup involves copying the default configuration file example and setting up dnsmasq to start automatically as a LaunchDaemon (launchd):

```
# Copy the default configuration file.
cp $(brew list dnsmasq | grep /dnsmasq.conf.example$) /usr/local/etc/dnsmasq.conf
# Copy the daemon configuration file into place.
sudo cp $(brew list dnsmasq | grep /homebrew.mxcl.dnsmasq.plist$) /Library/LaunchDaemons/
# Start dnsmasq automatically.
To have launchd start dnsmasq now and restart at startup:
  sudo brew services start dnsmasq
  -sudo launchctl load /Library/LaunchDaemons/homebrew.mxcl.dnsmasq.plist-
```

You should now have dnsmasq running. Let's configure it further! Open the configuration file `/usr/local/etc/dnsmasq.conf`using your favorite editor.

### Configuring dnsmasq to respond to all .local requests with 127.0.0.1
One the basic things that dnsmasq can do is compare DNS requests against a database of patterns and use these to resolve the correct response. I use this functionality to match any request which ends in .local and send 127.0.0.1 in response. 

The dnsmasq configuration directive should be added to the configuration file, its best to put it near the other `address=...` entries just to keep it all together.
```
address=/local/127.0.0.1
```

Changing the configuration file requires a restart of the dnsmasq instance for it to reload the configuration.  This can be done as follows:
```
sudo brew services stop dnsmasq
sudo brew services start dnsmasq
```

You can test dnsmasq by sending it a DNS query using the `dig` utility. Pick a name ending in `.local` and use dig to query your new DNS server:
```
dig myapp.local @127.0.0.1
```

You should get back a response something like:
```
;; ANSWER SECTION:
myapp.local. 0 IN	A	127.0.0.1
```

### Configuring a static Virtual IP for your machine on your network
In this section, we'll setup a new virtual interface on your machine so other devices or your LAN router on your network can then be configured to use this as first DNS responder

  1. Go to *System Preferences > Network*
  2. Click the plus *+* sign on the bottom left
  3. Select Interface: *Wi-FI*
  4. Set Service Name to something like *My Virtual Wi-Fi*
  5. Select this new Interface and click *Advanced...* on the right pane.
  6. Go to the *TCP/IP* tab and set *Configure IPv4:* to *Using DHCP with manual address*
  5. Enter your desired IP address in the *IPv4 Address:* field. For example use: `10.10.10.101`
  6. Unless you need IPv6, set it to *Off*
  7. Click *Ok* then *Apply*
  8. Now you should see the interface connected, with message: `My Virtual Wi-FI is connected to <SSID> and has the IP address 10.10.10.101.`

### Test
1. `dscacheutil -flushcache`
2. `dig google.com @10.10.10.101`

You should see something like this near the bottom:  
`;; SERVER: 10.10.10.101#53(10.10.10.101)`

### Configuring your MacOS system to use your new DNS server
Now that you have a working DNS server you can configure your operating system to use it. There are two approaches to this:

   1. Send all DNS queries to dnsmasq.
   2. Send only .local queries to dnsmasq

The first approach is easy – just change your DNS settings in System Preferences.  This might require additional changes to the dnsmasq configuration, so be sure to review rest of the configuration file.
### Add your IP to your DNS settings
   1. Go to *System Preferences > Network*
   2. Select the *Wi-Fi* interface
   2. Click *Advanced...* on the right pane
   3. Click the *DNS* tab
   4. Add 10.10.10.101 (your IP address) to the list and drag it to the top of the list if you can
   5. Click *Ok* then *Apply*
   6. Now repeat the same for your *My Virtual Wi-Fi* interface

You will need to do this on any device you would like to access your .local hostnames, including VMWare instances and iOS devices.

### Restart the mDNSResponder on MacOS
```
sudo launchctl unload -w /System/Library/LaunchDaemons/com.apple.mDNSResponder.plist
# To turn it back on, just do the opposite:
sudo launchctl load -w /System/Library/LaunchDaemons/com.apple.mDNSResponder.plist
# Alternatively, you can just send HUP:
sudo killall -INFO mDNSResponder
```

Now test the default DNS query:
1. `ping www.somewhere.local`  
You should see something like this:  
```
PING www.somewhere.local (127.0.0.1): 56 data bytes
64 bytes from 127.0.0.1: icmp_seq=0 ttl=64 time=0.035 ms
64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.046 ms
```

The second is a bit more tricky, but not much. Most UNIX-like operating systems have a configuration file called /etc/resolv.conf which controls the way DNS queries are performed, including the default server to use for DNS queries (this is the setting that gets set automatically when you connect to a network or change your DNS server/s in System Preferences).

OS X also allows you to configure additional resolvers by creating configuration files in the /etc/resolver/ directory. This directory probably won’t exist on your system, so your first step should be to create it:

```
sudo mkdir -p /etc/resolver
```
Now you should create a new file in this directory for each resolver you want to configure. Each resolver corresponds – roughly and for our purposes – to a top-level domain like our `.local`. There a number of details you can configure for each resolver generally you need two:

   1. the name of the resolver (which corresponds to the domain name to be resolved); 
   2. and the DNS server to be used.
For more information about these files, see the resolver(5) manual page: `man 5 resolver`

Create a new file with the same name as your new top-level domain in the /etc/resolver/ directory and add a nameserver to it by running the following commands:
```
sudo tee /etc/resolver/local >/dev/null <<EOF
nameserver 127.0.0.1
EOF
```
Here `local` is the top-level domain name that I’ve configured dnsmasq to respond to and 127.0.0.1 is the IP address of the server to use.

Once you’ve created this file, OS X will automatically read it and you’re done!

Testing
Testing you new configuration is easy; just use ping check that you can now resolve some DNS names in your new top-level domain:

# Make sure you haven't broken your DNS.
ping -c 1 www.google.com
# Check that .local names work
ping -c 1 www.test.local
You should see results that mention the IP address in your Dnsmasq configuration like this:

PING www.test.local (127.0.0.1): 56 data bytes
You can now just make up new DNS names under .local whenever you please. Congratulations!

### Configuring your router to send DNS requests to your machine
Here you will access your router web control panel and replace the DNS servers to point to your local virtual IP address configured earlier under *My Virtual Wi-Fi* interface.
  1. Login to your router's web control panel, for example: http://10.10.10.254/
  2. Find settings for DHCP Server, usually something like: Settings->Network->DHCP Server
  3. Change the *Primary DNS* server to your manually configured Virtual IP Address: `10.10.10.101`
  4. Apply/OK/restart the router

Now restart the router and disconnect/reconnect all the devices using this router. This will feed the default DNS configuration to the re-connected devices.

