---
author: Romain "Belaran" Pelisse
date: October 23th, 2024 (Ansible Anwender)
paging: Slide %d / %d
---

## An Ansible Event Driven labs, that goes beyond Helloword

The proposal for this lab:
1. is to explore together the feature of Event Driven Ansible,
2. using a more real life case scenario,
3. in order to go deeper than the average tutorial.




Powered by [Slides](https://github.com/maaslalani/slides/).

---

## Who am I?

- Romain "Belaran" Pelisse
    - a Java developer who loves RHEL and Ansible!
    - Open Source enthousiasts since 2004
    - A proud Red Hatters for 13 years
    - Into automation since 2011 (Puppet and then Ansible)
    - Authors for GNU/Linux Magazine France for almost twenty years
        - If you know what *Shadowrun* is, you might also know me as author...
- What do I **do** for Red Hat?
    - [Ansible Middleware Initiative](https://github.com/ansible-middleware/)
    - Integration between Red Hat Runtimes solution and Ansible (collections)
        - Red Hat JBoss Entreprise Application Platform (upstream Wildfly)
        - Red Hat JBoss Web Server (upstream aka as Tomcat)
        - Red Hat Single Sign On (upstream is Keycloak)
        - Red Hat AMQ Streams (upstream is Kafka)
        - Red Hat AMQ Broker (upstream Apache MQ)

---

## Our use case

- Use EDA to react to a reccuring incident
- We have a Java app hosted by Wildfly
    - so not Nginx, a real, legacy like application
- Prod often runs into
    - sometimes the server or the app becomes unavailable
    - random Java Out of Memory Error, needs to restart the app server

---

## Installing Wildfly using Ansible

Thanks to our collection for Wildfly (and JBoss EAP), it's super easy to install Wildfly!

```bash
$ ansible-galaxy collection install middleware_automation.wildfly
```

Then we can simply use the provided playbook:

```bash
$ ansible-playbook -i inventory middleware_automation.wildfly.playbook
```

In this lab, we'll use the some machine as both **controller** and **target**:

```ini
[all]
localhost ansible_connection=local
```
Let's check:
```bash
# systemctl status wildlfy
```

---

## Event Driven Ansible

First:
- Don't ask me about EDA architecture, I don't know anything about it!
- Well, no, ironically, I know it's using a Java project called Drools

Now, let's install EDA:

```bash
$ ansible-galaxy collection install ansible.eda middleware_automation.wildfly
```

To run EDA, we'll use the following command:
```bash
$ ansible-rulebook -i inventory -r rulebook1.yml --verbose
```

However, before doing so, we'll need a **rulebook1.yml**...

---

## Our first rulebook...

Let's start at the beginning, with a minimal rulebook:

```yaml
- name: EDA lab
  hosts: all
  sources:
    - ansible.eda.url_check:
        urls:
          - http://localhost:8080/
        delay: 10
  rules:
    - name: Restart Wildfly if url is not reachable
      condition: event.url_check.status == "down"
      action:
        run_playbook:
          name: restart_wildfly.yml

```

---

## Let's take a look at playbook

Here is the content of the playbook being triggered above:

```yaml
- name: Ensure Wildfly is install and running as a service
  hosts: all
  collections:
    - middleware_automation.wildfly
  tasks:
    - name: Restart Wildfly
      ansible.builtin.service:
        name: wildfly
        state: restarted
```

---

## Enhancing the healthcheck

Wildfly is an application server. While it may still be running, the hosted app may be unreachable. Let's enhance our rulebook accordingly.

First, let's deploy [a sample webapp](https://github.com/rpelisse/eda-app/releases/download/eda-app-v1/edapp-1.0.war)

```
$ /opt/wildfly/wildfly-34.0.0.Final/bin/jboss-cli.sh --connect --command='deploy info-1.2.war'
```
Now, let's enhance the rulebook to trigger a restart if the app is down:

```yaml
...
  sources:
    - ansible.eda.url_check:
        urls:
          - http://localhost:8080/
          - http://localhost:8080/info-1.2
...
```

---

## Diving deeper: file_watch

Let's tackle a more difficult requirement: restart the server is case of Out of Memory error. When this kind of error happens, it will be added to the server main logfile, so we

Looking at the available [`sources`](https://ansible.readthedocs.io/projects/rulebook/en/stable/sources.html), it seems like file_watch could be a good match.

Let's try to implement this requirement with file_watch...

---
## file_watch in practise

First, let's use file_watch to monitor the logfile
```yaml
 - ansible.eda.file_watch:
        path: /opt/wildfly/wildfly-33.0.2.Final/standalone/log/server.log
        recursive: false

  rules:

    - name: Restart Wildfly if OOM is detected
      condition: event.file_watch.change == "modified"
      action:
        run_playbook:
          name: oom.yml
          extra_vars:
            triggeredBy: "{{ event }}"
...
```
For now, the oom.yml is simply going to print the value of the variable triggeredBy.

To trigger an Out of Memory error, just access this [URL](http://localhost:8080/edapp-1.0/tutorial/app/hello)

---
## Why did we failed?

Simply put, file_watch is designed to operate and react on file level, rather than a file content. A file creation or deletion, but it can't react on a precise content change.

On top of this, the event provided does contained the added content to the logfile, which we would need to be able to detect the apparition of an out of memory error.

So... How do can we do it?

---
## journald

While, it's not documented yet (as the time of this presentation), there is a [journald source](https://github.com/ansible/event-driven-ansible/blob/main/extensions/eda/plugins/event_source/journald.py) available. As the [middleware_automation.wildfly collection](https://github.com/ansible-middleware/wildfly) integrates Wildfly into systemd, it also means the main logfile of the app server is echoed into journald, so we can try to leverage this source indeed.

However, as we lack documentation, we need first to have an idea of the structure of the event being returned by this source, so let's just catch and print any of them:

```
  sources:
      - ansible.eda.journald:
              match: ALL
  rules:
      - name: Match all on journald
        condition: True
        action:
          print_event:
            pretty: true
```

---
## Detecting OOM using conditions

As you can see the structure returns is immensily rich! We have plenty of informations available to us. The one we need is the message itself, which we can access using the **journald.message** attribute of the structure. Now we need to have a condition detecting when the event signals the apparition of a Java Out of Memory error.

To do so, we need first to refers to the available documentation on [conditions](https://ansible.readthedocs.io/projects/rulebook/en/stable/conditions.html).

From there, we can see there is a *is search* operator that will allow us to check for OOM:

```
  rules:
      - name: Detect Out of Memory erros in journald
        condition: event.journald.message is search("java.lang.OutOfMemoryError", ignorecase=false)
        action:
          run_playbook:
            name: oom.yml
            extra_vars:
              triggeredBy: "{{ event }}"
...

And voilà!

We know can trigger a playbook in case a Out of Memory is detected in the server!
