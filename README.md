# ansible-services-example
Ansible design pattern showing how to conveniently handle many services/roles
of which different hosts can include different services. The pattern supports
the basic Ansible concepts of --limit and --tag, i.e at command line you can
limit the set of hosts that the playbook runs against, and you can limit which
services are deployed using the --tag argument.

## An Example
Imagine that you have two hosts managing your home network, host1 and host2.
You also have defined a series of roles; nginx, grafana, grafana-agent and
home-assistant.
You want both host to include nginx and grafana-agent, but only host1 should
include grafana and host2 should include home-assistant. Then this Ansible
pattern allows you to set the set of services in each host vars. I.e. your
inventory would look like this:

```
host1 services="nginx,grafana-agent,grafana"
host2 services="nginx,grafana-agent,home-assistant"
```

That is all, no need to add one role entry in the playbook for each service
where you repeat the role name twice, once for the role name and once as tag so
that you can run the playbook for only that role. You also have to repeat the
hosts multiple times in your playbook, once for every service it should have,
or, alternatively, have complicated host groups.
This pattern also has the added benefit of making it dead simple to see which
services/roles a specific host implements.

You can test this by running:
```
ansible-playbook -i inventory provision.yml
```
This will output:
```
PLAY [all] ********************************************************************************

TASK [Gathering Facts] ********************************************************************
ok: [host2]
ok: [host1]

TASK [Run service roles] ******************************************************************
skipping: [host1] => (item=home-assistant)
skipping: [host2] => (item=grafana)
included: nginx for host1, host2 => (item=nginx)
included: grafana-agent for host1, host2 => (item=grafana-agent)
included: grafana for host1 => (item=grafana)
included: home-assistant for host2 => (item=home-assistant)

TASK [nginx : Replace with real tasks...] *************************************************
ok: [host1] => {
    "msg": "Tasks for setting up nginx here"
}
ok: [host2] => {
    "msg": "Tasks for setting up nginx here"
}

TASK [grafana-agent : Replace with real tasks...] *****************************************
ok: [host1] => {
    "msg": "Tasks for setting up grafana-agent here"
}
ok: [host2] => {
    "msg": "Tasks for setting up grafana-agent here"
}

TASK [grafana : Replace with real tasks...] ***********************************************
changed: [host1] => {
    "msg": "Tasks for setting up grafana here"
}

TASK [home-assistant : Replace with real tasks...] ****************************************
changed: [host2] => {
    "msg": "Tasks for setting up nginx here"
}

RUNNING HANDLER [Reload nginx config] *****************************************************
ok: [host1] => {
    "msg": "Restarting nginx config, well not really..."
}
ok: [host2] => {
    "msg": "Restarting nginx config, well not really..."
}

PLAY RECAP ********************************************************************************
host1                      : ok=8    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
host2                      : ok=8    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

Or maybe limit the set of hosts:
```
$ ansible-playbook -i inventory provision.yml --limit host1

PLAY [all] ********************************************************************************

TASK [Gathering Facts] ********************************************************************
ok: [host1]

TASK [Run service roles] ******************************************************************
skipping: [host1] => (item=home-assistant)
included: nginx for host1 => (item=nginx)
included: grafana-agent for host1 => (item=grafana-agent)
included: grafana for host1 => (item=grafana)

TASK [nginx : Replace with real tasks...] *************************************************
ok: [host1] => {
    "msg": "Tasks for setting up nginx here"
}

TASK [grafana-agent : Replace with real tasks...] *****************************************
ok: [host1] => {
    "msg": "Tasks for setting up grafana-agent here"
}

TASK [grafana : Replace with real tasks...] ***********************************************
changed: [host1] => {
    "msg": "Tasks for setting up grafana here"
}

RUNNING HANDLER [Reload nginx config] *****************************************************
ok: [host1] => {
    "msg": "Restarting nginx config, well not really..."
}

PLAY RECAP ********************************************************************************
host1                      : ok=8    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```
... or limit which services/roles are deployed using tags:
```
$ ansible-playbook -i inventory provision.yml --tag nginx,grafana

PLAY [all] ********************************************************************************

TASK [Gathering Facts] ********************************************************************
ok: [host1]
ok: [host2]

TASK [Run service roles] ******************************************************************
skipping: [host1] => (item=grafana-agent)
skipping: [host1] => (item=home-assistant)
skipping: [host2] => (item=grafana-agent)
skipping: [host2] => (item=grafana)
skipping: [host2] => (item=home-assistant)
included: nginx for host1, host2 => (item=nginx)
included: grafana for host1 => (item=grafana)

TASK [nginx : Replace with real tasks...] *************************************************
ok: [host1] => {
    "msg": "Tasks for setting up nginx here"
}
ok: [host2] => {
    "msg": "Tasks for setting up nginx here"
}

TASK [grafana : Replace with real tasks...] ***********************************************
changed: [host1] => {
    "msg": "Tasks for setting up grafana here"
}

RUNNING HANDLER [Reload nginx config] *****************************************************
ok: [host1] => {
    "msg": "Restarting nginx config, well not really..."
}

PLAY RECAP ********************************************************************************
host1                      : ok=6    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
host2                      : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```
## Future Improvements:
* Check that verifies that a hosts list of services is among the possible
  services.
* Dependencies for services. E.g. B depends on A, and only B is specified as A
  hosts services, include A automatically.
