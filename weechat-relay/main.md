-------------------------------------------------------------------------------

subject: Tutorial
title: Using WeeChat as an IRC Relay
subtitle: Never miss a conversation
author: Tobias Koch &lt;<tobias.koch@gmail.com>&gt;
date: 1 September 2018

document-type: report
bcor: 0cm
div: 16
lang: en_US
papersize: a4
parskip: half
secnumdepth: 5
secsplitdepth: 0
tocdepth: 5

legal:

 Copyright © 2018 Tobias Koch

 Permission is granted to copy, distribute and/or modify this document under the
 terms of the GNU Free Documentation License, Version 1.3 or any later version
 published by the Free Software Foundation; with no Invariant Sections, no
 Front-Cover Texts, and no Back-Cover Texts.

 The full license text is available at <https://www.gnu.org/licenses/fdl-1.3.en.html>

-------------------------------------------------------------------------------


Prerequisites
===============================================================================

In order to be able to be always on and logged in, you should consider
installing the WeeChat client on an Internet server at a hosting provider or in
the cloud.

On your server, install the WeeChat program with plugins and scripts:

```sh
apt-get install weechat
```

Also create a dedicated user:

```sh
adduser --disabled-password chat-relay
```


Basic Configuration
===============================================================================

Launch WeeChat as user chat-relay:

```sh
su - chat-relay
weechat
```

In the weechat command buffer create a handle for a new server, for example:

```
/server add freenode chat.freenode.net/6697 -ssl -autoconnect
```

Set your default real name (appears in whois queries):

```
/set irc.server_default.realname "Max Mustermann"
```

If you want to authenticate your nick, you need to do a variation of the
following:

```
/set irc.server.freenode.nicks nickname
/set irc.server.freenode.sasl_mechanism plain
/set irc.server.freenode.sasl_username nickname
/set irc.server.freenode.sasl_password secret
```

For the sake of convenience, you may also want to enable mouse handling:

```
/mouse enable
```


Configure the Relay
===============================================================================

Create the folder `.weechat/ssl`, change into it and generate a self-signed
certificate:

```
openssl req -nodes -newkey rsa:2048 -keyout relay.pem -x509 -days 365 -out relay.pem
```

Before configuring a relay, secure everything with a password and make sure the
certificate and key are loaded:

```
/set relay.network.password secret
/relay sslcertkey
```

Add a relay supporting the WeeChat protocol via

```
/relay add ssl.weechat 9000
```

Autostart
===============================================================================

In order to launch WeeChat automatically on system boot, create system service
file `weechat.service` in `/etc/systemd/system` with the following contents:

```
[Unit]
Description=Weechat IRC Client (in tmux)

[Service]
User=chat-relay
Type=forking
ExecStart=/usr/bin/tmux -2 new-session -d -s weechat /usr/bin/weechat
ExecStop=/usr/bin/tmux kill-session -t weechat

[Install]
WantedBy=multi-user.target
```

And remember to enable the service by running

```sh
systemctl enable weechat
systemctl start weechat
```

as root.

