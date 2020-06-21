Basic documentation for fixing/tweaking WSL2 for my dev use. Some of
this is useful, some of it might be just anecdotal. It's mostly a log
of what I did so that if/when I have to reset/refresh windows and/or
start on a new laptop, I can get up-and-running faster.

# Networking

## DNS

Lots of problems with name resolution and networking in general.

- https://github.com/microsoft/WSL/issues/4139, windows defender firewall
- https://github.com/microsoft/WSL/issues/4246, checkpoint vpn
- https://github.com/microsoft/WSL/issues/4585, windows firewall
- https://github.com/microsoft/WSL/issues/5420, `wsl2 keeps
  overwriting resolv.conf`

**Symptom**: `/etc/resolv.conf` includes the *correct* IP address of the
host, yet pings do not get through and names are not resolved.

**Fix**:

- disable auto-generation of `resolv.conf` by adding this to the
  in-WSL `/etc/wsl.conf`:

    ```
    [network]
    generateResolvConf = false
    ```

- `rm /etc/resolv.conf` (because it's a symlink to
  `/run/resolvconf/resolv.conf`)

- add `nameserver realnameserver` where "realnameserver" is the ip
  address of your actual local DNS name server (I used my local
  network router, a caching name server)

## `localhost`

Other complaints about networking involve the misconception that
`localhost` on the linux machine is the same as `localhost` in
windows, disregarding the whole host/guest-vm relationship. These are
just misunderstandings. While my first reaction is *"localhost is
unique for each"*, but apparently linux ports are accessible in
windows as `localhost` but not vice versa. I have no idea.

For the reverse, you'll likely need the windows IP address to get to
it from linux.

I'd like to automatically add the host IP address to `/etc/hosts`;
currently docker is adding its own stuff there, perhaps I need to find
or write a script that auto-fills (perhaps based on `ip route ls` or
similar?).

# SSH-agent Forwarding

Long-debated. I prefer using [KeePass](https://keepass.info/) and
[KeeAgent](https://lechnology.com/software/keeagent/)
([github](https://github.com/dlech/KeeAgent/)) instead of windows'
openssh key-management, so many of the links proved troublesome and/or
just unrelatable.

Long-story-short:

- in Windows, I have KeePass/KeeAgent configured with its
  cygwin-compatible socket file (vice its msysGit-compatible), and the
  `SSH_AUTH_SOCK` envvar is set in the windows *Environment Variables*
  thing;

- in linux, install `socat`; in debian-like, this is `apt install socat`;

- get `npiperelay.exe`; the "proper" way to go if you have `go`
  available on your windows machine is to go to
  https://github.com/jstarks/npiperelay and install it that way; an
  alternative is to download the `wsp-ssh-agent` package from
  https://github.com/rupor-github/wsl-ssh-agent/releases and extract
  the pre-compiled `npiperelay.exe` from its release and discard the
  rest (I'm currently looking at v1.4.2, commit dd85633, there might
  be differences later); save this `.exe` file somewhere on the
  windows side (does not need to be in the `PATH`, just somewhere
  accessible from the linux side);

- you can go as quick as using
  https://github.com/dlech/KeeAgent/issues/159#issuecomment-647083522:

    ```bash
    export SSH_AUTH_SOCK=$HOME/.ssh/agent.sock
    ss -a | grep -q $SSH_AUTH_SOCK
    if [ $? -ne 0   ]; then
        rm -f $SSH_AUTH_SOCK
        ( setsid socat UNIX-LISTEN:$SSH_AUTH_SOCK,fork EXEC:"/mnt/c/Tmp/npiperelay.exe -ei -s //./pipe/openssh-ssh-agent",nofork & ) >/dev/null 2>&1
    fi
    ```

    (changing the location of `npiperelay.exe`) though I don't
    understand why the need for `setsid`, and there is no PID tracking
    here. I've drafted a hackish script to do ever-so-slightly more,
    found in `./assets/wsl-ssh-agent-forwarder`.

(I should note that I don't know how the `//./pipe/openssh-ssh-agent`
is used. For instance, my windows-side `$SSH_AUTH_SOCK` is pointing to
a non-standard socket name ... if there is a standard ... and even
though I never define it, things work nonetheless. However, if I
change from that to any other string, it stops working, so ... keep
it.)

# X-windows Forwarding

Still WIP. My biggest issue is that I feel the need for something
`screen`-like (or `tmux`-like) with X windows, so that when (not if)
the network between windows and WSL2 glitches, I can reconnect and
continue using the windows. I'm told [Xpra](https://xpra.org/) is a
good fit for this.

But for starters:

- windows: download/install xpra client;
- linux: debian-like: `apt install xpra`
- in linux, start : `xpra start --bind-tcp=0.0.0.0:10000 :10000` (you
  can optionally put this in the background by appending `&`, it works
  well that way); N.B., I started it in linux and then `Ctrl-C`d it,
  but it was still running; if you want to keep it foreground in a
  terminal, killing it is more than just an interrupt
- in windows, start Xpra, then "Connect", then `username@localhost:10000`
- in linux, set a display an start something, e.g., `DISPLAY=:10000 emacs &`

More testing is required to see how "robust" this is. I've heard that
some WSL2 installations suspend or stop the WSL2 VM when not in use;
since I use docker with the WSL2 extension, I believe that is keeping
WSL2 active at all times even when no terminals are open. YMMV.
