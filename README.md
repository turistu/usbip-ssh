# usbip-ssh
This script is using the kernel's USB/IP modules and the ssh's connection
forwarding mechanism to share usb devices between different linux machines.

It does not depend on the USB/IP's project userland components and does not
require any extra software on either the local or the remote machine besides
an openssh client/server and a base perl installation.

The script itself only has to be installed on the local machine -- the only
configuration you have to do on the remote machine is setting up ssh auth
correctly (by copying your public key to `~/.ssh/authorized_keys`, etc)
so you can log in as root.

Assuming that you did that, you can try:
```
# /path/to/usbip-ssh root@raspbery-pi list
1-1  2109:3431  USB2.0 Hub
  1-1.3  00da:8510  Telink  Wireless Receiver
      :1.0 030102  mouse  [usbhid] event5 event3 mouse0 event4 hidraw0
      :1.1 030101  kbd    [usbhid] event6 hidraw1
# /path/to/usbip-ssh verbose=1 root@raspberry-pi Telink
...
```
After which you can use the keyboard / mouse connected to the remote
`raspberry-pi` machine as if they were connected to the local machine.

If that works, you set it to start automatically from an `/etc/boot.d`
script with:
```
exec /path/to/usbip-ssh verbose=1 daemon root@raspberry-pi Telink
```
The `daemon` option will cause it to use `syslog(3)` instead of stderr and
to keep trying (if e.g. the device/remote machine is not accessible, the device
was plugged out, the connection was broken, etc), but spacing out the retries
depending on how much the script has successfully run.

#### How does USB/IP work

Both the `usbip_host` driver (on the exporting/remote device) and `vhci_hdc`
(on the local/importing device) work by tunneling the USB protocol over a socket
file descriptor passed from a userland process; despite the "IP" in "USB/IP",
the socket can be *any* kind of *stream* socket, including a unix domain socket
created with `socketpair(2)`.

#### Why this script sucks so much

Despite it being theoretically easy to use any kind of medium for USB/IP (not
just open tcp connections over a "secure" network), the ineptness of the ssh
protocol (and the programs implementing it) makes everything much harder than
it has to be: ssh is neither able to forward simple file descriptors, nor use
a unix domain socket for the stdin/out of the remote program (the only options
being a pair of pipes or a pseudo-terminal).

So this script had to use ssh's "connection forwarding" mechanism, which turned
everything into a racy bloody mess, with temporary directories (which have to
removed after use), master and slave ssh commands, and extra dummy processes
which listen and connect to sockets.

