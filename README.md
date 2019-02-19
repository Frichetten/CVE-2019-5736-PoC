# CVE-2019-5736-PoC
PoC for CVE-2019-5736

Created with help from <a href="https://github.com/singe/container-breakouts">@singe</a>, @_cablethief, and <a href="https://github.com/feexd/pocs/tree/master/CVE-2019-5736">@feexd</a>

Tested on Ubuntu 18.04, Debian 9, and Arch Linux. Docker versions 18.09.1-ce and 18.03.1-ce. This PoC does not currently work with Ubuntu 16.04 and CentOS.

Go checkout the exploit code from Dragon Sector (the people who discovered the vulnerability) <a href="https://www.openwall.com/lists/oss-security/2019/02/13/3">here</a>.

### What is it?
This is a Go implementation of CVE-2019-5736, a container escape for Docker. The exploit works by overwriting and executing the host systems runc binary from within the container.

## How does the exploit work?
There are **2** use cases for the exploit. The first (which is what this repo is), is essentially a trap. An attacker would need to get command execution inside a container and start a malicious binary which would listen. When someone (attacker or victim) uses `docker exec` to get into the container, this will trigger the exploit which will allow code execution as root.

<p align="center"><img src="/screenshots/example1.gif"></p>

The second (which is **not** what this repo is), creates a malicious Docker image. When that image is run, the exploit will fire. No need to exec into the container. See the bottom of this readme for a gif example. 

### What do you need?
To exploit this vulnerability you need to have root (uid 0) inside the container.

### Are there side effects?
**Yes, you will overwrite your implementation of runc which will ensure your system will no longer be able to run Docker containers. Please backup either /usr/bin/docker-runc or /usr/bin/runc (depending on which you have; also check /usr/sbin).**

### How do I run it?
Modify the code however you see fit and compile it with `go build main.go`. Move that binary to the container you'd like to escape from. Execute the binary, and then the next time someone attaches to it and calls `/bin/sh` your payload will fire.

### Step by step explanation
This PoC was created using an excellent explanation from <a href="https://github.com/lxc/lxc/commit/6400238d08cdf1ca20d49bafb85f4e224348bf9d">this</a> commit to the lxc project (along with some helpful advice from others).

> As an example, if the target binary was /bin/bash, this could be replaced with an executable script specifying the interpreter path #!/proc/self/exe (/proc/self/exec is a symbolic link created by the kernel for every process which points to the binary that was executed for that process). As such when /bin/bash is executed inside the container, instead the target of /proc/self/exe will be executed - which will point to the runc binary on the host.

We implement this by overwriting `/bin/sh` in the container with `#!/proc/self/exe` which will point to the binary that started this process (the Docker exec).
<p align="center"><img src="/screenshots/1.png"/></p>

> The attacker can then proceed to write to the target of /proc/self/exe to try and overwrite the runc binary on the host. However in general, this will not succeed as the kernel will not permit it to be overwritten whilst runC is executing. To overcome this, the attacker can instead open a file descriptor to /proc/self/exe using the O_PATH flag and then proceed to reopen the binary as O_WRONLY through /proc/self/fd/<nr> and try to write to it in a busy loop from a separate process. 

Note: Some parts of the previous section are not entirely accurate. You do not need to use the O_PATH flag when getting the file descriptor for runcinit. Additionally you do not need to create the write loop in another process. We get a file descriptor for the `runcinit` by getting a file handle to `/proc/PID/exe`. From there, we then use that handle to get a file handle to `/proc/self/fd/FILEDESCRIPTOR`. This is the file handle we will use for writing.

<p align="center"><img src="/screenshots/2.png"></p>

> Ultimately it will succeed when the runC binary exits. After this the runC binary is compromised and can be used to attack other containers or the host itself.

If we are able to write to that file handle we have overwritten the `runc` binary on the host. We are able to execute arbitrary commands as root.

### Example of malicious Docker image
This repo does not contain this example but you can find it <a href="https://github.com/q3k/cve-2019-5736-poc">here</a>. To my mind, this is the much more dangerous scenario. Simply run the malicious container image and you get code execution as root. For a detailed explanation please see <a href="https://blog.dragonsector.pl/2019/02/cve-2019-5736-escape-from-docker-and.html">this</a>.

<p align="center"><img src="/screenshots/dangerous.gif"></p>
