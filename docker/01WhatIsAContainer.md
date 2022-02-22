# Course 1: Wat is een container?

https://www.katacoda.com/courses/container-runtimes/what-is-a-container

## Processen

`docker run -d --name=db redis:alpine`

container launched process -> redis-server

`ps aux | grep redis-server`

docker top geeft info over proces van container:

```bash
$ docker top db
UID                 PID                 PPID                C                   STIME               TTY                 TIME                CMD
999                 600                 583                 0                   10:15               ?                   00:00:00            redis-server *:6379
```

`ps aux | grep 583`  
Dit commando geeft info over het parent process

`pstree -c -p -A $(grep dockerd)`  
Zal alle sub processen oplijsten. In dit geval van de dockerd (Docker) process

## Process directory

```bash
DBPID=$(pgrep redis-server)
echo Redis is $DBPID
ls /proc
```

`ls /proc/$DBPID`

`cat /proc/$DBPID/environ`

`docker exec -it db env`

## Namespaces

Fundamenteel stuk van containers. Namespace is de plaats die een container access toe heeft, wat hij mag zien. Het limiteert processes om niet alles van het systeem te zien, maar enkel de stukken die ze nodig hebben.

The available namespaces are:

- Mount (mnt)
- Process ID (pid)
- Network (net)
- Interprocess Communication (ipc)
- UTS (hostnames)
- User ID (user)
- Control group (cgroup)

## Unshare can launch contained processes

Namespaces kunnen niet enkel gebruikt worden in docker containers, ook bij gwne processen

`unshare --help`

With unshare it's possible to launch a process and have it create a new namespace, such as Pid. By unsharing the Pid namespace from the host, it looks like the bash prompt is the only

```bash
sudo unshare --fork --pid --mount-proc bash
ps
exit
```

## What happens when we share a namespace

Namespaces zijn inode locations. processen kunnen dus samen in 1 namespace zitten

List all the namespaces with:  
`ls -lha /proc/$DBPID/ns/`

NSEnter is nog een tool: wordt gebruikt om processen te attachen aan existing namespaces. goe voor debugging

`nsenter --help`

`nsenter --target $DBPID --mount --uts --ipc --net --pid ps aux`

In docker kun je dit ook, onderstaand commando koppelt nginx aan de db namespace

```bash
docker run -d --name=web --net=container:db nginx:alpine
WEBPID=$(pgrep nginx | tail -n1)
echo nginx is $WEBPID
cat /proc/$WEBPID/cgroup
```

als we de namespaces vergelijken van beide processen zien we dat ze alle2 naar dezelfde locatie pointen.

`ls -lha /proc/$WEBPID/ns/ | grep net`  
`ls -lha /proc/$DBPID/ns/ | grep net`

## Chroot

provides ability om een process te doen starten met een different root directory dan de parent OS.

## Cgroups (Control Groups)

limiteerd de amount of resources dat een process kan consumeren. Cgroups zijn values defined in specifieke bestanden in de `/proc` directory

mappings bekijken:  
`cat /proc/$DBPID/cgroup`

## What are the CPU stats for a process?

Wordt ook in files opgeslagen:  
`cat /sys/fs/cgroup/cpu,cpuacct/docker/$DBID/cpuacct.stat`

$DBID is het process id van de db container

CPU Shares limit is defined here:  
`cat /sys/fs/cgroup/cpu,cpuacct/docker/$DBID/cpu.shares`

tot slot de Memory config van een container is hier gestored:
`ls /sys/fs/cgroup/memory/docker/`  
deze zijn gestored op basis van **container ID**, in tegenstelling tot de vorige 2 die op process ID zijn.  

```bash
DBID=$(docker ps --no-trunc | grep 'db' | awk '{print $1}')
WEBID=$(docker ps --no-trunc | grep 'nginx' | awk '{print $1}')
ls /sys/fs/cgroup/memory/docker/$DBID
```

## How to configure cgroups?

In docker kun je memory limits controllen

by default hebben containers geen limit op het aantal memory dat ze verbruiken. Kan bekeken worden met volgend commando:  
`docker stats db --no-stream` **lijkt mij zeer interessant**

memory quotes zitten in een file `memory.limit_in_bytes`

aanpassen memory quote:  
`echo 8000000 > /sys/fs/cgroup/memory/docker/$DBID/memory.limit_in_bytes`

als je terug bestand uitleest, gaat er 7.629M staan.  
`cat /sys/fs/cgroup/memory/docker/$DBID/memory.limit_in_bytes`

tot slot docker stats opnieuw checken:  
`docker stats db --no-stream`

## Seccomp - AppArmor

AppArmor is een application defined profile dat beschrijft welk stuk van het systeem een proces kan accessen

It's possible to view the current AppArmor profile assigned to a process via  
`cat /proc/$DBPID/attr/current`

Seccomp legt beveiliging en limitations op, zo kunnen vb gelimiteerd worden welke system calls er gemaakt worden, alsook het blocken van bijvoorbeeld kernel modules of changing the file permissions

## Capabilities

groupings die zeggen wat een proces of user mag doen, permission toe heeft. might cover multiple system calls or actions, zoals changing system time of hostname

status file contained ook de capabilities flag. Processen kunnen zoveel mogelijk capabilities droppen om ervoor te zorgen dat ze secure zijn.

`cat /proc/$DBPID/status | grep ^Cap`

de flags zijn opgeslagen als bitmask die gedecodeerd kan worden met capsh

`capsh --decode=00000000a80425fb`
