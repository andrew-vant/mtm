% Salt defaults.get

(*prerequisites: basic knowledge of salt, jinja, and yaml*)

[defaults.get][dget] is a wonderful Salt feature with less-than-optimal documentation.[^1]

Let's suppose you have a salt state that configures `some_application`. And let's suppose you want the location and source for the configuration file template to be settable in pillar. If you do it the usual way, the relevant part of your salt state and pillar might look something like this:

```salt
# salt/someapp/configure.sls
some_application_conf:
  file.managed:
    - name: {{ salt['pillar.get']("someapp:conffile", "/etc/someapp.conf") }}
    - source: {{ salt['pillar.get']("someapp:source", "salt://someapp/files/someapp.conf") }}
    - template: jinja
    ...

# pillar/someapp.sls
someapp:
  name: /opt/someapp/etc/someapp.conf
  source: salt://some/other/location/someapp.conf
```

And it will work. You'll get the pillar-specified value if it is set, and a sane default if it is not set. Nevertheless there are two problems with this, neither of which stop it from working, and both of which only apply as your state template grows:

1. You have your default values mixed in with code.
2. The pillarized lines are really long and not intuitive to follow.

If you have a ten-line state, neither of these is a problem. If you have a two-hundred-line .sls with lots of configurable options, it's easy to lose track of what's going on. Using defaults.get in place of pillar.get lets you keep your default values in a separate file within your formula; either defaults.json or defaults.yaml. 

Unfortunately the example from the [official documentation][dget] used json. It's contrary to the yaml used just about everywhere else in the documentation. That makes it harder to pick out exactly what the relationship is between the defaults file and the pillar. Presenting it as defaults.yaml makes things much easier to see:

```salt
# salt/someapp/configure.sls
some_application_conf:
  file.managed:
    - name: {{ salt['defaults.get']("conffile") }}
    - source: {{ salt['defaults.get']("source") }}
    - template: jinja
    ...

# salt/someapp/defaults.yaml
conffile: /etc/someapp.conf
source: salt://someapp/files/someapp.conf

# pillar/someapp.sls
someapp:
  conffile: /opt/someapp/etc/someapp.conf
  source: salt://some/other/location/someapp.conf
```

`defaults.yaml` just provides the default contents of the `someapp` dictionary in the pillar. defaults.get assumes the value it's looking for will be in a dictionary of the same name as its formula. The code part is much easier to read, and the values are much easier to find and work with. They are also all in one place. It's true that using this restricts your pillar structure somewhat, in that once you've committed to using defaults.get, your formula must have a pillar structure consisting of a single dictionary named after it. In my opinion that is best practice anyway, so I don't think it's an unreasonable restriction.

But wait! We can make things even more readable, by aliasing defaults.get[^2]:

```salt
# salt/someapp/configure.sls

{%- set dget = salt['defaults.get'] %}

some_application_conf:
  file.managed:
    - name: {{ dget("conffile") }}
    - source: {{ dget("source") }}
    - template: jinja
    ...
```

The original messy lines have collapsed to around a third of their original size. It is blindly obvious what they are doing (as long as you remember what `dget` refers to), and they no longer distract the eye from the surrounding lines. Multiply this by every pillarized option in a long .sls file, and the file as a whole becomes much, much easier to read and maintain.

**Known Bug: defaults.get in Templates**

What about configuration file templates? Can you use defaults.get there? Well, yes...but there's a bug if you try to do it the obvious way. This bug only occurs during highstate, but that's what you should be using most of the time anyway.

```salt
# salt/someapp/configure.sls
{%- set dget = salt['defaults.get'] %}
some_application_conf:
  file.managed:
    - name: {{ dget("conffile") }}
    - source: {{ dget("conffile") }}
    - template: jinja

# salt/someapp/defaults.yaml
conffile: /etc/someapp.conf
source: salt://someapp/files/someapp.conf.jinja
conf:
  opt1: 1
  opt2: 2
  opt3: 3

# salt/someapp/files/someapp.conf.jinja
{%- set dget = salt['defaults.get'] %}
opt1: something
opt2: something_else
opt3: {{ dget("conf:opt3") }}  # OOPS: returns empty string instead of 3
```

I filed a [bug report](https://github.com/saltstack/salt/issues/19856) for this, but right now it's still open. It seems to occur because for some reason, during highstate, the template doesn't know what sls is applying it; and so doesn't know where to look for defaults.yaml.

You can work around this by explicitly specifying the sls in the state:

```salt
# salt/someapp/configure.sls
{%- set dget = salt['defaults.get'] %}
some_application_conf:
  file.managed:
    - name: {{ dget("conffile") }}
    - source: {{ dget("source") }}
    - template: jinja
    - context:
        '__sls__': someapp.configure
```

`__sls__` is an internal variable, and I'm not sure exactly how it works, but I'm glad it's possible to override it here.

[^1]: At time of writing. I imagine someone will update it eventually. Maybe even me.

[^2]: This trick works just as well for pillar.get, or any of the other salt['something'] functions.

[dget]: http://docs.saltstack.com/en/latest/ref/modules/all/salt.modules.defaults.html#salt.modules.defaults.get
