# Creating and Running Containers

Kubernetes is a platform for creating, deploying, and managing distributed applications.
These applications come in many different shapes and size, but ultimately, they are all comprised of one or more programs that run on individual machines.
These programs accept input, manipulate data, and then return the results.
Before we can even consider building a distributed system, we must first consider how to build the applicaiton container images that contain these programs and make up the pieces of our distributed system.

Application programs are typically comprised of language runtime, libraries, and your source code.
In many cases, your application relies on external shared libraries such as `libc` and `libssl`.
These external libraries are generally shipped and shared components in the OS that you have installed on a particular machine.

This dependency on shared libraries cause problems when an application developed on a programmer's laptop has a dependency on a shared library that isn't available when the program is rolled out to the production OS.
Even when the development and production environments share the exact same version of the OS, problems can occur when developers forget to include dependent asset files inside a package that they deploy to production.

The traditional methods of running multiple programs on a single machine require that all of these programs share the same versions of shared libaries on thee system.
If different programs are developed by different teams or organizations, these shared dependencies add needless complexity and coupling between these teams.

A program can only execute successfully if it can be reliably deployed onto the machine where it should run.
Too often the state of the art for deployment involves running imperative scripts, which inevitably have twisty and byzantine failure cases.
This makes the task of rolling out a new version of all or parts of a distributed system a labour-intensive and difficult task.

In Chapter 1, we argued strongly for the value of immutable images and infrastructure.
This immutability is exactly what the container image provides.
As we will see, it easily solves all the problems of dependency management and encapsulation just described.

When working with application, it's often helpful to package them in a way that makes sharing them with other easy.
Docker, the default tool most people use for containers, makes it easy to package an exexutable and push it to a remote registry where it can later be pulled by others.
At the time of writing, container registries are available in all of the major public clouds, and services to build images in the cloud are also available in many of them.
You can also run your own registry using open source or commercial systems.
These registries make it easy for users to manage and deploy private images, while image-builder services provide easy integration with continous delivery systems.

For this chapter, and the remainder of the book, we are going to work with a simple example application that we build to show this workflow in action. You can find the application on GitHub.
https://github.com/kubernetes-up-and-running/kuard

Container images bundle a program and its dependencies into a single artifact under a root filesystem.
The most popular container image format is the Docker image format, which has been standardized by the Open Container Initiative to the OCI image format.
Kubernetes supports both Docker- and OCI-compatible images via Docker and other runtimes.
Docker images also include additional metadata used by a container runtime to start a running application instance based on the contents of the container image.

This chapter covers the following topics:

- How to package an application using the Docker image format

- How to start an application using the Docker container runtime.

## Container Images

For nearly everyone, their first interaction with any container technology is with a container image.
A container image is a binary package that encapsulates all of the files necessary to run a program inside of an OS container.
Depending on how your first experiment with containers, you will either build a container image from your local filesystem or downlaod a preexisting image from a container registry.
In either case, once the container image is present on your computer, you can run that image to produce a running application inside an OS container.

The most popular and widespread container image format is the Docker image format, which was developed by the Docker open source project for packaging, distributing, and running containers using the `docker` command. Subsequently, work has begun by Docker Inc, and other to standardize the container image format via the Open Container Initiative (OCI) project.
While the OCI standard achieved a 1.0 release milestone in mid 2017, adoption of these standards is proceeding slowly.
The Docker image format continues to be the de facto standard and is made up of a series of filesystem layers.
Each layer adds, removes, or modifies files from the preceding layer in the filesystem.
This is an example of an overlay filesystem.
The overlay system is used both when packaging up the image and when the image is actually being used.
During runtime, there are a variety of different concrete implementation of such filesystems, including `aufs`, `overlay`, and `overlay2`.

### Container Layering

The phrases "Docker image format" and "container images" may be a bit confusing.
The image isn't a single file but rather a specification for a manifest file that points to other files.
The manifest and associated files are often treated by users as a unit. The level of indirection allows for more efficent storage and transmittal.
Associate with this format is an API for uploading and downloading images to an image registry.

Container images are constructed with a series of filesystem layers, where each layer inherits and modifies the layer that came before it.
To help explain this in detail, let's build some containers.
Not that for correctness, the ordering of the layers should be bottom up, but for ease of understanding we take the opposite approach:

```
.
L___container A: a base operting system only such as Debian
    L___container B: build upon #A, by adding Ruby v2.1.10
    L___container C: build upon #A, by adding Goland v1.6

```

At this point we have three containers: A, B, and C.
B and C are forked from A and share nothing besides the base container's files. Taking it further, we can build on top of B by adding Ruby on Rails (version 4.2.6).
We may also want to support a legacy application that requires an older version of Ruby on Rails (e.g. version 3.2.X).
We can build a container image to support that applicaiton based on B also, planning to someday migrate the app to version 4:

```
. (Continuing from above)
L___container B: Build upon #A, by adding Ruby v2.1.10
    L___container D: build upon #B, by adding Rails v4.2.6
    L___container E: build upon #B, by adding Rails v3.2.X
```

Conceptually, each container image layer builds upon a previous one.
Each parent reference is a pointer.
While the example here is a simple set of containers, other real-world containers can be part of a larger extensive directed acyclic graph.
--------------------------
Container images are typically combined with a container configuration file, which provides instructions on how to set up the container environment and execute an application entry point.
The container configuration often includes information on how to set up networking, namespace isolation, resource constraints (cgroups), and what `syscall` restriction should be placed on a running container instance.
The container root filsystem and configuration file are typically bundled using the Docker image format.

Containers fall into two main categories:

- System Containers
- Application Containers

System containers seek to mimic virtual machines and often run a full boot process.
They often include a set of system services typically found in a VM, such as `ssh`, `cron`, and `syslog`.
When Docker was new, these types of containers where much more common.
Over time, they have come to be seen as poor practice and application containers have gained favor.

Application containers differ from system containers in that they commonly run a single program.
While running a single program per container might seem like an unnessary constraint, it provides the perfect level of granularity for composing scalable applicaitons and is a design philosophy that is leveraged heavily by Pods. We will examine how pods work in detail 