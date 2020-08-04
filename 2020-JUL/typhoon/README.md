Some folks have asked for the assets I used when first attempting typhoon.
There are some pointers on the typhoon website, at
(https://typhoon.psdn.io/flatcar-linux/bare-metal/), and interestingly the
author is probably using virtualization ( the MAC addresses in the initial
walktrhough are typical MAC addresses you would see provisioned by qemu) .
However, I am not aware of a system that lets you boot similar instances with
IPMI. Here are the detailed configs I used for my trial run with
virtualization:

I performed this on on debian bullseye, currently in testing. Which is awesome,
because I found a bug in debian!
(https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=966517)

I also ran everything as root, because this was a purpose-built install.


Regarding networking, we'll be creating a network bridge called matchbox . This
is kind of like an internal network switch in linux, which we can tell qemu to
plug the virtual nic's into. It should also be usable with our host machine to
serve dhcp, dns, and the matchbox http server.

```
brctl addbr matchbox
ip link set matchbox up
ip addr add 10.0.2.1/24 dev matchbox
```

We also need to enable external access to VMs connected to this bridge, I used the following:
```
iptables -t nat -A POSTROUTING -s 10.0.2.0/24 -j MASQUERADE
echo 1 >/proc/sys/net/ipv4/ip_forward
iptables -P FORWARD ACCEPT
```

That is a pretty permissive forwarding policy, and may be overwritten by the Docker daemon on some
installations, just FYI.


The first setup daemon was dns. I used dnsmasq, installed from the debian repos, with a couple of custom nodes defined:
/etc/dnsmasq.conf:

```
resolv-file=/etc/upstream.conf
address=/node1.local/10.0.2.2
address=/matchbox.local/10.0.2.1
address=/worker1.local/10.0.2.3
listen-address=127.0.0.1,10.0.1.1,10.0.2.1
```
I had to tell dnsmasq what to query upstream, good old 8.8.8.8

/etc/upstream.conf:
```
nameserver 8.8.8.8
```

DNSMasq comes with a systemd unit, and can be restarted with `systemctl restart dnsmasq`

Next, I had to set up dhcp. This is what the qemu instances will use to connect
to matchbox.  Fortunately, qemu comes with ipxe baked in. This means that we
don't additionally have to serve a pxe to ipxe binary over tftp. I'll be using
the busybox udhcpd server. I install busybox and use the following udhcpd.conf:

```
start   10.0.2.30
end   10.0.2.90
interface matchbox
siaddr   http://matchbox.local
option  subnet  255.255.255.0
opt router 10.0.2.1
opt dns 10.0.2.1 8.8.8.8
boot_file http://matchbox.local/boot.ipxe
static_lease 52:54:00:12:34:56 10.0.2.2
static_lease 82:94:3D:44:C1:3A 10.0.2.3
```

I run a lot of the commands in the foreground in gnu screen sessions to see the different
parts working. `busybox udhcpd -fvv` will do this with decent logging.

Next, we need to install the matchbox server. This is pretty well documented.
Rather than install any systemd units, I just downloaded the binary and put it
in /usr/local/bin . I will detail in a moment how I ran it. There are a few
notes on how to set up tls, with matchbox. Since matchbox is configured via
grpc, it typically requires TLS, and therefore to setup tls certs. The beauty
of this is that terraform can provision the certs for us!

We also need a recent version of terraform, the matchbox terraform provider,
and the ct terraform provider.

First, I provision the certs, and copy them to the appropriate places:
```
provider "tls" {
}

provider "local" {
}

resource "tls_private_key" "acme_ca" {
  algorithm = "RSA"
  rsa_bits  = "4096"
}

resource "local_file" "acme_ca_key" {
  content  = "${tls_private_key.acme_ca.private_key_pem}"
  filename = "${path.module}/certs/acme_ca_private_key.pem"
}

resource "tls_self_signed_cert" "acme_ca" {
  key_algorithm     = "RSA"
  private_key_pem   = "${tls_private_key.acme_ca.private_key_pem}"
  is_ca_certificate = true
  subject {
    common_name         = "Acme Self Signed CA"
    organization        = "Acme Self Signed"
    organizational_unit = "acme"
  }

  validity_period_hours = 87659

  allowed_uses = [
    "digital_signature",
    "cert_signing",
    "crl_signing",
  ]
}

resource "local_file" "acme_ca_cert" {
  content  = "${tls_self_signed_cert.acme_ca.cert_pem}"
  filename = "${path.module}/certs/acme_ca.pem"
}

resource "tls_private_key" "matchbox_local" {
  algorithm = "RSA"
  rsa_bits  = "4096"
}

resource "local_file" "matchbox_local_key" {
  content  = "${tls_private_key.matchbox_local.private_key_pem}"
  filename = "${path.module}/certs/matchbox_local_private_key.pem"
}

resource "tls_cert_request" "matchbox_local" {
  key_algorithm   = "RSA"
  private_key_pem = "${tls_private_key.matchbox_local.private_key_pem}"

  dns_names = ["matchbox.local"]

  subject {
    common_name         = "matchbox.local"
    organization        = "Example Self Signed"
    country             = "US"
    organizational_unit = "matchbox.local"
  }
}

resource "tls_locally_signed_cert" "matchbox_local" {
  cert_request_pem   = "${tls_cert_request.matchbox_local.cert_request_pem}"
  ca_key_algorithm   = "RSA"
  ca_private_key_pem = "${tls_private_key.acme_ca.private_key_pem}"
  ca_cert_pem        = "${tls_self_signed_cert.acme_ca.cert_pem}"

  validity_period_hours = 87659

  allowed_uses = [
    "digital_signature",
    "key_encipherment",
    "server_auth",
    "client_auth",
  ]
}

resource "local_file" "matchbox_local_cert_pem" {
  content  = "${tls_locally_signed_cert.matchbox_local.cert_pem}"
  filename = "${path.module}/certs/matchbox_local_cert.pem"
}            
```

This will create a CA cert/key pair and a matchbox cert/key pair. I added
client_auth to the allowed uses, which means that I can use the same matchbox
cert/key pair to serve with matchbox, and in the terraform client to matchbox.
This is pure laziness, and I recommend you try it this way and then fix it
before prod. I copied the certs into place after terraform applying them:
```
cp -iv certs/matchbox_local_cert.pem /etc/matchbox/server.crt
cp -iv certs/matchbox_local_private_key.pem /etc/matchbox/server.key
cp -iv certs/acme_ca.pem /etc/matchbox/ca.crt
```

Then I could start matchbox. I did so in another screen session ala:
```MATCHBOX_RPC_ADDRESS=0.0.0.0:8081  MATCHBOX_ADDRESS=0.0.0.0:80 matchbox```

Note I am serving matchbox on port 80, which is what jives with the udhcpd config above.
You will have to run as root to run matchbox in this way.

I downloaded the flatcar linux assets to the matchbox asset directory:

```
 ls -laR /var/lib/matchbox/assets/
/var/lib/matchbox/assets/:
total 0
drwxr-xr-x 1 root root 14 Jul 25 20:09 .
drwxr-xr-x 1 root root 56 Jul 25 19:36 ..
drwxr-xr-x 1 root root 16 Jul 25 20:09 flatcar

/var/lib/matchbox/assets/flatcar:
total 0
drwxr-xr-x 1 root root  16 Jul 25 20:09 .
drwxr-xr-x 1 root root  14 Jul 25 20:09 ..
drwxr-xr-x 1 root root 268 Jul 26 00:31 2345.3.1

/var/lib/matchbox/assets/flatcar/2345.3.1:
total 952128
drwxr-xr-x 1 root root       268 Jul 26 00:31 .
drwxr-xr-x 1 root root        16 Jul 25 20:09 ..
-rw-r--r-- 1 root root 490738516 Mar 26 16:50 flatcar_production_image.bin.bz2
-rw-r--r-- 1 root root       594 Mar 26 16:50 flatcar_production_image.bin.bz2.sig
-rw-r--r-- 1 root root 430218673 Mar 26 17:52 flatcar_production_pxe_image.cpio.gz
-rw-r--r-- 1 root root  54010528 Mar 26 17:52 flatcar_production_pxe.vmlinuz
```

With DNS, DHCP, and Matchbox started, and NAT enabled from our bridge, we should be ready to go!

We need an ssh key. I did something like `ssh-keygen -f ~/.ssh/typhoon` . The public key (in my case
~/.ssh/typhoon.pub) will need to be embedded into the terraform below.

Here is our terraform for typhoon:
```
provider "matchbox" {
  endpoint    = "matchbox.local:8081"
  client_cert = file("/etc/matchbox/server.crt")
  client_key  = file("/etc/matchbox/server.key")
  ca          = file("/etc/matchbox/ca.crt")
}

provider "ct" {
}

module "mercury" {
  source = "git::https://github.com/poseidon/typhoon//bare-metal/container-linux/kubernetes?ref=v1.18.6"

  # bare-metal
  cluster_name            = "mercury"
  matchbox_http_endpoint  = "http://matchbox.local"
  os_channel              = "flatcar-stable"
  os_version              = "2345.3.1"

  # configuration
  k8s_domain_name    = "node1.local"
  ssh_authorized_key = "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCX8q6sWgx48VhgRRxO0UTEuM2AxNDr7opXyufPbWRB6L9btVMGLVeRRYU1uJnoZhoAjjBuWWrSk347QDuSge/P0+GwiK3qb4pRGTfHUfAvHEyU13brmymilv0uzQHm5KmmCAbl6TBh7DdcsUf0VOnJZK/qE7IqdeajAZfqbSv6gFS+XzbZyy97U7hZonK7K82ERWKQ6kv12abu2ZDjSNy8MFivOxN/53q14nomTaSz1i47jnuxCNEL85Vm6lZRVtgYmzCQrCEIhHTJ5FqKYNiNQYv/x+iMOdla/oo+Ccyf/Su5gY4VfhH95d8czeQhjaft/4bAuSd6+M4pI+xRt7F7xvTDtEeyMF6npbhhbATAZHzVcWsnhjM78/HuXCQjfrZAApDU1YtKJHCjl2wjT9jE2klKseZDvZu0RpTcwOfTcbQeS3XaHxzSo5FElqwjhGHvhaCTjvWnrmbxJRWz0Bqn+2F3u5RsbqzmfhIL6PCwPHOfnTTkESksgr9z6pBm4pk="

  kernel_args = ["coreos.autologin=ttyS0"]
  cached_install = "true"
  # machines
  controllers = [{
    name   = "node1"
    mac    = "52:54:00:12:34:56"
    domain = "node1.local"
  }]
  workers = [{
    name   = "worker1"
    mac    = "82:94:3D:44:C1:3A"
    domain = "worker1.local"
  }]
  # set to http only if you cannot chainload to iPXE firmware with https support
  # download_protocol = "http"
}

resource "local_file" "kubeconfig-mercury" {
  content  = module.mercury.kubeconfig-admin
  filename = "/root/workspace/mercury-config"
}
```

I add an autologin on ttyS0 for coreos. This will let us use the serial console qemu gives us to
explore/observe while the machines are booting.

I ensure qemu knows it is allowed to connect to the matchbox bridge:
```mkdir -p /etc/qemu && echo allow matchbox >/etc/qemu/bridge.conf```

I create the disks that the qemu instances will use (these end up at about 10G each, but grow gradually due to the qcow file format).
``` qemu-img create -f qcow2 flatcar-node.img 10G ```
``` qemu-img create -f qcow2 flatcar-worker.img 10G

We will need to install the terraform ct and matchbox providers, the instructions for that are on the typhoon website ( https://typhoon.psdn.io/flatcar-linux/bare-metal/#terraform-setup )
We also need to run with the private key for the above ssh pubkey (embedded in they typhoon terraform) in an ssh agent. run `. <(ssh-agent)` and the ssh-add with a key you generated for this practice.
For me it was `ssh-add ~/.ssh/typhoon` .

I run `terraform init && terraform apply -auto-approve`

This will provision out matchbox, and wait for us to 
start them, until the typhoon provider can ssh into them to distribute the kubernetes secrets.

First I start the node (the controller in this case):
`qemu-system-x86_64 -nic bridge,br=matchbox -nographic -machine accel=kvm -m 4G -smp 2 -hda flatcar-node.img`

This should show you the virtual machine progressively booting.

Then I start the worker node:

`qemu-system-x86_64 -nic bridge,br=matchbox,mac=82:94:3D:44:C1:3A -nographic -machine accel=kvm -m 4G -smp 2 -hda flatcar-worker.img`

After about 20 minutes worth of provisioning, depending on your environment, and how fast the registries for the various components serve them to you, you should have
a two-node kubernetes cluster ready for testing! The kubeconfig should be created automatically by the last stanza in the above terraform.

Happy trails, and let me know if you run into any issues!
Eldon
