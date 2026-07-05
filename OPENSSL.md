## OpenSSL stuff on my machine

In [README.md](/README.md), I ran `spirlctl` in a chroot jail, and in order
for it to access the internet, I had to copy a couple of files into the jail:
* /etc/resolv.conf
* /etc/ssl/certs/ca-certificates.crt

I forget where I saw these mentioned; I think I did a google search for
"minimalist chroot jail" or something.
I had known about them before, but not well enough to know offhand to copy
exactly those 2 files into the jail.

I got curious though, how does Golang know where to find these files?
I know /etc/resolv.conf is a classic Unix file, so it's probably baked
right into Golang.
And indeed, there it is:
```
~/repos/go$ ack /etc/resolv.conf
src/net/dnsclient_unix.go
369:	return getSystemDNSConfigNamed("/etc/resolv.conf")
381:	conf.dnsConfig.Store(dnsReadConfig("/etc/resolv.conf"))
```

Regarding ca-certificates.crt, it's actually baked in as well:
```
~/repos/go$ ack ca-certificates.crt
src/crypto/x509/root_linux.go
11:	"/etc/ssl/certs/ca-certificates.crt",                // Debian/Ubuntu/Gentoo etc.
```

Here's the full list of paths it checks:
```
// Possible certificate files; stop after finding one.
var certFiles = []string{
	"/etc/ssl/certs/ca-certificates.crt",                // Debian/Ubuntu/Gentoo etc.
	"/etc/pki/tls/certs/ca-bundle.crt",                  // Fedora/RHEL 6
	"/etc/ssl/ca-bundle.pem",                            // OpenSUSE
	"/etc/pki/tls/cacert.pem",                           // OpenELEC
	"/etc/pki/ca-trust/extracted/pem/tls-ca-bundle.pem", // CentOS/RHEL 7
	"/etc/ssl/cert.pem",                                 // Alpine Linux
}
```

Next, I got curious what part of the SSL stack on my machine looks for
those things.
Like... in Python, I know there's a built-in `ssl` module written in Python,
and a built-in `_ssl` module written in C, but what does `_ssl` use?..
```
$ ldd `which python3`
	linux-vdso.so.1 (0x00007eace6d13000)
	libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6 (0x00007eace6bff000)
	libz.so.1 => /lib/x86_64-linux-gnu/libz.so.1 (0x00007eace6be3000)
	libexpat.so.1 => /lib/x86_64-linux-gnu/libexpat.so.1 (0x00007eace6bb7000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007eace6800000)
	/lib64/ld-linux-x86-64.so.2 (0x00007eace6d15000)
```

Hmmm the Python binary itself has suspiciously few dependencies.
```
$ python3
Python 3.12.3 (main, Mar 23 2026, 19:04:32) [GCC 13.3.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import _ssl
>>> _ssl.__file__
'/usr/lib/python3.12/lib-dynload/_ssl.cpython-312-x86_64-linux-gnu.so'
```

Getting closer... we found the .so file for `_ssl`, now what does it depend on?
```
$ ldd /usr/lib/python3.12/lib-dynload/_ssl.cpython-312-x86_64-linux-gnu.so
	linux-vdso.so.1 (0x0000780e6289c000)
	libssl.so.3 => /lib/x86_64-linux-gnu/libssl.so.3 (0x0000780e6278f000)
	libcrypto.so.3 => /lib/x86_64-linux-gnu/libcrypto.so.3 (0x0000780e62200000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x0000780e61e00000)
	/lib64/ld-linux-x86-64.so.2 (0x0000780e6289e000)
```

Bingo, we've got `libssl` and `libcrypto`.
I threw them into `ldd` and it looks like `libssl` depends on `libcrypto`,
and they both depend on `libc`.

So what is `libssl`?.. is that OpenSSL?..
```
$ strings /lib/x86_64-linux-gnu/libssl.so.3 | grep -i openssl
OPENSSL_sk_new_null
OPENSSL_sk_find
OPENSSL_sk_push
OPENSSL_sk_free
...etc...
```

Yup!
And does it look for ca-certificates.crt?..
I poked around in `strings /lib/x86_64-linux-gnu/libssl.so.3` for a bit,
and in `man ssl`, but didn't find anything.
I googled around, and found this:
https://stackoverflow.com/a/40626999
...which says to look at `openssl version -d`.
```
$ openssl version -d
OPENSSLDIR: "/usr/lib/ssl"
```

Which by the way, is apparently baked into `libcrypto`, not `libssl`:
```
$ strings /lib/x86_64-linux-gnu/libssl.so.3 | grep /usr/lib/ssl
$ strings /lib/x86_64-linux-gnu/libcrypto.so.3 | grep /usr/lib/ssl
/usr/lib/ssl/ct_log_list.cnf
OPENSSLDIR: "/usr/lib/ssl"
/usr/lib/ssl
/usr/lib/ssl/private
/usr/lib/ssl/certs
/usr/lib/ssl/cert.pem
```

So what the heck?.. why /usr/lib/ssl and not /etc/ssl?..
Ha haaaa because the one is a symlink for the other:
```
$ ls -l /usr/lib/ssl
total 4
lrwxrwxrwx 1 root root   34 Jun  2 12:33 cert.pem -> /etc/ssl/certs/ca-certificates.crt
lrwxrwxrwx 1 root root   14 Mar 16  2022 certs -> /etc/ssl/certs
drwxr-xr-x 2 root root 4096 Jun 28 08:41 misc
lrwxrwxrwx 1 root root   20 Jun  2 12:33 openssl.cnf -> /etc/ssl/openssl.cnf
lrwxrwxrwx 1 root root   16 Mar 16  2022 private -> /etc/ssl/private
```

And who has done this?!
```
$ dpkg -S /usr/lib/ssl
openssl: /usr/lib/ssl
```

Ah, the `openssl` package itself. Cool.


## Bonus: DNS resolving stuff on my machine

Maybe worth a refresher, I believe systemd has taken over DNS resolution
these days. Aaaand yes it has:
```
$ ls -l /etc/resolv.conf
lrwxrwxrwx 1 root root 39 Jun 28  2023 /etc/resolv.conf -> ../run/systemd/resolve/stub-resolv.conf

$ cat /etc/resolv.conf
# This is /run/systemd/resolve/stub-resolv.conf managed by man:systemd-resolved(8).
# Do not edit.
...etc...
nameserver 127.0.0.53
options edns0 trust-ad
search gv.shawcable.net
```

And who is listening at 127.0.0.53?.. it's `systemd-resolved`!
```
$ sudo netstat -tulpn | grep 127.0.0.53
tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN      332899/systemd-reso 
udp        0      0 127.0.0.53:53           0.0.0.0:*                           332899/systemd-reso 

$ ps aux | grep systemd-reso 
systemd+  332899  0.0  0.0  22064 10408 ?        Ss   Jun28   0:32 /usr/lib/systemd/systemd-resolved
```
