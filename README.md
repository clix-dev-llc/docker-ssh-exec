docker-ssh-exec - Secure SSH key injection for Docker builds
================
Allows commands that require an SSH key to be run from within a `Dockerfile`, without leaving the key in the resulting image.

----------------
Overview
----------------
This program runs in two different modes:

* a server mode, run as the Docker image `mdsol/docker-ssh-exec`, which transmits an SSH key on request to the the client; and
* a client mode, invoked from within the `Dockerfile`, that grabs the key from the server, writes it to the filesystem, runs the desired build command, and then *deletes the key* before the filesystem is snapshotted into the build.

----------------
Installation
----------------
To install the server, just pull `mdsol/docker-ssh-exec` like any other Docker image.

To install the client, just grab it from the [releases page][1], uncompress the archive, and copy the binary to somewhere in your `$PATH`. Remember that the client is run during the `docker build...` process, so either install the client just before invoking it, or make sure it's already present in your source image. Here's an example of the code you might run in your source image, to prepare it for SSH cloning from GitHub:

    # install Medidata docker-ssh-exec build tool from S3 bucket "mybucket"
    curl https://s3.amazonaws.com/mybucket/docker-ssh-exec/\
    docker-ssh-exec_0.5.1_linux_amd64.tar.gz | \
      tar -xz --strip-components=1 -C /usr/local/bin \
      docker-ssh-exec_0.5.1_linux_amd64/docker-ssh-exec
    mkdir -p /root/.ssh && chmod 0700 /root/.ssh
    ssh-keyscan github.com >/root/.ssh/known_hosts


----------------
Usage
----------------
To run the server component, pass it the private half of your SSH key, either as a shared volume:

    docker run -v ~/.ssh/id_rsa:/root/.ssh/id_rsa --name=keyserver -d \
      mdsol/docker-ssh-exec -server

or as an ENV var:

    docker run -e DOCKER-SSH-KEY="$(cat ~/.ssh/id_rsa)" --name=keyserver -d \
      mdsol/docker-ssh-exec -server

The benefit of this second method is that OS X systems using a virtual Docker host cannot easily use Docker's shared volume feature with files on the OS X side. The drawback is that the kay data is exposed in the process list.

Then, run a quick test of the client, to make sure it can get the key:

    docker run --rm -it mdsol/docker-ssh-exec cat /root/.ssh/id_rsa

Finally, as long as the source image is set up to trust (or ignore) GitHub's server key, you can clone private repositories from within the `Dockerfile` like this:

    docker-exec-ssh git clone git@github.com:my_user/my_private_repo.git

The client first transfers the key from the server, writing it to `$HOME/.ssh/id_rsa` (by default), then executes whatever command you supply as arguments. Before exiting, it deletes the key from the filesystem.

Here's the command-line help:

    Usage of docker-ssh-exec:
      -key string
          path to key file (default "~/.ssh/id_rsa")
      -port int
          server receiving port (default 1067)
      -pwd string
          password for encrypted RSA key
      -server
          run key server instead of command
      -version
          print version and exit
      -wait int
          client timeout, in seconds (default 3)

The software quits with a non-zero exit code (>100) on any error -- except a timeout from the keyserver, in which case it will just ignore the timeout and try to run the build command anyway. If the build command fails, `docker-ssh-exec` returns the exit code of the failed command.


----------------
Known Limitations / Bugs
----------------
The key data is limited to 4096 bytes.

On macOS 10.14 or later, the default format of `ssh-keygen` will produce
an "OpenSSH private key" ([reference][2]). For example:

```
$ ssh-keygen -t rsa -b 4096 -C "...@email.com" -f ~/.ssh/before_rsa
Generating public/private rsa key pair.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in ${HOME}/.ssh/before_rsa.
Your public key has been saved in ${HOME}/.ssh/before_rsa.pub.
The key fingerprint is:
...
$ head -2 ~/.ssh/before_rsa
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAACmFlczI1Ni1jdHIAAAAGYmNyeXB0AAAAGAAAABAZOJlIwH
```

To use a passphrase, this library requires an actual "RSA private key".
To make `ssh-keygen` produce one, use the `-m` (key format) flag:

```
$ ssh-keygen -t rsa -b 4096 -C "...@email.com" -f ~/.ssh/after_rsa -m PEM
...
$ head -5 ~/.ssh/after_rsa
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: AES-128-CBC,70B1F7ECFCC66C9DF073996B92D3C01E

GNhm2zcN6oz+K9yZimDMx6w5PD+mDz7ylVulz+PnYVP5TVs4yZuVZF3GGlu/NYZ1
```

----------------
Contribution / Development
----------------
This software was created by Benton Roberts _(broberts@mdsol.com)_

To build it yourself, just `go get` and `go install` as usual:

    go get github.com/mdsol/docker-ssh-exec
    cd $GOPATH/src/github.com/mdsol/docker-ssh-exec
    go install


--------
[1]: https://github.com/mdsol/docker-ssh-exec/releases
[2]: https://serverfault.com/q/939909/167925
