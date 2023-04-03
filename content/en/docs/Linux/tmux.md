---
title: "tmux"
weight: 20
description: >
  tmux cheatsheet.
---

`tmux` (terminal multiplexer) is a Linux program that lets you open multiple terminal windows and sessions in the same terminal window.

## Sessions



Create a new session:
```shell
$ tmux
```
When you have multiple windows, the `*` indicates what window you have in focus.

Create a new horizontal pane with `CTRL + b` and then `%` (`SHIFT` + `5`).
To switch between horizontal panes, use `CTRL + b` and then the arrow key (`->` or `<-`) that points to the pane that you want to put in focus.

Create a vertical pane beneath the in-focus pane, use `CTRL + b` and the quotation mark (`"`). 
To switch between vertical panes, use `CTRL + b` and then the up or down arrow key that points to the pane that you want to put in focus.

To close a pane, either use `CTRL + D` or type `exit`.

### Detach a session

`tmux` sessions preserve state.

To detach from a session, use `CTRL + b` and `d`. This brings you back to your normal terminal session. 

To view the sessions that are currently running, enter the following:

```shell
$ tmux ls
0: 2 windows (created Sun Apr  2 22:34:09 2023)
```
In the previous example, `0` is the name of the session. `0` is the default session name.
To rejoin a session, enter the following: 

```shell
$ tmux attach -t <session-name>
```
To rename a session, enter the following:

```shell
$ tmux rename-session -t <current-name> <new-name>
```
To give a session a custom name from the outset:
```shell
$ tmux new -s <session-name>
```
To delete a session:
```shell
$ tmux kill-session -t <session-name>
```

`CTRL + b`



## Windows

Create a new window with `CTRL + b` and `c`. This opens up a new window. 
To switch between windows, use `CTRL + b` and the number of the window, e.g. `0`. Terminal windows are zero-indexed.

rename your window with `CTRL + b` and a comma (`,`). Then, type the new name.

To close a window, either use `CTRL + D` or type `exit`.