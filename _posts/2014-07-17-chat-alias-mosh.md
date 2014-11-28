---
layout: post
title: chat alias mosh
author: klein2de
---
Gehen wir davon aus, dass Du eine screen-Session auf dem Server laufen hast und darin läuft irssi oder weechat-curses. Dahin verbindest Du Dich häufig. Es empfiehlt sich daher, sich dafür einen Alias anzulegen, der so in die .bashrc bzw. unter OSX in die .bash_profile gespeichert werden kann.

Voraussetzung ist, dass mosh sowohl auf dem Server als auch auf Deinem Client installiert sind. Auch praktisch ist es, wenn bereits die Keys getauscht wurden.

`alias chat="mosh --ssh='ssh -p [PORT]' [USER]@[SERVER] -- screen -r -D"`

Verlasse das Terminal mit Exit und melde Dich erneut an. Nun ist der Alias verfügbar.