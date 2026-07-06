## We go on a giant tangent to see what spirlctl is doing

In [README.md](/README.md), at one point we tried running `spirlctl login` in a
chroot jail.
For fun, let's see what exactly that's doing.
Let's start with `strace`, just to make sure it's not doing anything strange,
then `tcpdump`, to see if we can find where it's getting that session token from.


### Strace

By the way, apparently despite the login failing, `spirlctl` created a config file
(`~/.spirl/config.json`) within the jail:
```shell
$ sudo cat root/.spirl/config.json
{"creds":{"token":""}}
```

I don't want to run `strace` from inside the jail, so I'm gonna put `sh` into
the jail, and use it to run a little script which prints the PID, and gives me
a chance to use it with `strace` in another terminal:
```shell
$ cp `which sh` .
$ ldd sh
	linux-vdso.so.1 (0x0000726e3c174000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x0000726e3be00000)
	/lib64/ld-linux-x86-64.so.2 (0x0000726e3c176000)
$ mkdir -p lib/x86_64-linux-gnu
$ mkdir lib64
$ for f in /lib/x86_64-linux-gnu/libc.so.6 /lib64/ld-linux-x86-64.so.2 lib64/; do cp $f ./$f; done
$ sudo chroot . ./sh -c 'echo "PID: $$"; echo "Press Enter..."; read x; ./spirlctl login'
PID: 553245
Press Enter...
```

In another terminal, I ran `sudo strace -fp 553577 2>trace.txt`.
In the "Press Enter..." terminal, I pressed Enter, it ran `spirlctl login`,
I got the login URL, etc etc.
So now, in trace.txt I've got 2420 lines worth of syscalls!
Starting with `sh` forking off `./spirlctl login` after I hit Enter:
```
read(0, "\n", 1)                        = 1
rt_sigprocmask(SIG_SETMASK, ~[RTMIN RT_1], NULL, 8) = 0
vfork(strace: Process 553584 attached
 <unfinished ...>
[pid 553584] rt_sigprocmask(SIG_SETMASK, [], ~[KILL STOP RTMIN RT_1], 8) = 0
[pid 553584] execve("./spirlctl", ["./spirlctl", "login"], 0x5a296de39960 /* 28 vars */ <unfinished ...>
```

I can see Golang setting up various blocks of memory:
```
[pid 553584] mmap(0x7cfbbb041000, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0x7cfbbb041000
[pid 553584] prctl(PR_SET_VMA, PR_SET_VMA_ANON_NAME, 0x7cfbbb041000, 4096, " Go: page alloc") = 0
```

...then setting up signal handlers with `rt_sigaction`, then trying to open
various files which don't exist in the jail:
* /etc/localtime
* GOROOT/lib/time/zoneinfo.zip
* /proc/self/exe
* /etc/os-release

It's kind of cool it's willing to keep going after not finding any of that stuff.

It uses `clone()` here and there as well. How many children did we end
up with over the course of a single `spirlctl login`?..

```
$ ack -ho "Process .*attached" trace.txt 
Process 553577 attached
Process 553584 attached
Process 553585 attached
Process 553586 attached
Process 553587 attached
Process 553588 attached
Process 553589 attached
Process 553590 attached
Process 553591 attached
Process 553592 attached
Process 553593 attached
Process 553594 attached
Process 553595 attached
```

Wow! I think I've seen this with Golang programs before, though.
As I recall, one thread is created for each core?.. I have 16 cores, but
looks like at the point where `spirlctl` is saying "Please navigate to the
login URL to complete the login process", it only has 12 threads:
```
$ pstree 554780
sh───spirlctl───12*[{spirlctl}]
```

Anyway, next I see some datadog configuration checks:
```
[pid 553584] newfstatat(AT_FDCWD, "/etc/datadog-agent/managed/datadog-agent/stable/application_monitoring.yaml", 0x822b7634448, 0) = -1 ENOENT (No such file or directory)
[pid 553584] newfstatat(AT_FDCWD, "/etc/datadog-agent/application_monitoring.yaml", 0x822b7634518, 0) = -1 ENOENT (No such file or directory)
[pid 553584] newfstatat(AT_FDCWD, "/etc/datadog-agent/application_monitoring.yaml", 0x822b76345e8, 0) = -1 ENOENT (No such file or directory)
[pid 553584] newfstatat(AT_FDCWD, "/etc/datadog-agent/managed/datadog-agent/stable/application_monitoring.yaml", 0x822b76346b8, 0) = -1 ENOENT (No such file or directory)
```

Interesting. I poked around in `strings spirlctl`, and see a bunch of DataDog
libraries being used:
```
$ strings spirlctl | ack -ho "github.com/DataDog/[^ /.@]+" | sort -u
github.com/DataDog/datadog-agent
github.com/DataDog/datadog-agentcloud
github.com/DataDog/datadog-go
github.com/DataDog/dd-trace-go
github.com/DataDog/go-libddwaf
github.com/DataDog/go-libddwafgithub
github.com/DataDog/go-runtime-metrics-internal
github.com/DataDog/go-sqllexer
github.com/DataDog/go-tuf
github.com/DataDog/orchestrioncloud
github.com/DataDog/semantic-core
github.com/DataDog/sketches-go
```

I guess the idea is that `spirlctl` might be used in automated scripts, and
you might want to be able to open a dashboard in DataDog and see where errors
are popping up?..
I'm not familiar enough yet with how `spirlctl` is used.

Alright, we're almost through the trace... next I see it reading config.json:
```
[pid 553584] openat(AT_FDCWD, "/root/.spirl/config.json", O_RDONLY|O_CLOEXEC <unfinished ...>
[pid 553584] read(5, "{\"creds\":{\"token\":\"\"}}", 512) = 22
```

It sees the stuff written to it on a previous run of `spirlctl login`.
Now I start seeing network stuff: it reads from /etc/resolv.conf, tries to
read from /etc/nsswitch.conf and /etc/hosts but they're not in the jail,
it connects to 127.0.0.53 (where systemd-resolved is listening), and then...
```
[pid 553594] write(6, "\223\277\1 \0\1\0\0\0\0\0\1\3api\5spirl\3com\0\0\1\0\1\0"..., 42 <unfinished ...>
[pid 553588] read(6, "\223\277\201\200\0\1\0\5\0\1\0\1\3api\5spirl\3com\0\0\1\0\1\300"..., 1232) = 275
[pid 553588] close(6)                   = 0
```

Cool, so it presumably did a DNS lookup for `api.spirl.com` (note the
`\3api\5spirl\3com` in the write/read trace).
If I do that myself, I see some AWS servers, with a mention of gRPC
(`k8s-spirl-grpcwebp-...`):
```
$ dig api.spirl.com
...etc...
api.spirl.com.		300	IN	CNAME	k8s-spirl-grpcwebp-ef4de965a5-532dd5f9c51ae759.elb.us-west-2.amazonaws.com.
k8s-spirl-grpcwebp-ef4de965a5-532dd5f9c51ae759.elb.us-west-2.amazonaws.com. 60 IN A 44.225.108.7
k8s-spirl-grpcwebp-ef4de965a5-532dd5f9c51ae759.elb.us-west-2.amazonaws.com. 60 IN A 52.89.92.44
k8s-spirl-grpcwebp-ef4de965a5-532dd5f9c51ae759.elb.us-west-2.amazonaws.com. 60 IN A 35.85.127.200
k8s-spirl-grpcwebp-ef4de965a5-532dd5f9c51ae759.elb.us-west-2.amazonaws.com. 60 IN A 34.215.188.226
```

Back in the trace, I see `spirlctl` connecting to... *all* of the IP addresses
we saw in `dig`'s output, one after the other, on port 53?..
```
[pid 553588] connect(5, {sa_family=AF_INET, sin_port=htons(53), sin_addr=inet_addr("34.215.188.226")}, 16) = 0
[pid 553588] close(5 <unfinished ...>
[pid 553588] connect(5, {sa_family=AF_INET, sin_port=htons(53), sin_addr=inet_addr("52.89.92.44")}, 16) = 0
[pid 553588] close(5 <unfinished ...>
[pid 553588] connect(5, {sa_family=AF_INET, sin_port=htons(53), sin_addr=inet_addr("44.225.108.7")}, 16) = 0
[pid 553588] close(5 <unfinished ...>
[pid 553588] connect(5, {sa_family=AF_INET, sin_port=htons(53), sin_addr=inet_addr("35.85.127.200")}, 16) = 0
[pid 553588] close(5 <unfinished ...>
```

...and then it seems to choose one of them to connect to on port 443.
It adds the resulting fd 5 to an epoll list, waits on that for a bit.
Meanwhile, another thread grabs some random junk and then writes something
to its fd 5.
Then the first thread wakes up, reads some stuff from fd 5 (that is,
api.spirl.com:443), and opens & reads the CA certificates file.
```
[pid 553588] connect(5, {sa_family=AF_INET, sin_port=htons(443), sin_addr=inet_addr("34.215.188.226")}, 16 <unfinished ...>
[pid 553588] epoll_ctl(3, EPOLL_CTL_ADD, 5, {events=EPOLLIN|EPOLLOUT|EPOLLRDHUP|EPOLLET, data={u32=1979711496, u64=4502960001345650696}} <unfinished ...>
...etc...
[pid 553594] getrandom("\xaa\x19\xaa\x9f\xb1\x41\x37\xf9\x94\xbe\x6a\x79\x10\xe5\x87\x31\x02\xa0\x8e\xc7\x15\x9d\xdc\xef\x24\xa3\x4e\x74\x30\xb1\xc3\xce", 32, 0) = 32
[pid 553594] write(5, "\26\3\1\5\353\1\0\5\347\3\0032Z\201l`\1\246\377\246 6\340\30~`f\344J\24\267\360"..., 1520 <unfinished ...>
...etc...
[pid 553588] <... epoll_pwait resumed>[{events=EPOLLIN|EPOLLOUT, data={u32=1979711496, u64=4502960001345650696}}], 128, 9999, NULL, 0) = 1
[pid 553588] read(5,  <unfinished ...>
[pid 553588] <... read resumed>"d\20\300\314\16\263\207\220\300\265\253}\31\311\224\234\2001\275n\30\366y\241r\222\265B\351\n\267\27"..., 4468) = 3761
[pid 553588] openat(AT_FDCWD, "/etc/ssl/certs/ca-certificates.crt", O_RDONLY|O_CLOEXEC <unfinished ...>
[pid 553588] <... openat resumed>)      = 6
[pid 553588] read(6,  <unfinished ...>
[pid 553588] <... read resumed>"-----BEGIN CERTIFICATE-----\nMIIH"..., 182141) = 182140
[pid 553588] close(6)                   = 0
```

Now another thread reads some binary-looking stuff from fd 5, betcha that's
HTTPS, and then another thread writes the message with the URL and session
token to stdout:
```
[pid 553589] read(5,  <unfinished ...>
[pid 553589] <... read resumed>"\27\3\3\4]]N \323|\325z\230]&\7\203z\255\370.\254\344\22\214OJ1\36\275.\301"..., 4864) = 1122
...etc...
[pid 553591] write(1, "Login URL: https://auth.api.spir"..., 442 <unfinished ...>
```

...and then we can see it presumably trying to open the browser, and failing
because there's no xdg-open in the jail:
```
[pid 553591] newfstatat(AT_FDCWD, "/usr/local/sbin/xdg-open", 0x822b77c4518, 0) = -1 ENOENT (No such file or directory)
[pid 553591] newfstatat(AT_FDCWD, "/usr/local/bin/xdg-open",  <unfinished ...>
[pid 553591] <... newfstatat resumed>0x822b77c45e8, 0) = -1 ENOENT (No such file or directory)
[pid 553591] newfstatat(AT_FDCWD, "/usr/sbin/xdg-open", 0x822b77c46b8, 0) = -1 ENOENT (No such file or directory)
...etc...
```

...and that's basically it!
That all mostly makes sense. It's truly just making an HTTPS call to one
endpoint of api.spirl.com, getting back a session token, and then displaying
it to the user as part of a login URL.

By the way, looks like our DNS sockets were always fd 5 and 6, and our
HTTPS socket was fd 5:
```
$ ack "connect(\(| resumed)" trace.txt 
[pid 553591] connect(5, {sa_family=AF_INET, sin_port=htons(53), sin_addr=inet_addr("127.0.0.53")}, 16 <unfinished ...>
[pid 553591] <... connect resumed>)     = 0
[pid 553594] connect(6, {sa_family=AF_INET, sin_port=htons(53), sin_addr=inet_addr("127.0.0.53")}, 16 <unfinished ...>
[pid 553594] <... connect resumed>)     = 0
[pid 553588] connect(5, {sa_family=AF_INET, sin_port=htons(53), sin_addr=inet_addr("34.215.188.226")}, 16) = 0
[pid 553588] connect(5, {sa_family=AF_INET, sin_port=htons(53), sin_addr=inet_addr("52.89.92.44")}, 16) = 0
[pid 553588] connect(5, {sa_family=AF_INET, sin_port=htons(53), sin_addr=inet_addr("44.225.108.7")}, 16) = 0
[pid 553588] connect(5, {sa_family=AF_INET, sin_port=htons(53), sin_addr=inet_addr("35.85.127.200")}, 16) = 0
[pid 553588] connect(5, {sa_family=AF_INET, sin_port=htons(443), sin_addr=inet_addr("34.215.188.226")}, 16 <unfinished ...>
[pid 553588] <... connect resumed>)     = -1 EINPROGRESS (Operation now in progress)
```


### Tcpdump

Let's use the same trick as above to get a PID, and start watching it with
`tcpdump` before actually running `spirlctl` in it.
I used `exec ./spirlctl login` this time, so `spirlctl` uses the same PID,
because I don't think there's a way to ask `tcpdump` to watch an entire
process group?.. actually maybe there is. Well anyway.
```
$ sudo chroot . ./sh -c 'echo "PID: $$"; echo "Press Enter..."; read x; exec ./spirlctl login'
PID: 559778
Press Enter...
```

OH WAIT!.. looks like `tcpdump` doesn't have a way to watch a specific
PID. I did find a `ptcpdump` (get it?.. `p` + `tcpdump`, for "process TCP
dump"?..) by some random person, which seems to use BPF to match sockets
to processes:
https://github.com/mozillazg/ptcpdump

...but there's a lot of code in there, and I don't get the sense this
is widely used, sooo I don't want to risk installing it.

Anyway, I think I actually got all the information I needed from `strace`.
