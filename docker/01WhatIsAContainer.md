# Course 1: Wat is een container?

https://www.katacoda.com/courses/container-runtimes/what-is-a-container

## Processen

`docker run -d --name=db redis:alpine`

`ps aux | grep redis-server`

`docker top db`

`ps aux | grep <ppid>`

`pstree -c -p -A $(grep dockerd)`

## Process directory

## Namespaces

## Unshare can launch contained processes

## What happens when we share a namespace

## Chroot

## Cgroups (Control Groups)

## What are the CPU stats for a process?

## How to configure cgroups?

## Seccomp - AppArmor

## Capabilities