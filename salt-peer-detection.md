% Peer Discovery Using Salt Mine

You have an environment managed by Salt. Let's say the environment consists of a set of webheads serving applications, which are backed by a cluster of database servers. The webheads need to know the IP addresses of the database servers in order to query them. The database servers need to know the IP addresses of the webheads, so that they can allow them to connect and block everything else.

Let's also say that both clusters are made up of cloud servers. You won't know what IP addresses they're going to get until after they have been built. So, you can't specify an explicit list of your servers' addresses in a pillar. Alternately, you just despise explicit lists (greetings, comrade!) and want a better way.

Your problem is this: How can each server get the addresses it needs to know, without having an explicit list defined anywhere?

Enter Salt Mine. Salt Mine is a mechanism for minions to get information about each other. Mine's official documentation is quite good. [Go read it][mine].

First you want minions' network interfaces exposed to the mine. You can do this from either the minion configuration file or the pillar, but it's preferable to do it from the pillar, because applying it only requires a pillar refresh rather than a minion service restart. 

```
# pillar/salt/mine.sls
mine_functions:
  network.interfaces: []
  test.ping: []
  # and anything else you want available....
```

After adding this, refresh the pillar (or just run a highstate). That makes the network interface information for your minion available in the salt mine. For purposes of this example we are looking for the first ip address on the interface 'eth0'. We're also assuming that the database servers are named to match the glob \*.database.\*. For example, node01.database.example.com:

```
# salt/someapp/config.sls

{%- set database_ips = [] %}
{%- for minion in salt['mine.get']('*.database.example.com', 'network.interfaces').values() %}
{%-   do database_ips.append(minion['eth0']['inet'][0]['address']) %}
{%- endfor %}

# Now database_ips is an array containing the ip addresses of all
# minions matching the *.database.example.com glob, and we can use
# it in the state.

someapp-conf:
  file.managed:
    - name: /etc/someapp.conf
    - source: salt://someapp/files/someapp.conf
    - template: jinja
    - context:
        dbservers: {{ database_ips|yaml }} # Not sure if the |yaml is needed here...
```

Your config file template now receives an array, containing the ip addresses of the database servers in a context variable named 'dbservers'. The same technique can be used on the database side to get the addresses of the webheads; or, for that matter, to allow the database servers to discover each other. Of course, the query could be performed inside the template instead, if desired.

There is still a flaw in the above code. It assumes a particular naming scheme for the database servers. Deployment-specific information like that shouldn't go in a salt formula. It should be in the pillar. 

Fortunately, we can pull the mine query itself from the pillar!

```
# pillar/someapp.sls
someapp:
  database_minions: "*.database.example.com"

# salt/someapp/config.sls
# Using some aliases for clarity.
{%- set pget = salt['pillar.get'] %}
{%- set mget = salt['mine.get'] %}

{%- set database_ips = [] %}
{%- set query = pget("someapp:database_minions") %}

{%- for minion in mget(query, 'network.interfaces').values() %}
{%-   do database_ips.append(minion['eth0']['inet'][0]['address']) %}
{%- endfor %}

# And the rest of the state as before...
```

One appealing element of this solution is that, if we add servers to a cluster or change their addresses, the changes can be picked up without altering any code. Just give them the same naming scheme as the existing cluster, update the mine, and run a highstate:

```
salt '*' mine.update
salt '*' state.highstate
```

It gets more complicated if you have interdependencies that have to be set up in a particular order, but for a lot of use cases it really is just that simple.

[mine]: http://docs.saltstack.com/en/latest/topics/mine/
