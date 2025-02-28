---
layout: default
title: Non-Go Prototyping
pre: "<b>5. </b>"
weight: 5
aliases:
  - /prototyping.html
---

To realize the full benefits of gokrazy, you need to use only
software written in Go. If there is no Go software for what you want
to do, creating that piece of software can pose a seemingly
unsurmountable hurdle. To make some quick progress and figure out if
your idea can be implemented, it might make sense to temporarily use
existing software before starting your own implementation.

This article shows a couple of techniques for getting non-Go
software to work on gokrazy, in increasing order of complexity.

Note that software which is manually installed like shown
here **will not be automatically updated** by gokrazy
and hence poses a security risk. Use these techniques only for
prototyping.
	  
## Go software not written for gokrazy: node-exporter
	  
The Prometheus node-exporter doesn’t use cgo and needs no
command-line parameters, configuration files or other assets can be
added to the <code>gokr-packer</code> command line.

## Go software not written for gokrazy: Grafana
	  
It would not suffice to add Grafana to your <code>gokr-packer</code>
command, as the resulting Grafana binary requires assets, supports
plugins, keeps state, etc.
	  
Hence, you need to manually install Grafana into a directory
underneath <code>/perm</code>. A convenient way to do that is to
use <a href="https://github.com/gokrazy/breakglass">breakglass</a>
to download the “Standalone Linux Binaries” release from
<a href="https://grafana.com/grafana/download?platform=arm">https://grafana.com/grafana/download?platform=arm</a>. Note
that I am serving the file from my computer because my busybox
version supports neither HTTPS nor DNS.
	  
```text
/tmp/breakglass531810560 # wget http://10.0.0.76:4080/grafana-5.3.2.linux-arm64.tar.gz
/tmp/breakglass531810560 # tar xf grafana-5.3.2.linux-arm64.tar.gz
```

We cannot start Grafana yet, as its binary is dynamically
linked. One way to fix this is to place the sources which correspond
to the release you just unpacked (e.g. from
<a href="https://github.com/grafana/grafana/tree/v5.3.2">https://github.com/grafana/grafana/tree/v5.3.2</a>)
in your <code>$GOPATH</code> and recompile the binaries:
	  

```shell
GOARCH=arm64 CGO_ENABLED=1 CC=aarch64-linux-gnu-gcc go install \
  -ldflags "-linkmode external -extldflags -static" \
  github.com/grafana/grafana/pkg/cmd/...
```

Note that it is usually easier to set the environment
variable <code>CGO_ENABLED=0</code> to get a statically linked
binary, but Grafana uses sqlite3, which is written in C, so we
resort to the <code>-ldflags</code> variant.

At this point, we can start Grafana from breakglass:
	  
```text
/tmp/breakglass531810560 # cd grafana-5.3.2
/tmp/breakglass531810560/grafana-5.3.2 # wget http://10.0.0.76:4080/grafana-server
/tmp/breakglass531810560/grafana-5.3.2 # install -m 755 grafana-server bin/ && rm grafana-server
/tmp/breakglass531810560/grafana-5.3.2 # ./bin/grafana-server
INFO[10-30|19:27:51] Starting Grafana                         logger=server version=5.0.0 commit=NA compiled=2018-10-30T19:27:51+0100
…
```
	  
To have gokrazy start Grafana, we can use a Go package like this:
	  
```go
package main

import (
  "log"
  "syscall"
)

func main() {
  const bin = "/perm/grafana/bin/grafana-server"
  if err := syscall.Exec(bin, []string{bin, "-homepath=/perm/grafana"}, nil); err != nil {
    log.Fatal(err)
  }
}
```

## C software: WireGuard

	  
WireGuard is a modern VPN tunnel, which consists of a Linux kernel
module and a configuration
tool. See <a href="https://github.com/rtr7/kernel/commit/c7afbc1fd2efdb9e1149d271c4d2be59cc5c98f4">rtr7/kernel@c7afbc1f</a>
for how the kernel module was added to the router7 kernel.

The configuration tool can be statically cross-compiled. We can run
Debian in a Docker container to not mess with our host system:
	  
```text
% mkdir /tmp/wg
% cd /tmp/wg
% docker run -t -i debian
root@d1728eaaa6e1:/# dpkg --add-architecture arm64
root@d1728eaaa6e1:/# apt update
root@d1728eaaa6e1:/# apt install libmnl-dev:arm64 libelf-dev:arm64 linux-headers-amd64 crossbuild-essential-arm64 pkg-config wget
root@d1728eaaa6e1:/# wget https://git.zx2c4.com/WireGuard/snapshot/WireGuard-0.0.20181018.tar.xz
root@d1728eaaa6e1:/# tar xf WireGuard-0.0.20181018.tar.xz
root@d1728eaaa6e1:/# cd WireGuard-0.0.20181018/src/tools
root@d1728eaaa6e1:/# make CC=aarch64-linux-gnu-gcc LDFLAGS=-static
root@d1728eaaa6e1:/# exit
% docker cp -L d1728eaaa6e1:/WireGuard-0.0.20181018/src/tools/wg .
```
	  
Now we can copy and run the <code>wg</code> binary via breakglass:
	  
```text
/tmp/breakglass531810560 # wget http://10.0.0.76:4080/wg
/tmp/breakglass531810560 # chmod +x wg
/tmp/breakglass531810560 # ./wg --help
Usage: ./wg <cmd> [<args>]
…
```

## C software: tc
	  
Linux’s Traffic Control system (used e.g. for traffic shaping) is
configured with the <code>tc</code> tool.

<code>tc</code> is a special case in that it requires to be
dynamically linked. The different queueing disciplines are
implemented as plugins, and statically linking <code>tc</code>
results in a binary which starts but won’t be able to display or
change queueing disciplines.

Because gokrazy doesn’t include a C runtime environment, we’ll need
to copy not only the <code>tc</code> binary, but also the dynamic
loader and all required shared libraries. We can run Debian in a
Docker container to not mess with our host system:
	  
```text
% mkdir /tmp/iproute
% cd /tmp/iproute
% docker run -t -i debian
root@6e530a973d45:/# dpkg --add-architecture arm64
root@6e530a973d45:/# apt update
root@6e530a973d45:/# apt install iproute2:arm64 qemu-user-static
root@6e530a973d45:/# LD_TRACE_LOADED_OBJECTS=1 qemu-aarch64-static /sbin/tc
	libelf.so.1 => /usr/lib/aarch64-linux-gnu/libelf.so.1 (0x00000040008a6000)
	libm.so.6 => /lib/aarch64-linux-gnu/libm.so.6 (0x00000040008cb000)
	libdl.so.2 => /lib/aarch64-linux-gnu/libdl.so.2 (0x0000004000976000)
	libc.so.6 => /lib/aarch64-linux-gnu/libc.so.6 (0x0000004000989000)
	/lib/ld-linux-aarch64.so.1 (0x0000004000870000)
	libz.so.1 => /lib/aarch64-linux-gnu/libz.so.1 (0x0000004000ad3000)
root@6e530a973d45:/# exit
% docker cp -L 6e530a973d45:/sbin/tc .
% docker cp -L 6e530a973d45:/lib/ld-linux-aarch64.so.1 .
% docker cp -L 6e530a973d45:/usr/lib/aarch64-linux-gnu/libelf.so.1 .
% docker cp -L 6e530a973d45:/lib/aarch64-linux-gnu/libm.so.6 .
% docker cp -L 6e530a973d45:/lib/aarch64-linux-gnu/libdl.so.2 .
% docker cp -L 6e530a973d45:/lib/aarch64-linux-gnu/libc.so.6 .
% docker cp -L 6e530a973d45:/lib/aarch64-linux-gnu/libz.so.1 .
```
	  
The above workflow is automated by
the <a href="https://github.com/gokrazy/freeze"><code>freeze</code>
tool</a>.

Now we can copy the contents of the temporary directory to
e.g. <code>/perm/tc</code> and run the <code>tc</code> command in
breakglass:
	  
```text
/tmp/breakglass531810560 # wget -O- http://10.0.0.76:4080/tc.tar | tar xf -
/tmp/breakglass531810560 # LD_LIBRARY_PATH=$PWD ./ld-linux-aarch64.so.1 ./tc
Usage: tc [ OPTIONS ] OBJECT { COMMAND | help }
…
```
