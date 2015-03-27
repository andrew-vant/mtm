% Pillar Jinja Magic

*(prerequisite: Previous article on peer discovery)*

Problem: You have five datacenters. In each datacenter, you are running five webheads and two database servers. You want to configure each webhead to talk to the database servers only in its own datacenter. Your server naming scheme looks like this:[^1]

```
host.cluster.datacenter.example.com # (generic)
node01.web.dc1.example.com          # (a webhead in the first datacenter)
node01.db.dc2.example.com           # (a database server in the second datacenter)
```

You're doing peer discovery in the manner of the previous article, so the pillar for your configuration file looks like this, at least to begin with:

```
# pillar/webapp.sls
webapp:
  db_server_glob: *.db.*.example.com
```

Oops. This will result in your webheads trying to contact DBs in other datacenters, which will be dog-slow at best and return the wrong data at worst. Let's try that again:

```
# pillar/webapp.sls
webapp:
  db_server_glob:
    dc1: *.db.dc1.example.com
    dc2: *.db.dc2.example.com
    # ...and so on
```

Oops again. Now your formula needs to be know what datacenter your server is in, in order to get the right entry from webapp:db_server_glob. It can get that information from the minion's ID or possibly from grains, but those are implementation details that may differ from one project to another. When possible we want that information in the pillar, not the state. 

Again:

```
# pillar/webapp/dc1.sls
webapp:
  db_server_glob: *.db.dc1.example.com

# pillar/webapp/dc2.sls
webapp:
  db_server_glob: *.db.dc2.example.com

# ...and so on.
```

This gets the job done, technically, but it sucks to manage. It clutters your pillar tree, and potentially makes a mess of the top file as well. I've tried all three of these and found them all unsatisfactory.

This is the solution I eventually settled on:

```
# pillar/webapp.sls
{%- set datacenter = grains['id'].split(".")[2] %}
webapp:
  db_server_glob: *.db.{{ datacenter }}.example.com
```

You can do that in pillar? I didn't know you could do that in pillar.

Yes, at least some grains are accessible during pillar rendering. I'm not sure which ones. Maybe all of them. The minion's ID always works, or always has for me. The bit of jinja in this example pulls the datacenter-element of the minion's ID, and inserts it into the glob used to search for database servers. For the use-case above, this lets you use the same pillar file to point your webheads to different sets of database servers. All the project-specific information remains in the pillar, not the formula.

I expect this technique to be useful in lots of places where a set of servers differ only slightly, and those differences match up with differences in their minion IDs.

mine.get, sadly, is not available during pillar rendering as far as I can tell. So you can't use this trick with Mine. If you want to use mine-based peer discovery, it has to be written into the formula to begin with. A lot of saltstack formulas don't support that, which is a shame.

[^1]: I use this scheme when I have the option, but any naming scheme will work as long as it's consistent and easy to parse.