# Chapter 22: Organizing Your Application

Throughout the book we have described various components of an application built on top of Kubernetes.
We have described how to wrap programs up as containers, place those containers in Pods, replicate those Pods with ReplicaSets, and roll them out with Deployments.
We have even desctibed how to deploy stateful and real-world applications that collect these objects into a single distributed system.
But we have not covered how to actually work with such an application in a practical way.
How can you lay out, share, manage, and update the various configurations that make up your application?
That is the topic of this chapter.

## Principles to Guide Us

Before digging into the concrete details of how to structure your application, it's worth considering the goals that drive this structure.
Obviously, reliability and agility are the general goals of developing a cloud native application in Kubernetes, but how does this relate to how you design your application's maintenance and deployment?
The following sections describe three principals that can guide you in designing a structure that best suits these goals.
The principles are:

- Treat filesystems as the source of truth
- Conduct code review to ensure the quality of changes
- Use feature flags to stage rollouts and rollbacks

### Filesystems as the Source of Truth

When you first begin to explore Kubernetes, as we did in the beginning of this book, you generally interact with it imperatively.
You can run commands like `kubectl run` or `kubectl edit` to cretate and modify Pods or other objects running in your cluster.
Even when we started exploring how to write and use YAML files, this was presented in an ad-hoc manner, as if the file itself is just a way station on the road to modifying the state of the cluster.
In reality, in a true productionized application the opposite should be true.

Rather than viewing the state of the cluster--the data in `etcd`--as the source of truth, it is optimal to view the filesystem of YAML objects as the source of truth for your application.
The API objects deployed into your Kubernetes cluster(s) are then a reflection fo the truth stored in the filesystem.

There are numerous reasons why this is the right point of view.
The first and foremost is that is largely enables you to treat your cluster as if it is immutable infrastructure.
As we moved into cloud native architectures. we have become increasingly comfortable with the notion that our applications and their containers are immutable infrastructure, but treating a cluster as such is less common.
And yet, the same reasons for moving our applications to immutable infrastructure apply to our clusters.
If your cluster is a snowflake, you made by applying random YAML files downloaded from the internet ad hoc, it is as dangerous as a virtual machine built from imperative bash scripts.

Additionally, managing the cluster state via the filesystem makes it very easy to collaborate with multiple team members.
Source-control systems are well understood and can easily enable multiple people to edit the state of the cluster simultaneously, while making conflicts (and the resolution of those conflicts) clear to everyone.

**NOTE**
It is absolutely a first principle that all applications deployed to Kubernetes should first be descrived in files stored in a filesystem.
The actual API objects are then just a projection of this filesystem into a particular cluster.
**EON**

### The Role of Code Review

It wasn't long ago that code review for application source code was a novel idea.
But it is clear now that multiple people looking at a piece of code before it is committed to an application is a best practice for producing high-quality, reliable code.

It is therefore suprising that the same is somewhat less true for the configurations used to deploy those applications.
All of the same reasons for reviewing code apply directly to application configurations.
But when you think about it, it is also obvious that code review of these configurations is critical to the reliable deployment of services.
In our experience, most service outages are self-inflicted via expected consequences, typos, or other simple mistakes.
Ensuring that at least two people look at any configuration change significantly decreases the probability of such errors.

**NOTE**
The second principle of our application layout is that it must facilitate the review of every change merged into the set of files that represents the source of truth for our cluster.

### Feature Gates

Once your application code and your deployment configuration files are in source control, one of the most common questions is how these respositories relate to one another.
Should you use the same repository for application source code and configuration?
This can work for small projects, but in larger projects it often makes sense to separate the two.
Even if the same people are responsible for both building and deploying the application, the perspectives of the builder versus those of the deployer are different enough that this separation of concern makes sense.

If this is the case, then how do you bridge the development of new features in source control with the deployment of those features into production environment?
This is where feature gates play an important role.

The idea is that when some new feature is developed, that development takes place entirely behind a feature flag or gate.
The gate looks something like

```
if (featureFlags.myFlag) {
    // Feature implementation goes here
}
```

There are a variety of benefits to this approach.
First, it lets the team commit to the production branch long before the feature is ready to ship.
This enables feature development to stay much more closely aligned with the `HEAD` of a repository, and thus you avoid the horrendous merge conflicts of a long-lived branch.

Working behing a feature flag also means that enabling a feature simply involves making a configuration change to activate the flag.
This makes it very clear what changed in the production environment, and very simple to roll back the feature activation if it causes problems.

Using feature flags thus both simplifies debugging and ensure that disabling a feature doesn't require a binary rollback to an older version of the code that would remove all of the bug fixes and other improvements made by the newer version.

**NOTE**

The third principle of application layout is that code lands in source control, by default off, behind a feature flag, and is only activated through a code-reviewed change to configuration files.
**EON**

## Managing Your Application in Source Control

Now that we have determined that the filesystem should represent the source of truth of your cluster, the next important question is how to actually lay out the files in the filesystem.
Obviously, filesystems contain hierachical directories, and a source-control system adds concepts like tags and branches, so this section descrives how to put these together to represent and manage your application.

### Filesystem Layout

This section descrives how to lay out an instance of your application for a single cluster.
In later sections, we will descrive how to parameterize this layout for multiple instances.
It's worth getting this organization right when you begin.
Much like modifying the layout of packages in source control, modifying your deployment configurations after the fact is a complicated and expensive refactor that you'll probably never get around to.

The first cardinality on which you want to organize your application is the semantic componenet or layer (for instance, frontend or batch work queue).
Though early on this might seem like overkill, since a single team manages all of these components, it sets the stage for team scaling--eventually, different teams (or subteams) may be responsible for each of these components.

This, for an application with a frontend that uses two services, the filesystem might look like this

```
frontend/
service-1/
service-2/
```

Within each of these directories, the configurations for each application are stored.
These are the YAML files that directly represent the current state of the cluster.
It's generally useful to include both the service name and the object type within the same file.

**NOTE**
While Kubernetes allows you to create YAML files with multiple objects in the same file, this is generally an antipattern.
The only good reason to group several objects in the same file is if they are conceptually identical.
When deciding what to include in a single YAML file, consider design principles similar to those for defining a class or struct.
If grouping the objects together doesn't form a single concept, they probably shouldn't be in a single file.
**EON**

Thus, extending our previous example, the filesystem might look like:
```
frontend/
   frontend-deployment.yaml
   frontend-service.yaml
   frontend-ingress.yaml
service-1/
   service-1-deployment.yaml
   service-1-service.yaml
   service-1-configmap.yaml
...
```

### Managing Periodic Versions

What about managing releases?
It is very useful to be able to look back and see what your application deployment previously looked like.
Similarly, it is very useful to be able to iterate a configuration forward while still deploying a stable release configuration.

Consequently, it's handy to be able to simultaneously store and maintain multiple revisions of your configuration.
There are two different approaches that you can use with the file and version control systems we've outlined here.
The first is to use tags, branches, and source-control features.
This is convenient because it maps the way people manage revisions in source control, and leads to a more simplified directory structure.
The other option is to clone the configuration within the filesystem and use directories for different revisions.
This makes viewing the configurations simultaneously very straightforward.

These approaches have the same capabilities in terms of managing different release versions, so it is ultimately an aesthetic choice between the two.
We will discuss both approaches and let you or your team decide which you prefer.

#### Versioning with branches and tags

When you use branches and tags to manage configuration revisions, the directory structure does not change from the example in the previous section.
When you are ready for a release, you place a source-control tag (such as `git tag v1.0`) in the configuration source-control system.
The tag represents the configuration used for the version, and the `HEAD` of source control continues to iterate forward.

Updating the release configuration is somewhat more complicated, but the approach models what you would do in source control.
First, you commit the change to the `HEAD` of the repository.
Then you create a new branch named `v1` at the `v1.0` tag.
You cherry-pick the desired change onto the release branch (`git cherry-pick <edit>`), and finally you tag this branch with the `v1.1` tag to indicate a new point release.

**NOTE**
One common error when cherry-picking fixes into a release branch is to only pick the changes into the latest release.
It's a good idea to cherry-pick it into all active releases, in case you need to roll back versions but the fix is still needed.
**EON**