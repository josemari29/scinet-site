---
title: Apptainer on Ceres
description: A guide for building and running containers on Ceres
permalink: /guide/apptainer
author: VRSC
layout: page
---

#### Table of Contents
1. [Introduction](#1-introduction)
2. [Prerequisites](#2-prerequisites)
3. [Container Images](#3-container-images)
4. [Docker Images](#4-docker-images)
5. [Apptainer Images](#5-apptainer-images)
6. [Executing Containers](#6-executing-containers)
7. [Ceres Container Repository](#7-ceres-container-repository)
8. [Support](#8-support)


# 1. Introduction

Apptainer [https://sylabs.io/](https://sylabs.io/) is an application for running containers on an HPC cluster. Containers are self-contained application execution environments that contain all necessary software to run an application or workflow, so you don't need to worry about installing all the dependencies. There are many pre-built container images for scientific applications available for download and use, see section [Container Images](#3-container-images).



# 2. Prerequisites

To run containers on Ceres, you'll need to execute the singularity command from a compute node. For example, to run an interactive session on a compute node, use the SLURM salloc command:

```bash
[user.name@ceres ~]$ salloc
salloc: Granted job allocation 1695904
salloc: Waiting for resource configuration
salloc: Nodes ceres14-compute-36 are ready for job
export TMPDIR=/local/bgfs//1695904
export TMOUT=5400
[user.name@sn-cn-11-2 ~]$ type apptainer
apptainer is hashed (/usr/bin/apptainer)  
```

NOTE: salloc by default runs on a single hyper-threaded core (2 logical cores) with 6000 MB of allocated memory on one of the compute nodes. The session will last for 2 days, but will timeout after 1.5 hours of inactivity (no commands runnning). See the [Ceres User Manual](/guide/ceres/) for more info on how to request resources for interactive jobs.

NOTE: on Atlas apptainer is available through environment module. Before issuing any singularity command, load the module by issuing "module load apptainer" command.


# 3. Container Images

Apptainer executes a container from a Apptainer container image either created by the user or downloaded from Apptainer Library [https://cloud.sylabs.io/library](https://cloud.sylabs.io/library), and can also import and execute Docker [https://www.docker.com/](https://www.docker.com/) container images, either directly uploaded by the user, or downloaded from Docker Hub [https://hub.docker.com/](https://hub.docker.com/).

BioContainers is a "community-driven project that provides the infrastructure and basic guidelines to create, manage and distribute Bioinformatics [Docker] containers with special focus in Proteomics, Genomics, Transcriptomics and Metabolomics."

BioContainers can be obtained either via docker [https://hub.docker.com/u/biocontainers/](https://hub.docker.com/u/biocontainers/) or via Quay [https://quay.io/](https://quay.io/)

For leveraging GPU using containers, Nvidia provides a container library NGC. [https://ngc.nvidia.com/catalog/all](https://ngc.nvidia.com/catalog/all) 

> NGC offers a comprehensive catalog of GPU-accelerated software for deep learning, machine learning, and HPC. NGC containers deliver powerful and easy-to-deploy software proven to deliver the fastest results. By taking care of the plumbing, NGC enables users to focus on building lean models, producing optimal solutions and gathering faster insights.

# 4. Docker Images

While Apptainer can only execute containers from Apptainer images, it can easily import Docker images directly from Docker Hub to create a Apptainer image. For example, to download a Docker image of the R programming language from Docker Hub [https://hub.docker.com/_/r-base/](https://hub.docker.com/_/r-base/) and import it into a Apptainer image, change the docker pull rbase command listed at the aforementioned URL to the equivalent Apptainer command  `apptainer pull docker://r-base`  (where specifying "docker://" before the image name lets Apptainer know the image is a Docker image, and by default fetch the image from Docker Hub):

```bash
[user.name@sn-cn-11-2 ~]$ apptainer pull docker://r-base
INFO:    Converting OCI blobs to SIF format
INFO:    Starting build...
Getting image source signatures
Copying blob 1fcd5305bc72 done
 ...
INFO:    Creating SIF file...
INFO:    Build complete: r-base_latest.sif  
```

The resulting apptainer image (r-base.img) contains a complete environment (operating system, libraries, R installation) for running R.
A tag may be specified when selecting the Docker image to download; e.g., a list of tags for r-base is at: [https://hub.docker.com/r/library/r-base/tags/](https://hub.docker.com/r/library/r-base/tags/). To specify a specific tag, append ":TAG" to the image name (e.g.,  `apptainer pull docker://r-base:3.3.3` ). If the tag is omitted, Apptainer will look for an image labeled with the "latest" tag (note that the "latest" tag is merely a Docker Hub convention, and is not guaranteed to exist, nor is it guaranteed to point to the latest image—when in doubt, specify the tag).

**Note on Home directory and Apptainer**

While pulling/building the containers, pay attention to the home directory as the cached image blobs will be saved in ${HOME}/.apptainer .
Since the home directory has a limited amount of space, this can fill up quite easily. Users can change where the files will be cached by setting APPTAINER_CACHEDIR and APPTAINER_TMPDIR environment variables. On Ceres we set APPTAINER_TMPDIR to $TMPDIR. We also set APPTAINER_CACHEDIR to $TMPDIR for all Slurm jobs if it's not set by the user. As of January 13, 2022, these variables are not automatically set on Atlas. We recommend adding the following two commands to the job scripts that use apptainer:

```
export APPTAINER_CACHEDIR=$TMPDIR 
export APPTAINER_TMPDIR=$TMPDIR
```
One can also use --disable-cache option to avoid saving cached image blobs (e.g., `apptainer pull --disable-cache docker://r-base:3.3.3` ).

In case the home directory is full, it is safe to delete the contents of ~/.apptainer folder.


# 5. Apptainer Images

### Apptainer Library previously known as Singularity Library

Apptainer Library is a service that builds Apptainer images from community-provided recipes (called "bootstrap files") stored in GitHub repositories and makes them available for search and download. To download an apptainer image, search the Apptainer Library container collections, and follow the instructions to download and execute the container. This is similar to the directions for container
images from Docker Hub, except replace the "docker://" URI prefix with "library://".

### Creating Your Own Apptainer Images

Root access is needed to create a Apptainer image from a bootstrap file. As Ceres users do not have root access, to create your own Apptainer image, you can paste the bootstrap file to Apptainer cloud builder [https://cloud.sylabs.io/builder](https://cloud.sylabs.io/builder). 

Bootstrap examples - https://github.com/sylabs/apptainer/tree/master/examples


# 6. Executing Containers

Three commands may be used to execute applications inside Singularity containers:

  `apptainer run`,

  `apptainer exec`,

  and (less commonly) `apptainer shell`

Note that applications executing within a container do not have access to applications installed outside of the container (e.g., environment modules, or executables installed in /usr/bin on the compute node).

### `apptainer run`

Apptainer images may have a run script (Docker: ENTRYPOINT) that is executed when the apptainer run command is used. To see the run script defined by the container (if any), use the apptainer inspect command:

```bash
[user.name@sn-cn-11-2 ~]$ apptainer inspect --runscript r-base.img
#!/bin/sh
exec R "$@"
```

The output above shows that in the aforementioned Docker image for the R programming language retrieved from Docker Hub,  `apptainer run r-base.img`  will run R interactively in the container environment. Your home directory is automatically mounted in the container, so any files therein can be accessed (and R packages installed within the container, as illustrated in the example below):

```bash
[user.name@sn-cn-11-2 ~]$ apptainer run r-base.img
R version 3.4.0 (2017-04-21) -- "You Stupid Darkness"
Copyright (C) 2017 The R Foundation for Statistical Computing
Platform: x86_64-pc-linux-gnu (64-bit)
...
> install.packages("stringr")
Installing package into '/usr/local/lib/R/site-library'
(as 'lib' is unspecified)
Warning in install.packages("stringr") :
'lib = "/usr/local/lib/R/site-library"' is not writable
Would you like to use a personal library instead? (y/n) y
Would you like to create a personal library
/home/user.name/R/x86_64-pc-linux-gnu-library/3.4
to install packages into? (y/n) y
trying URL 'https://cran.rstudio.com/src/contrib/stringr_1.2.0.tar.gz'
Content type 'application/x-gzip' length 94095 bytes (91 KB)
==================================================
downloaded 91 KB
* installing *source* package 'stringr' ...
...
* DONE (stringr)
The downloaded source packages are in
'/tmp/RtmpwP5YNn/downloaded_packages'
> library(stringr)
>
```

### `apptainer exec`

A container image can contain many executables / scripts. The singularity exec command can be used to select which program to run in the container. For example, to run a simple R script using the Rscript command in the container, prefix the Rscript command with  `apptainer exec r-base.img`:

```bash
[user.name@sn-cn-11-2 ~]$ cat test.R
summary(c(1:10))
[user.name@sn-cn-11-2 ~]$ apptainer exec r-base.img Rscript test.R
Min. 1st Qu. Median Mean 3rd Qu. Max.
1.00 3.25 5.50 5.50 7.75 10.00
```

### `apptainer shell`

apptainer shell starts an interactive shell within the container image, allowing you to inspect files (and execute programs) within the container. For example, here we see that the r-base.img container image was created using Debian Linux as the base operating system:

```bash
[user.name@sn-cn-11-2 ~]$ type R
bash: type: R: not found
[user.name@sn-cn-11-2 ~]$ apptainer shell r-base.img
Apptainer: Invoking an interactive shell within container...
Apptainer r-base.img:~> type R
R is /usr/bin/R
Apptainer r-base.img:~> cat /etc/os-release
PRETTY_NAME="Debian GNU/Linux 9 (stretch)"
NAME="Debian GNU/Linux"
VERSION_ID="9"
VERSION="9 (stretch)"
ID=debian
HOME_URL="https://www.debian.org/"
SUPPORT_URL="https://www.debian.org/support"
BUG_REPORT_URL="https://bugs.debian.org/"
Singularity r-base.img:~> exit
exit
[user.name@sn-cn-11-2 ~]$
```

# 7. Ceres Container Repository

There are some containers locally available on Ceres at
/reference/containers

These are available via modules, so the user doesn't have to perform any additional tasks.

**List of containers**
(may be out of date, `ls /reference/containers` on Ceres for the most up to date info):

* RStudio
* agbase_interproscan
* antismash
* bamm_groopm_checkm_refinem
* bigscape
* braker
* cDNA_cupcake
* cactus
* cnvnator
* combine_gafs
* concoct
* dastool
* envi-jupyter
* faststructure
* funannotate
* gatk
* goanna
* intarna
* itasser
* jupyter_notebook
* kobas
* metawrap
* minimac4
* nanopolish
* nanopolish_500kbp
* opendronemap
* pb_assembly
* plasflow
* python36
* qiime
* qiime2
* redundans
* revbayes
* rgi
* roary
* salmon
* salsa
* toil_vg
* trinityrnaseq


# 8. Support

The SCINet Virtual Research Support Core (VRSC) can provide support for the Apptainer application itself, but has no control over the contents of and cannot be expected to support container images.

Questions about application-specific container images may be directed to the SCINet community via the [SCINet Forum](https://forum.scinet.usda.gov) (must have a SCINet account to access), or to the community responsible for maintaining the container image.
