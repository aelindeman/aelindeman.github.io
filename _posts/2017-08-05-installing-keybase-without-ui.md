---
title: Installing Keybase without the UI
excerpt: >-
  An experiment liberating the Keybase command line from the packaged, Electron-based UI.
tags:
  - keybase
  - linux
  - electron
published: false
---
I use [Keybase][keybase] regularly at work to exchange API keys, certificates, and stuff with my coworkers. I use it at home too, to securely store and sync files I'd rather not put on Dropbox, like my [Keepass][keepass] keychain. And I like it a lot!

The Keybase package includes a separate UI component, which is ~~[garbage](https://drewdevault.com/2016/11/24/Electron-considered-harmful.html)~~ an Electron app -- WebKit sans browser chrome -- so it's not particularly fast or responsive, and eats up a lot of RAM. So instead of installing their package the correct way, I've ripped out the binaries, and forego the UI entirely. I then also added two `systemd` units to cover automatically starting the `keybase` daemon and to attach the `kbfs` filesystem.

### Hacking apart the package

Keybase uploads their packages to <https://prerelease.keybase.io>. I downloaded the latest version of the Debian package and extracted its contents.

```shell
mkdir keybase-package
dpkg -x keybase_amd64.deb keybase-package/
```

Then I simply installed `fuse` (a dependency to `kbfsfuse`) and copied the binaries the Keybase package would have installed to `/usr/local/bin`.

```shell
sudo apt install fuse
cd keybase-package/usr/bin/
sudo cp keybase kbfsfuse kbnm /usr/local/bin/
```

So at this point, you can run `keybase` and it will work fine. The first time you call `keybase` you'll notice it fork a daemon process first. After the daemon is running you can also call `kbfsfuse ~/keybase` and it will mount the Keybase filesystem to your home folder.

### Installing Keybase as a service

I wanted to have this all happen when I log in, so I created two `systemd` units: one for the Keybase backend daemon, and another for the Keybase filesystem.

```ini
# ~/.config/systemd/user/keybase.service
[Unit]
Description=Keybase
After=network.target

[Service]
ExecStart=/usr/local/bin/keybase --debug service

[Install]
WantedBy=default.target
```

```ini
# ~/.config/systemd/user/keybase-kbfs.service
[Unit]
Description=Keybase KBFS
After=network.target keybase.service

[Service]
ExecStart=/usr/local/bin/kbfsfuse -debug /home/alex/Keybase

[Install]
WantedBy=default.target
```

Then I enabled and started them both:

```shell
systemctl --user daemon-reload
systemctl --user enable keybase keybase-kbfs
systemctl --user start keybase keybase-kbfs
```

If all is well you should see this under the service status:

```shell
~ $ systemctl --user status keybase
● keybase.service - Keybase
   Loaded: loaded (/home/alex/.config/systemd/user/keybase.service; enabled; vendor preset: enabled)
   Active: active (running) since Sat 2017-08-05 11:18:12 EDT; 30s ago
```

So now when I log in, Keybase is running and the filesystem is automatically mounted in home directory:

```shell
~ $ ls Keybase
private  public
```

And can use the Keybase CLI without issue:

```shell
~ $ keybase status
Username:      aelindeman
Logged in:     yes
```

And, most importantly, since the UI is not running, my laptop has an additional 200 megs or so of RAM available -- which, at home on my workhorse ThinkPad X220, is important.

[keybase]: https://keybase.io
[keepass]: https://keepass.info
