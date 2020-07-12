---
title: "Emacs Remote Rails Console"
date:  2020-07-12
tags:  emacs elisp rails ssh
---

As a rails dev, it's very helpful to connect to a remote instance within your
~~os~~ editor (which is Emacs, of course!). Let's add a nice binding to our
[Doom Emacs](https://github.com/hlissner/doom-emacs) config.

## Connecting to a rails instance running in docker

Since the servers I'm connecting to are usually running docker, I want to use
the following command inside of Emacs:

```bash
ssh -tt my.server.com 'docker exec -it rails c'
```

coupled with a ssh config that manages my bastions and user names/keys.

## Adding a new file to keep our Doom config clean

To prevent my config from becoming an unreadable mess, I split it into a few
private files. A new layer would be too much overhead ;)

```elisp
;; ~/.doom/+rails.el -*- lexical-binding: t; -*-

(setq
  rails/app '("app-1" "app-2")
  rails/server '("my.server.com" "my.other-server.com"))

(defun rails/remote-console ()
  "Start a remote console"
  (interactive)
  (let ((app (ivy-read "App: " rails/apps))
        (server (ivy-read "Server: " rails/servers)))
    (compile
       (format "ssh -tt %s.com 'docker exec -it %s %s'"
               server app "rails c") t)
    (rename-buffer (format "*rails %s@%s*" app server))))
```

It'd be pretty easy to change `ivy-read` for `helm`, for example. I also rename
the buffer so I can connect to different servers without closing the existing
connections.

## Keybindings & Adding to the config

Thanks to a great short [blog
article](https://rameezkhan.me/adding-keybindings-to-doom-emacs/) by Rameez
Khan, I figured out how to add the bindings to the rails file conveniently:

```elisp
;; ~/.doom/+rails.el -*- lexical-binding: t; -*-

(map! :leader
      (:prefix-map ("R" . "rails")
       (:prefix "c" . "console")
        :desc "remote" "r" #'rails/remote-console'))
               
(provide +rails)
```

This will enable our function under `SPC R c r` at any point. Now, we only need
to link our file in the main `config.el`:

```elisp
;;; $DOOMDIR/config.el -*- lexical-binding: t; -*-
(load! "+rails")
```

That's it! This integration gave me a pretty nice efficiency boost over going
through the shell since I have my source code and an org buffer with some
snippets available.
