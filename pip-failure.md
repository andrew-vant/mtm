pip, virtualenv, packaging

giving up on things like "not having to install the same dependency six times for six different programs".

Some time ago at work, I had cause to upgrade the python Requests library via pip. It broke pip. That was not the behavior I expected. In investigating the problem I did a lot of reading about pip, APT, virtualenv, and linux packaging in general.

I concluded that most packaging systems work pretty well right up until they interact with each other.

What happened to me was something like this: There is a system provided version of pip, installed by APT. APT knows what it depends on to work. Among those dependencies is that the requests library must exist and be within a certain version range.

Pip itself is also a package manager; it handles only python packages, usually from PyPI, and it has its own dependency management. But my copy came with the system; whatever data structure it uses to track dependency limitations doesn't seem to include itself.

When I pip installed Requests, I picture something like this happening: Pip is installed as a system package via APT, and had a dependency on requests vX.0 <= version <= vY.0 (no idea what the specific version numbers were). If I'd tried to upgrade requests via APT, it probably would have complained "hey, can't do that, your upgrade is too new for this other package and shit will break."

But the newer requests was only available via pip, so I installed it via pip. Pip looked at it, looks at its own dependency tree (I assume, I don't know the nuts and bolts under pip), sees no conflict with any other pip-installed package, and does what I told it to do without complaint. Then it blows up.

The way most of the Internet says to handle this is with virtualenv, which gives you a whole new python installation independent of the system one -- APT doesn't touch it at all. Yay? Well, no, not yay.

I don't like virtualenv, and I don't like pip. Virtualenv seems like a step backwards, a sacrifice of certain things I've come to expect, like not having to install six copies of the same library for the six applications that use it. Virtualenv as a go-to solution also excuses library developers from versioning their product properly.

Pip introduces a slightly different problem: Multiple package managers on the system that don't play nice together. Pip doesn't know what APT has installed and APT doesn't know what pip has installed, so dependency violations involving one package from each of them aren't caught.

This seems like a Bad Thing, but I'm not sure what to do about it.

As a systems engineer, I look askance any time I have to install something that isn't available from an apt repo. As a programmer, I have to admit that I don't want my users to have to wait on downstream packagers for every update, and I don't want to be concerned with the system packaging differences across platforms. To me a large part of the point of using python is not having to care what OS my program is running on!

But as a user, I'm convinced that's the lazy response. In my ideal world, packaging would be handled by the team writing the program, and they'd provide (at least) an APT and yum repo alongside the customary tarball, ready for integration into the target platform's system. Pip-based packages shouldn't be the first and only thing you provide; they should be the *fallback*, the thing users have to use if they're on an unsupported platform.

This doesn't seem like a huge request. Most python devs already provide pip packages (which aren't always trivial to do right) and often a windows installer as well. Covering the most popular linux distributions (especially when your target platforms *are exclusively linux*) is only finishing the job.





[^1]: Don't even get me started on requirements.txt; yes, it has its uses, but way too often I see it used as a substitute for setup.py. It's an end-user file for reproducable deployments, *not* a way to specify your dependencies!
