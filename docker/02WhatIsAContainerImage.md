# Course 2: wat is een container image?

https://www.katacoda.com/courses/container-runtimes/what-is-a-container-image

## Container Image

een tar file die tar files bevat. Bezie het als een zip files waarin zip files zitten en zo een structuur opmaken. Als alles is uitgepakt heeft men de volledige filestructure die nodig is.

aantonen hoe het in elkaar zit:

- This can be explored via Docker. Pull the layers onto your local system.  
  `docker pull redis:3.2.11-alpine`
- Export the image into the raw tar format.  
  `docker save redis:3.2.11-alpine > redis.tar`
- Extract to the disk  
  `tar -xvf redis.tar`
- All of the layer tar files are now viewable.  
  `ls`
- The image also includes metadata about the image, such as version information and tag names.  
  `cat repositories`  
  `cat manifest.json`
- Extracting a layer will show you which files that layer provides.  
  `tar -xvf da2a73e79c2ccb87834d7ce3e43d274a750177fe6527ea3f8492d08d3bb0123c/layer.tar`

## Creating Empty Image

tar file maken:
`tar cv --files-from /dev/null | docker import - empty`

`docker images`

## Creating image without Dockerfile

het idee uit [Creating Empty Image](#creating-empty-image) kan uitgebreid worden door een hele image te maken:

Busybox gebruiken als basis, geeft ons foundation aan linux commands

`curl -LO https://raw.githubusercontent.com/moby/moby/a575b0b1384b2ba89b79cbd7e770fbeb616758b3/contrib/mkimage/busybox-static && chmod +x busybox-static`

`ls -lha busybox`

default busybox roofts doesnt include version information so lets create a file:

`echo KatacodaPrivateBuild > busybox/release`

As before, the directory can be converted into a tar and automatically imported into Docker as an image.

`tar -C busybox -c . | docker import - busybox`

This can now be launched as a container.

`docker run busybox cat /release`