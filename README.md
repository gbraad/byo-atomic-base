Build Your Own Atomic Image, Updated
================

When [Project Atomic](http://www.projectatomic.io) got off the ground in April, I wrote a [blog post](http://www.projectatomic.io/blog/2014/04/build-your-own-atomic-host-on-fedora-20/) about how anyone could Build Your Own Atomic host, based on Fedora 20. Since that time, there have been some changes in the rpm-ostree tooling used to produce these images. 

What's more, there's a new distro around the corner, [CentOS 7](http://seven.centos.org), that you may wish to build into an Atomic host. Part of what's great about the Atomic model is the way it can apply to different distributions. Here's our chance to play with that.

Here's an updated guide to building your own Atomic host(s), based on Fedora 20 or on (the currently pre-release) CentOS 7.

## First, build and configure the builder:

Install Fedora 20 (CentOS 7 should work, too, but I've found Fedora to be more nested-virt friendly. The rpm-ostree tools use kvm during the image-building process, and if your builder is a VM, like mine is, you'll be tangling w/ nested virt.)

Disable selinux by changing `enforced` to `disabled` in `/etc/selinux/config` and then `systemctl reboot` to complete selinux disabling. While we're never happy about disabling SELinux, it's necessary ([for now](https://bugzilla.redhat.com/show_bug.cgi?id=1060423)) to disable it on your builder in order to enable it on the Atomic instances you build.

The rpm-ostree commands need to be run as root or w/ sudo, but for some reason, the image-building part of the process is only working for me while running as root (not sudo), so I log in as root and work in `/root`.

````
# yum install -y git
# git clone https://github.com/jasonbrooks/byo-atomic.git
# mv byo-atomic/walters-rpm-ostree-fedora-20-i386.repo /etc/yum.repos.d/
# yum install -y rpm-ostree rpm-ostree-toolbox nss-altfiles yum-plugin-protectbase httpd
````

Now, edit `/etc/nsswitch.conf` change lines `passwd: files` and `group: files` to `passwd: files altfiles` and `group: files altfiles` [(details)](https://github.com/projectatomic/rpm-ostree).

Then, edit `/etc/libvirt/qemu.conf` to uncomment the line `user = "root"` and `systemctl restart libvirtd`.

Now, we'll set up a repository from which our eventual Atomic hosts will fetch upgrades:

````
# mkdir -p /srv/rpm-ostree/repo && cd /srv/rpm-ostree/ && sudo ostree --repo=repo init --mode=archive-z2
# cat > /etc/httpd/conf.d/rpm-ostree.conf <<EOF
DocumentRoot /srv/rpm-ostree
<Directory "/srv/rpm-ostree">
Options Indexes FollowSymLinks
AllowOverride None
Require all granted
</Directory>
EOF
# systemctl daemon-reload &&
systemctl enable httpd &&
systemctl start httpd &&
systemctl reload httpd &&
firewall-cmd --add-service=http &&
firewall-cmd --add-service=http --permanent
````

## Next, build the Atomic host:

The *.json files in the c7 and f20 directories contain the definitions for these Atomic hosts. 

### For CentOS 7:

````
# cd /root/byo-atomic/c7
# rpm-ostree compose tree --repo=/srv/rpm-ostree/repo centos-atomic-server-docker-host.json
# rpm-ostree-toolbox create-vm-disk /srv/rpm-ostree/repo centos-atomic-host centos-atomic/7/x86_64/server/docker-host c7-atomic.qcow2
````

### For Fedora 20:

````
# cd /root/byo-atomic/f20
# rpm-ostree compose tree --repo=/srv/rpm-ostree/repo fedora-atomic-server-docker-host.json
# rpm-ostree-toolbox create-vm-disk /srv/rpm-ostree/repo fedora-atomic-host fedora-atomic/20/x86_64/server/docker-host f20-atomic.qcow2
````

After you've created your image(s), future runs of the `rpm-ostree compose tree` command will add updated packages to your repo, which you can pull down to an Atomic instance. For more information on updating, see "Configuring your Atomic instance to receive updates," below.

These scripts produce qcow2 images, which are ready to use with OpenStack or with virt-manager/virsh. To produce *.vdi images, use qemu-img to convert:

`qemu-img convert -f qcow2 c7-atomic.qcow2 -O vdi c7-atomic.vdi`

The atomic images are born with no root password, so it's necessary to supply a password or key to log in using cloud-init. If you're using a virtualization application without cloud-init support, such as virt-manager or VirtualBox, you can create a simple iso image to provide a key or password to your image when it boots.

To create this iso image, you must first create two text files.

Create a file named "meta-data" that includes an "instance-id" name and a "local-hostname." For instance: 

````
instance-id: Atomic0
local-hostname: atomic-00
````

The second file is named "user-data," and includes password and key information. For instance:

````
#cloud-config
password: atomic
chpasswd: {expire: False}
ssh_pwauth: True
ssh_authorized_keys:
  - ssh-rsa AAA...SDvz user1@yourdomain.com
  - ssh-rsa AAB...QTuo user2@yourdomain.com
````

Once you have completed your files, they need to packaged into an ISO image. For instance:

````
# genisoimage -output atomic0-cidata.iso -volid cidata -joliet -rock user-data meta-data
````
You can boot from this iso image, and the auth details it contains will be passed along to your Atomic instance.

For more information about creating these cloud-init iso images, see http://cloudinit.readthedocs.org/en/latest/topics/datasources.html#config-drive.

### What's my IP address?

If, once you've launced your Atomic image, you need to find out what IP address it's been assigned, check out this blog post: http://rwmj.wordpress.com/2010/10/26/tip-find-the-ip-address-of-a-virtual-machine/ for more info.

### Configuring your Atomic instance to receive updates

As created using these instructions, your Atomic image won't be configured to receive updates. To configure your image to receive updates from your build machine, edit (as root) the file `/ostree/repo/config` and add a section like this:

````
[remote "centos-atomic-host"]
url=http://$YOUR_BUILD_MACHINE/repo
branches=centos-atomic-/7/x86_64/server;
gpg-verify=false
````
Or, for Fedora:

````
[remote "fedora-atomic-host"]
url=http://$YOUR_BUILD_MACHINE/repo
branches=fedora-atomic-/20/x86_64/server;
gpg-verify=false
````

With your repo configured, you can check for updates with the command `sudo rpm-ostree upgrade`, followed by a reboot. Don't like the changes? You can rollback with `rpm-ostree rollback`, followed by another reboot.

## Till Next Time

If you run into trouble following this walkthrough, I’ll be happy to help you get up and running or get pointed in the right direction. Ping me at jbrooks in #atomic on freenode irc or [@jasonbrooks](https://twitter.com/jasonbrooks) on Twitter. Also, be sure to check out the [Project Atomic Q&A site](http://ask.projectatomic.io/en/questions/).












