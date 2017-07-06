## ansible 
```
introduction 
playbooks:A Begining 
Inventory:Describing Your Server
Variables and Facts
```

#### Introduction 
`
We now deploy software applications by stringing together services that run on a distributed set of computing resources and communicate over different networking pro‐ tocols. A typical application can include web servers, application servers, memory- based caching systems, task queues, message queues, SQL databases, NoSQL datastores, and load balancers.   `     

`We also need to make sure we have the appropriate redundancies in place, so that when failures happen (and they will), our software systems will handle these failures gracefully. Then there are the secondary services that we also need to deploy and maintain, such as logging, monitoring, and analytics, as well as third-party services we need to interact with, such as infrastructure-as-a-service endpoints for managing virtual machine instances.    `     

`Ansible is a great tool for doing deploy‐ ment as well as configuration management. Using a single tool for both configuration management and deployment makes life simpler for the folks responsible for opera‐ tions.    `

*问：如果要部署大量的机器, 同时会有大量的下载，fileserver 会跑满，有什么方法？*    
*答：使用集群，部署多个入口，使用 dns 轮训。*
```
ansible 并发执行规则：
- Ansible runs each task in parallel across all hosts.
- Ansible waits until all hosts have completed a task before moving to the next task.
- Ansible runs the tasks in the order that you specify them.

ansile 的优点：
- Easy-to-Read Syntax
- Nothing to Install on Remotes Hosts
- Push-Based
- Ansible Scales Down 
- Built-in Modules
- Very Thin Layer of Abstraction
```

#### Installing Ansible 
```
pip     
yum    
...
```
*好的习惯:  inventory files 使用版本控制*

- ping modules      
`-m ping `    
- command modules  [default]
```
-m command -a flag   
if our command contains spaces, we need to quote it so that the shell passes the entire string as a single argument to Ansible. 
```
-  -s to use sudo

####  Playbooks
- playbook/web-nginx.yaml
```
- name: Configure webserver with nginx
  hosts: webservers
  sudo: True
  tasks:
    - name: install nginx
      apt: name=nginx update_cache=yes
    - name: copy nginx config file
      copy: src=files/nginx.conf dest=/etc/nginx/sites-available/default
    - name: enable configuration
      file: >
        dest=/etc/nginx/sites-enabled/default
        src=/etc/nginx/sites-available/default
        state=link
    - name: copy index.html
      template: src=templates/index.html.j2 dest=/usr/share/nginx/html/index.html
mode=0644
    - name: restart nginx
      service: name=nginx state=restarted
```
```
Ansible is pretty flexible on how you represent truthy and falsey values in playbooks. Strictly speaking, module arguments (like update_cache=yes) are treated differently from values elsewhere in playbooks (like sudo: True). 
YAML truthy:       true, True, TRUE, yes, Yes, YES, on, On, ON, y, Y 
YAML falsey:       false, False, FALSE, no, No, NO, off, Off, OFF, n, N
module arg truthy: yes, on, 1, true
module arg falsey: no, off, 0, false
```
- playbook/file/nginx.conf
```
server {
    listen 80 default_server;
    listen [::]:80 default_server ipv6only=on;
    server_name localhost;

    root /usr/share/nginx/html;
    index index.html index.htm;

    location / {
        try_files $uri $uri/ =404;
    }
}
```
-  playbooks/templates/index.html.j2
```
<html>
  <head>
<title>Welcome to ansible</title> </head>
<body>
<h1>nginx, configured by Ansible</h1>
<p>If you can see this, Ansible successfully installed nginx.</p>
<p>{{ ansible_managed }}</p>
  </body>
</html>
```

- playbook/hosts
```
[webservers]
testserver ansible_ssh_host=127.0.0.1 ansible_ssh_port=2222
```

- run 
```
$ ansible webservers -m ping
$ ansible-playbook web-notls.yml
if your playbook file is marked as executable and starts with a line that looks like this:
    #!/usr/bin/env ansible-playbook
then you can execute it by invoking it directly, like this:
    $ ./web-notls.yml
```
*关注 Gathering Facts 的所以信息*    
`A valid JSON file is also a valid YAML file. This is because YAML allows strings to be quoted, considers true and false to be valid Booleans, and has inline lists and dictionary syntaxes that are the same as JSON arrays and objects. But don’t write your playbooks as JSON—the whole point of YAML is that it’s easier for people to read`


#### Modules 
- apt `Installs or removes packages using the apt package manager`
- copy `Copies a file from local machine to the hosts.`
- file `Sets the attribute of a file, symlink, or directory.`
- service `Sets the attribute of a file, symlink, or directory.`
- template `Generates a file from a template and copies it to the hosts.`


#### Variables
- web-nginx-tsl.yaml
```
- name: Configure webserver with nginx and tls
  hosts: webservers
  sudo: True
  vars:
    key_file: /etc/nginx/ssl/nginx.key
    cert_file: /etc/nginx/ssl/nginx.crt
    conf_file: /etc/nginx/sites-available/default
    server_name: localhost
tasks:
    - name: Install nginx
      apt: name=nginx update_cache=yes cache_valid_time=3600
    - name: create directories for ssl certificates
      file: path=/etc/nginx/ssl state=directory
    - name: copy TLS key
      copy: src=files/nginx.key dest={{ key_file }} owner=root mode=0600
      notify: restart nginx
    - name: copy TLS certificate
      copy: src=files/nginx.crt dest={{ cert_file }}
      notify: restart nginx
    - name: copy nginx config file
      template: src=templates/nginx.conf.j2 dest={{ conf_file }}
      notify: restart nginx
    - name: enable configuration
      file: dest=/etc/nginx/sites-enabled/default src={{ conf_file }} state=link
      notify: restart nginx
    - name: copy index.html
      template: src=templates/index.html.j2 dest=/usr/share/nginx/html/index.html
mode=0644
  handlers:
    - name: restart nginx
      service: name=nginx state=restarted
```
- teplates/nginx.conf.j2
```
server {
    listen 80 default_server;
    listen [::]:80 default_server ipv6only=on;
    
    listen 443 ssl;
    
    root /usr/share/nginx/html; 
    ndex index.html index.htm;

    server_name {{ server_name }};

    ssl_certificate {{ cert_file }};
    ssl_certificate_key {{ key_file }};

    location / {
    try_files $uri $uri/ =404;
    } 
}
```

#### Handlers
`Handlers are one of the conditional forms that Ansible supports. A handler is similar to a task, but it runs only if it has been notified by a task.`
- Handlers only run after all of the tasks are run, and they only run once, even if they are notified multiple times. 
```
此处有坑：
1. I run a playbook.
2. One of my tasks with a notify on it changes state.
3. An error occurs on a subsequent task, stopping Ansible.
4. I fix the error in my playbook.
5. I run Ansible again.
6. None of the tasks report a state change the second time around, so Ansible doesn’t run the handler.
```

### Inventory: Describing Your Servers
- inventory: `The collection of hosts that Ansible knows about `
- inventory file :`The default way to describe your hosts in Ansible is to list them in text files`
```
Behavioral inventory parameters:
ansible_ssh_port 
ansible_ssh_port 
ansible_ssh_user
ansible_ssh_pass
ansible_connection
ansible_ssh_private_key_file
ansible_shell_type
ansible_python_interpreter
ansible_*_interpreter
```

#### Hosts and Group Variables: Inside the Inventory
- This variable can then be used in a playbook, just like any other variable.    
`Personally, I don’t often attach variables to specific hosts. On the other hand, I often associate variables with groups.`
- group variables are organized into sections named \[\<group name\>:vars\].
```
# ansible.cfg
[all:vars]
ntp_server=ntp.ubuntu.com
[production:vars]
db_primary_host=rhodeisland.example.com
db_primary_port=5432
db_replica_host=virginia.example.com
db_name=widget_production
db_user=widgetuser
db_password=pFmMxcyD;Fc6)6
rabbitmq_host=pennsylvania.example.com
rabbitmq_port=5672
....
```

#### Host and Group Variables: In Their Own Files
`Ansible looks for host variable files in a directory called host_vars and group variable files in a directory called group_vars. Ansible expects these directories to be either in the directory that contains your playbooks or in the directory adjacent to your inven‐ tory file. In our case, those two directories are the same.`
```
db_primary_host: rhodeisland.example.com
db_replica_host: virginia.example.com
db_name: widget_production
db_user: widgetuser
db_password: pFmMxcyD;Fc6)6
rabbitmq_host:pennsylvania.example.com
```
```
# playbooks/group_vars/production 
db:
    user: widgetuser
    password: pFmMxcyD;Fc6)6
    name: widget_production
    primary:
        host: rhodeisland.example.com
        port: 5432
    replica:
        host: virginia.example.com
        port: 5432
rabbitmq:
    host: pennsylvania.example.com
    port: 5672
```
```
if we choose YAML dictionaries, that changes the way we access the variables:    
    {{ db_primary_host }}    
versus:    
    {{ db.primary.host }}
```

#### Dynamic Inventory
- The Interface for a Dynamic Inventory Script
```
If the inventory file is marked executable, Ansible will assume it is a dynamic inventory script and will execute the file instead of reading it.
To mark a file as executable, use the chmod +x command.
An Ansible dynamic inventory script must support two command-line flags:
  --host=<hostname> for showing host details
  --list for listing groups
```
```
showing host details
To get the details of the individual host, Ansible will call the inventory script like this:
$ ./dynamic.py --host=vagrant2
The output should contain any host-specific variables, including behavioral parame‐
ters, like this:
   { "ansible_ssh_host": "127.0.0.1", "ansible_ssh_port": 2200, "ansible_ssh_user": "vagrant"}
The output is a single JSON object where the names are variable names, and the values are the variable values.
```
```
Listing groups
Dynamic inventory scripts need to be able to list all of the groups, and details about the individual hosts. For example, if our script is called dynamic.py, Ansible will call it like this to get a list of all of the groups:
    $ ./dynamic.py --list
The output should look something like this:
{
  "production": [
    "delaware.example.com",
    "georgia.example.com",
    "maryland.example.com",
    "newhampshire.example.com",
    "newjersey.example.com",
    "newyork.example.com",
    "northcarolina.example.com",
    "pennsylvania.example.com",
    "rhodeisland.example.com",
    "virginia.example.com"
  ],
  "staging": [
    "ontario.example.com",
    "quebec.example.com"
  ],
  "vagrant": [
    "vagrant1",
    "vagrant2",
    "vagrant3"
  ],
  "lb": [
    "delaware.example.com"
  ],
  "web": [
    "georgia.example.com",
    "newhampshire.example.com",
    "newjersey.example.com",
    "ontario.example.com",
    "vagrant1"
  ],
  "task": [
    "newyork.example.com",
    "northcarolina.example.com",
    "ontario.example.com",
    "vagrant2"
  ],
  "rabbitmq": [
    "pennsylvania.example.com",
    "quebec.example.com",
    "vagrant3"
  ],
  "db": [
    "rhodeisland.example.com",
    "virginia.example.com",
    "vagrant3"
  ]
}

The output is a single JSON object where the names are Ansible group names, and the values are arrays of host names.
As an optimization, the --list command can contain the values of the host variables for all of the hosts, which saves Ansible the trouble of making a separate --host invo‐ cation to retrieve the variables for the individual hosts.
To take advantage of this optimization, the --list command should return a key named _meta that contains the variables for each host, in this form:
"_meta" :
{ "hostvars" :
"vagrant1" : { "ansible_ssh_host": "127.0.0.1", "ansible_ssh_port": 2222, "ansible_ssh_user": "vagrant"},
"vagrant2": { "ansible_ssh_host": "127.0.0.1", "ansible_ssh_port": 2200, "ansible_ssh_user": "vagrant"},
...
}
```

#### Writing a Dynamic Inventory Script
```
Example 3-12. vagrant.py
#!/usr/bin/env python
# Adapted from Mark Mandel's implementation
# https://github.com/ansible/ansible/blob/devel/plugins/inventory/vagrant.py
# License: GNU General Public License, Version 3 <http://www.gnu.org/licenses/>

import argparse
import json
import paramiko
import subprocess
import sys

def parse_args():
    parser = argparse.ArgumentParser(description="Vagrant inventory script") 
    group = parser.add_mutually_exclusive_group(required=True) 
    group.add_argument('--list', action='store_true') 
    group.add_argument('--host')
    return parser.parse_args()

def list_running_hosts():
    cmd = "vagrant status --machine-readable"
    status = subprocess.check_output(cmd.split()).rstrip() 
    hosts = []
    for line in status.split('\n'):
        (_, host, key, value) = line.split(',') 
        if key == 'state' and value == 'running':
            hosts.append(host) 
    return hosts

def get_host_details(host):
    cmd = "vagrant ssh-config {}".format(host)
    p = subprocess.Popen(cmd.split(), stdout=subprocess.PIPE) 
    config = paramiko.SSHConfig()
    config.parse(p.stdout)
    c = config.lookup(host)
    return {'ansible_ssh_host': c['hostname'],
                'ansible_ssh_port': c['port'],
                'ansible_ssh_user': c['user'],
                'ansible_ssh_private_key_file': c['identityfile'][0]}
def main():
    args = parse_args() 
    if args.list:
        hosts = list_running_hosts()
        json.dump({'vagrant': hosts}, sys.stdout) else:
        details = get_host_details(args.host)
        json.dump(details, sys.stdout)

if __name__ == '__main__': main()
```

#### add_host
```
The add_host module adds a host to the inventory. This module is useful if you’re using Ansible to provision new virtual machine instances inside of an infrastructure- as-a-service cloud.
```
- Invoking the module looks like this:    
    `add_host name=hostname groups=web,staging myvar=myval`
```

    - name: Provision a vagrant machine
      hosts: localhost
      vars:
        box: trusty64
      tasks:
        - name: create a Vagrantfile
          command: vagrant init {{ box }} creates=Vagrantfile
        - name: Bring up a vagrant server
          command: vagrant up
        - name: add the Vagrant hosts to the inventory
          add_host: >
                name=vagrant
                ansible_ssh_host=127.0.0.1
                ansible_ssh_port=2222
                ansible_ssh_user=vagrant
                ansible_ssh_private_key_file=/Users/lorinhochstein/.vagrant.d/
                insecure_private_key
    - name: Do something to the vagrant machine
      hosts: vagrant
      sudo: yes
      tasks:
        # The list of tasks would go here
 ...

```

#### group_by
```
Ansible also allows you to create new groups during execution of a playbook, using the group_by module. This lets you create a group based on the value of a variable that has been set on each host, which Ansible refers to as a fact.
```
```
# Creating ad-hoc groups based on Linux distribution
- name: group hosts by distribution
  hosts: myhosts
  gather_facts: True
  tasks:
    - name: create groups based on distro
      group_by: key={{ ansible_distribution }}
- name: do something to Ubuntu hosts
  hosts: Ubuntu
  tasks:
    - name: install htop
      apt: name=htop
# ...
- name: do something else to CentOS hosts
  hosts: CentOS
  tasks:
    - name: install htop
      yum: name=htop
# ...
```

### Variables and Facts
`Ansible is not a full-fledged programming language, but it does have several pro‐ gramming language features, and one of the most important of these is variable sub‐ stitution. In this chapter, we'lll cover Ansible’s support for variables in more detail, including a certain type of variable that Ansible calls a fact.`
- Defining Variables in Playbooks
     - The simplest way to define variables is to put a vars section in your playbook with the names and values of variables. 
     ```
      vars:
      key_file: /etc/nginx/ssl/nginx.key
      cert_file: /etc/nginx/ssl/nginx.crt
      conf_file: /etc/nginx/sites-available/default
      server_name: localhost
     ```
     - Ansible also allows you to put variables into one or more files, using a section called vars_files.
     ```
      vars_files:
        - nginx.yaml
     ```
     ```
      # nginx.yaml
      key_file: /etc/nginx/ssl/nginx.key
      cert_file: /etc/nginx/ssl/nginx.crt
      conf_file: /etc/nginx/sites-available/default
      server_name: localhost
     ```

#### Viewing the Values of Variables
`- debug: var=myvarname`

#### Registering Variables
`Often, you’ll find that you need to set the value of a variable based on the result of a task. To do so, we create a registered variable using the register clause when invok‐ ing a module`
- Capturing the output of a command to a variable
```
# capture the output of the whoami command to a variable named login.
- name: capture output of whoami command
  command: whoami
  register: login
#In order to use the login variable later, we need to know what type of value to expect. The value of a variable set using the register clause is always a dictionary, but the spe‐ cific keys of the dictionary are different, depending on the module that was invoked.
```
```
# the simplest way to find out what a module returns is to register a variable and then output that variable with the debug module:
# Example 4-3. whoami.yml
- name: show return value of command module
  hosts: server1
  tasks:
    - name: capture output of id command
      command: id -un
      register: login
- debug: var=login

# The output of the debug module would look like this:
    TASK: [debug var=login] *******************************************************
    ok: [server1] => {
        "login": {
            "changed": true,
            "cmd": [
                "id",
                "-un"
            ],
            "delta": "0:00:00.002180",
            "end": "2015-01-11 15:57:19.193699",
            "invocation": {
                "module_args": "id -un",
                "module_name": "command"
            },
            "rc": 0,
            "start": "2015-01-11 15:57:19.191519",
            "stderr": "",
            "stdout": "vagrant",
            "stdout_lines": [
                "vagrant"
                ],
            "warnings": []
        }
}

```
#### Accessing Dictionary Keys in a Variable
- If a variable contains a dictionary, then you can access the keys of the dictionary using either a dot \(.\) or a subscript \(\[ \]\).

#### Facts
`When Ansible gathers facts, it connects to the host and queries the host for all kinds of details about the host: CPU architecture, operating system, IP addresses, memory info, disk info, and more. This information is stored in variables that are called facts, and they behave just like any other variable doe`
```
    - name: print out operating system
      hosts: all
      gather_facts: True
      tasks:
      - debug: var=ansible_distribution
```

#### Viewing All Facts Associated with a Server
- Ansible implements fact collecting through the use of a special module called the setup module     
`You don’t need to call this module in your playbooks because Ansible does that automatically when it gathers facts.`    
`manually : ansible server1 -m setup`

#### Viewing a Subset of Facts
- Because Ansible collects many facts, the setup module supports a filter parameter that lets you filter by fact name by specifying a glob.
`ansible web -m setup -a 'filter=ansible_eth*'`

#### Any Module Can Return Facts
`If a module returns a dictionary that contains ansible_facts as a key, then Ansible will create variable names in the environment with those values and associate them with the active host.For modules that return facts, there’s no need to register variables, since Ansible cre‐ ates these variables for you automatically. `

#### Local Facts
- Ansible also provides an additional mechanism for associating facts with a host. You can place one or more files on the host machine in the /etc/ansible/facts.d directory.
```
Ansible will recognize the file if it’s:
 In.iniformat
 In JSON format
 An executable that takes no arguments and outputs JSON on standard out
```
- These facts are available as keys of a special variable named ansible_local.
```
Example 4-9. /etc/ansible/facts.d/example.fact
[book]
title=Ansible: Up and Running
author=Lorin Hochstein
publisher=O'Reilly Media
If we copy this file to /etc/ansible/facts.d/example.fact on the remote host, we can access the contents of the ansible_local variable in a playbook:

    - name: print ansible_local
      debug: var=ansible_local
    - name: print book title
      debug: msg="The title of the book is {{ ansible_local.example.book.title }}"

The output of these tasks looks like this:
    TASK: [print ansible_local] ***************************************************
    ok: [server1] => {
        "ansible_local": {
            "example": {
                "book": {
                    "author": "Lorin Hochstein",
                    "publisher": "O'Reilly Media",
                    "title": "Ansible: Up and Running"
                }
            }
        }
    }
    TASK: [print book title] ******************************************************
    ok: [server1] => {
        "msg": "The title of the book is Ansible: Up and Running"
    }
```

#### Using set_fact to De ne a New Variable
- Ansible also allows you to set a fact (effectively the same as defining a new variable) in a task using the set_fact module.
-  often like to use set_fact immediately after register to make it simpler to refer to a variable. 
```
Using set_fact to simplify variable reference
- name: get snapshot id
  shell: >
    aws ec2 describe-snapshots --filters
    Name=tag:Name,Values=my-snapshot
    | jq --raw-output ".Snapshots[].SnapshotId"
  register: snap_result

- set_fact: snap={{ snap_result.stdout }}

- name: delete old snapshot
  command: aws ec2 delete-snapshot --snapshot-id "{{ snap }}"
```

#### Built-in Variables
```
hostvars            A dict whose keys are Ansible host names and values are dicts that map variable names to values
inventory_hostname  Name of the current host as known by Ansible
group_names         A list of all groups that the current host is a member of
groups              A dict whose keys are Ansible group names and values are a list of hostnames that are members of the group. Includes all and ungrouped groups: {"all": [...], "web": [...], "ungrou ped": [...]}
play_hosts          A list of inventory hostnames that are active in the current play
nsible_version      A dict with Ansible version info: {"full": 1.8.2", "major": 1, "minor": 8, "revision": 2, "string": "1.8.2"}
```
- hostvars variable. `This is a dictionary that contains all of the variables defined on all of the hosts, keyed by the hostname as known to Ansible.`    
```
If Ansible has not yet gathered facts on a host, then you will not be able to access its facts using the hostvars variable, unless fact caching is enabled.
Continuing our example, if our database server is db.example.com, then we could put the following in a configuration template:
    {{ hostvars['db.example.com'].ansible_eth1.ipv4.address }}
```

- inventory_hostname  `The inventory_hostname is the hostname of the current host, as known by Ansible.`
```
If you have defined an alias for a host, then this is the alias name. For example, if your inventory contains a line like this:
    server1 ansible_ssh_host=192.168.4.10
then the inventory_hostname would be server1.
You can output all of the variables associated with the current host with the help of the hostvars and inventory_hostname variables:
    - debug: var=hostvars[inventory_hostname]
```

- Groups  `The groups variable can be useful when you need to access variables for a group of hosts.`
```
we are configuring a load balancing host, and our configuration file needs the IP addresses of all of the servers in our web group. Our configuration file would contain a fragment that looks like this:
   backend web-backend
    {% for host in groups.web %}
      server {{ host.inventory_hostname }} {{ host.ansible_default_ipv4.address }}:80
    {% endfor %}
The generated file would look like this:
    backend web-backend
      server georgia.example.com 203.0.113.15:80
      server newhampshire.example.com 203.0.113.25:80
      server newjersey.example.com 203.0.113.38:80

```

#### Setting Variables on the Command Line
```
# greet.yml
- name: pass a message on the command line
  hosts: localhost
  vars:
    greeting: "you didn't specify a message"
  tasks:
    - name: output a message
      debug: msg="{{ greeting }}"
```
- Variables set by passing -e var=value to ansible-playbook have the highest prece‐ dence, which means you can use this to override variables that are already defined.
```
$ ansible-playbook greet.yml -e greeting=hiya
$ ansible-playbook greet.yml -e 'greeting="hi there"'
```
- Ansible also allows you to pass a file containing the variables instead of passing them directly on the command line by passing @filename.yml as the argument to -e
```
$ ansible-playbook greet.yml -e @greetvars.yml
```

#### Precedence
- The basic rules of precedence are:
    - (Highest) ansible-playbook -e var=value
    - Everything else not mentioned in this list
    - On a host or group, either defined in inventory file or YAML file
    - Facts
    - In defaults/main.yml of a role.


### Deploying Mezzanine with Ansible 
#### Listing Tasks in a Playbook
```
$ ansible-playbook --list-tasks mezzanine.yml
```
```
play #1 (Deploy mezzanine on vagrant):  
  install apt packages
  check out the repository on the host
    install required python packages
    install requirements.txt
    create a user
    create the database
    generate the settings file
    sync the database, apply migrations, collect static content
    set the site id
    set the admin password
    set the gunicorn config file
    set the supervisor config file
    set the nginx config file
    enable the nginx config file
    remove the default nginx config file
    ensure config path exists
    create tls certificates
    install poll twitter cron job
```

#### Using Iteration (with_items) to Install Multiple Packages
```
# Installing system packages
- name: install apt packages
  apt: pkg={{ item }} update_cache=yes cache_valid_time=3600
  sudo: True
  with_items:
    - git
    - libjpeg-dev
    - libpq-dev
    - memcached
    - nginx
    - postgresql
    - python-dev
    ...
# There’s a lot to unpack here. Because we’re installing multiple packages, we use Ansible’s iteration functionality, the with_items clause.
```
```
We could have installed the pack‐ ages one at a time, like this:
    - name: install git
      apt: pkg=git
    - name: install libjpeg-dev
      apt: pkg=libjpeg-dev
    ...
```

#### Checking Out the Project Using Git
```
# Checking out the Git repository
- name: check out the repository on the host
  git: repo={{ repo_url }} dest={{ proj_path }} accept_hostkey=yes
```

#### Installing Multiple Packages into a virtualenv
- Ansible’s pip module has support for installing packages into a virtualenv and for cre‐ ating the virtualenv if it is not available.
```
#  Install Python packages
- name: install required python packages
  pip: name={{ item }} virtualenv={{ venv_path }}
  with_items:
    - gunicorn
    - setproctitle
    - south
    - psycopg2
    - django-compressor
    - python-memcached
- name: install requirements.txt
  pip: requirements={{ proj_path }}/{{ reqs_path }} virtualenv={{ venv_path }}
```
```
$ source ~/mezzanine_example/bin/activate
$ pip freeze > requirements.txt

# installing from requirements.txt
- name: copy requirements.txt file
  copy: src=files/requirements.txt dest=~/requirements.txt
- name: install packages
  pip: requirements=~/requirements.txt virtualenv={{ venv_path }}
```
```
 Specifying package names and version
- name: python packages
  pip: name={{ item.name }} version={{ item.version }} virtualenv={{ venv_path }}
  with_items:
    - {name: mezzanine, version: 3.1.10 }
    - {name: gunicorn, version: 19.1.1 }
    - {name: setproctitle, version: 1.1.8 }
    - {name: south, version: 1.0.1 }
    - {name: psycopg2, version: 2.5.4 }
    - {name: django-compressor, version: 1.4 }
    - {name: python-memcached, version: 1.53 }
```

#### Creating the Database and Database User
- Ansible ships with the postgresql_user and postgresql_db modules for creating users and databases inside of Postgres.
- Ansible ships with a django_manage module that invokes manage.py commands.

#### Running Custom Python Scripts in the Context of the Application
- used the script module
```
You can pass command-line arguments to script modules and parse them out, but I decided to pass the arguments as environment variables instead. I didn’t want to pass passwords via command-line argument (those show up in the process list when you run the ps command), and it’s easier to parse out environment variables in the scripts than it is to parse command-line arguments.

# Using the script module to invoke custom Python code
- name: set the site id
  script: scripts/setsite.py
  environment:
    PATH: "{{ venv_path }}/bin"
    PROJECT_DIR: "{{ proj_path }}"
    WEBSITE_DOMAIN: "{{ live_hostname }}"
- name: set the admin password
  script: scripts/setadmin.py
  environment:
    PATH: "{{ venv_path }}/bin"
    PROJECT_DIR: "{{ proj_path }}"
    ADMIN_PASSWORD: "{{ admin_pass }}"
```

#### Setting Service Conguration Files 
- jija

#### Enabling the Nginx Conguration 
```
# Enabling nginx conguration
- name: enable the nginx config file
  file:
    src: /etc/nginx/sites-available/mezzanine.conf
    dest: /etc/nginx/sites-enabled/mezzanine.conf
    state: link
  notify: restart nginx
  sudo: True
- name: remove the default nginx config file
  file: path=/etc/nginx/sites-enabled/default state=absent
  notify: restart nginx
  sudo: True
```

#### Installing TLS Certi cates
```
- name: ensure config path exists
  file: path={{ conf_path }} state=directory
  sudo: True
  when: tls_enabled

- name: create tls certificates
  command: >
    openssl req -new -x509 -nodes -out {{ proj_name }}.crt
    -keyout {{ proj_name }}.key -subj '/CN={{ domains[0] }}' -days 3650
    chdir={{ conf_path }}
    creates={{ conf_path }}/{{ proj_name }}.crt
  sudo: True
  when: tls_enabled
  notify: restart nginx

# The chdir parameter changes directory before running the command. 
# The creates parameter implements idempotence: Ansible will first check to see if the file {{ conf_path }}/ {{ proj_name }}.crt exists on the host. If it already exists, then Ansible will skip this task.
```

#### Installing  Cron Job
```
installing cron job for polling twitter
- name: install poll twitter cron job
  cron: name="poll twitter" minute="*/5" user={{ user }} job="{{ manage }} poll_twitter"
```
```
#  deleting cron jobs by name. 
 - name: remove cron job
      cron: name="poll twitter" state=absent
```


#### The Full Playbook 
```
### mezzanine.yml: the complete playbook
---
- name: Deploy mezzanine
  hosts: web
  vars:
    user: "{{ ansible_ssh_user }}"
    proj_name: mezzanine-example
    venv_home: "{{ ansible_env.HOME }}"
    venv_path: "{{ venv_home }}/{{ proj_name }}"
    proj_dirname: project
    proj_path: "{{ venv_path }}/{{ proj_dirname }}"
    reqs_path: requirements.txt
    manage: "{{ python }} {{ proj_path }}/manage.py"
    live_hostname: 192.168.33.10.xip.io
    domains:
      - 192.168.33.10.xip.io
      - www.192.168.33.10.xip.io
    repo_url: git@github.com:lorin/mezzanine-example.git gunicorn_port: 8000
    locale: en_US.UTF-8
    # Variables below don't appear in Mezannine's fabfile.py # but I've added them for convenience
    conf_path: /etc/nginx/conf
    tls_enabled: True
    python: "{{ venv_path }}/bin/python"
    database_name: "{{ proj_name }}"
    database_user: "{{ proj_name }}"
    database_host: localhost
    database_port: 5432
    gunicorn_proc_name: mezzanine
vars_files:
  - secrets.yml
tasks:
  - name: install apt packages
    apt: pkg={{ item }} update_cache=yes cache_valid_time=3600
    sudo: True
    with_items:
      - git
      - libjpeg-dev
      - libpq-dev
      - memcached
      - nginx
      - postgresql
      - python-dev
      - python-pip
      - python-psycopg2
      - python-setuptools
      - python-virtualenv
      - supervisor
  - name: check out the repository on the host
    git: repo={{ repo_url }} dest={{ proj_path }} accept_hostkey=yes
  - name: install required python packages
    pip: name={{ item }} virtualenv={{ venv_path }}
    with_items:
      - gunicorn
      - setproctitle
      - south
      - psycopg2
      - django-compressor
      - python-memcached
  - name: install requirements.txt
    pip: requirements={{ proj_path }}/{{ reqs_path }} virtualenv={{ venv_path }}
  - name: create a user
    postgresql_user:
      name: "{{ database_user }}"
      password: "{{ db_pass }}"
    sudo: True
    sudo_user: postgres
  - name: create the database
    postgresql_db:
      name: "{{ database_name }}"
      owner: "{{ database_user }}"
    encoding: UTF8
    lc_ctype: "{{ locale }}"
    lc_collate: "{{ locale }}"
    template: template0
    sudo: True
    sudo_user: postgres

  - name: generate the settings file
    template:
      src: templates/local_settings.py.j2
      dest: "{{ proj_path }}/local_settings.py"
  
  - name: sync the database, apply migrations, collect static content
    django_manage:
      command: "{{ item }}"
      app_path: "{{ proj_path }}"
      virtualenv: "{{ venv_path }}"
    with_items:
      - syncdb
      - migrate
      - collectstatic
  
  - name: set the site id
    script: scripts/setsite.py
    environment:
      PATH: "{{ venv_path }}/bin"
      PROJECT_DIR: "{{ proj_path }}"
      WEBSITE_DOMAIN: "{{ live_hostname }}"
  
  - name: set the admin password
    script: scripts/setadmin.py
    environment:
      PATH: "{{ venv_path }}/bin"
      PROJECT_DIR: "{{ proj_path }}"
      ADMIN_PASSWORD: "{{ admin_pass }}"
  
  - name: set the gunicorn config file
    template:
      src: templates/gunicorn.conf.py.j2
      dest: "{{ proj_path }}/gunicorn.conf.py"
  
  - name: set the supervisor config file
    template:
      src: templates/supervisor.conf.j2
      dest: /etc/supervisor/conf.d/mezzanine.conf
    sudo: True
    notify: restart supervisor
  
  - name: set the nginx config file
    template:
      src: templates/nginx.conf.j2
      dest: /etc/nginx/sites-available/mezzanine.conf
        notify: restart nginx
        sudo: True
  
   - name: enable the nginx config file
     file:
       src: /etc/nginx/sites-available/mezzanine.conf
       dest: /etc/nginx/sites-enabled/mezzanine.conf
       state: link
     notify: restart nginx
     sudo: True
  
   - name: remove the default nginx config file
     file: path=/etc/nginx/sites-enabled/default state=absent
     notify: restart nginx
     sudo: True
  
   - name: ensure config path exists
     file: path={{ conf_path }} state=directory
     sudo: True
     when: tls_enabled
  
   - name: create tls certificates
     command: >
       openssl req -new -x509 -nodes -out {{ proj_name }}.crt
       -keyout {{ proj_name }}.key -subj '/CN={{ domains[0] }}' -days 3650
       chdir={{ conf_path }}
       creates={{ conf_path }}/{{ proj_name }}.crt
     sudo: True
     when: tls_enabled
     notify: restart nginx
  
   - name: install poll twitter cron job
     cron: name="poll twitter" minute="*/5" user={{ user }}
       job="{{ manage }} poll_twitter"
  
handlers:
  - name: restart supervisor
    supervisorctl: name=gunicorn_mezzanine state=restarted
    sudo: True
  - name: restart nginx
    service: name=nginx state=restarted
    sudo: True
```

####  Running the Playbook 
```
$ ansible-playbook mezzanine.yml
```


### Complex Playbooks
#### Running a Task on the Control Machine
- Sometimes you want to run a particular task on the control machine instead of on the remote host. Ansible provides the local_action clause for tasks to support this.
```
Imagine that the server we wanted to install Mezzanine onto had just booted, so that if we ran our playbook too soon, it would error out because the server hadn’t fully started up yet.
We could start off our playbook by invoking the wait_for module to wait until the SSH server was ready to accept connections before we executed the rest of the play‐ book. In this case, we want this module to execute on our laptop, not on the remote host.
The first task of our playbook would have to start off like this:
    - name: wait for ssh server to be running
      local_action: wait_for port=22 host="{{ inventory_hostname }}"
        search_regex=OpenSSH
```

#### Running a Task on a Machine Other Than the Host 
- You can use the `delegate_to` clause to run the task on a different host.
```
Using delegate_to with Nagios
- name: enable alerts for web servers
  hosts: web
  tasks:
    - name: enable alerts
      nagios: action=enable_alerts service=web host={{ inventory_hostname }}
      delegate_to: nagios.example.com
```
#### Manually Gathering Facts
```
# Waiting for ssh server to come up
- name: Deploy mezzanine
hosts: web
gather_facts: False
# vars & vars_files section not shown here tasks:
    - name: wait for ssh server to be running
      local_action: wait_for port=22 host="{{ inventory_hostname }}"
        search_regex=OpenSSH
    - name: gather facts
      setup:
```

#### Running on One Host at a Time
-  use the serial clause on a play to tell Ansible to restrict the number of hosts that a play runs on .
```
Removing hosts from load balancer and upgrading packages
- name: upgrade packages on servers behind load balancer
  hosts: myhosts
  serial: 1
  tasks:
    - name: get the ec2 instance id and elastic load balancer id
      ec2_facts:
    - name: take the host out of the elastic load balancer
      local_action: ec2_elb
      args:
        instance_id: "{{ ansible_ec2_instance_id }}"
        state: absent
    - name: upgrade packages
      apt: update_cache=yes upgrade=yes
    - name: put the host back in the elastic load balancer
      local_action: ec2_elb
      args:
        instance_id: "{{ ansible_ec2_instance_id }}"
        state: present
        ec2_elbs: "{{ item }}"
      with_items: ec2_elbs
```
- use a max_fail_percentage clause along with the serial clause to specify the maximum percentage of failed hosts before Ansible fails the entire play. 
`If you want Ansible to fail if any of the hosts fail a task, set the max_fail_percentage to 0.`

#### Running Only Once
- use the run_once clause to tell Ansible to run the command only once
```
  - name: run the database migrations
      command: /opt/run_migrations
      run_once: true
```
```
# Using run_once can be particularly useful when using local_action if your playbook involves multiple hosts, and you want to run the local task only once:
    - name: run the task locally, only once
      local_action: command /opt/my-custom-command
      run_once: true
```

#### Dealing with Badly Behaved Commands: changed_when and failed_when
- failed_when 
```
# Viewing the output of a task
- name: initialize the database
  django_manage:
    command: createdb --noinput --nodata
    app_path: "{{ proj_path }}"
    virtualenv: "{{ venv_path }}"
  failed_when: False
  register: result
- debug: var=result
- fail:
# capture the output of a failed task, you add a register clause to save the output to a variable and a failed_when: False clause so that the execution doesn’t stop even if the module returns failure. Then add a debug task to print out the variable, and finally a fail clause so that the playbook stops executing,
```
- changed_when
 `changed is set to false even though it did, indeed, change the state of the database`
```
# Idempotent manage.py createdb
- name: initialize the database
  django_manage:
    command: createdb --noinput --nodata
    app_path: "{{ proj_path }}"
    virtualenv: "{{ venv_path }}"
  register: result
  changed_when: not result.failed and "Creating tables" in result.out
  failed_when: result.failed and "Database already created" not in result.msg
```

#### Retrieving the IP Address from the Host
- Ansible retrieves the IP address of each host and stores it as a fact
```
# define our variables like this:
    live_hostname: "{{ ansible_eth1.ipv4.address }}.xip.io"
    domains:
      - ansible_eth1.ipv4.address.xip.io
      - www.ansible_eth1.ipv4.address.xip.io
```

#### Encrypting Sensitive Data with Vault
- ansible-vault `allows you to create and edit an encrypted file that ansible-playbook will recognize and decrypt automatically, given the pass‐ word.`    
```
We can encrypt an existing file like this:
    $ ansible-vault encrypt secrets.yml
Alternately, we can create a new encrypted secrets.yml file by doing:
    $ ansible-vault create secrets.yml
We do need to tell ansible-playbook to prompt us for the password of the encrypted file, or it will simply error out. Do so by using the --ask-vault-pass argument:
    $ ansible-playbook mezzanine.yml --ask-vault-pass
You can also store the password in a text file and tell ansible-playbook the location
of this password file using the --vault-password-file flag:
    $ ansible-playbook mezzanine --vault-password-file ~/password.txt 
```

#### Patterns for Specifying Hosts
```
Action                      Example usage
------------------------------------------
All hosts                   all
All hosts                        *
Union                       dev:staging
Intersection                staging:&database
Exclusion                   dev:!queue
Wildcard                     *.example.com
Range of numbered servers   web[5:10]
Regular expression          ~web\d\.example\.(com
  
```

#### Limiting Which Hosts Run
```
Limiting which hosts run
    $ ansible-playbook -l hosts playbook.yml
    $ ansible-playbook --limit hosts playbook.yml
You can use the pattern syntax just described to specify arbitrary combinations of hosts. For example:
    $ ansible-playbook -l 'staging:&database' playbook.yml
```

#### Filters
`Filters are a feature of the Jinja2 templating engine. Since Ansible uses Jinja2 for eval‐ uating variables, as well as for templates, you can use filters inside of {{ braces }} in your playbooks, as well as inside of your template files.`
- The Default Filter
```
The default filter is a useful one.
  "HOST": "{{ database_host | default('localhost') }}",
```
- Filters for Registered Variables
`we want to run a task and print out its output, even if the task fails. However, if the task did fail, we want Ansible to fail for that host after printing the output. shows how we would use the failed filter in the argument to the failed_when clause.`
```
- name: Run myprog
  command: /opt/myprog
  register: result
  ignore_errors: True
- debug: var=result
- debug: msg="Stop running the playbook if myprog failed"
  failed_when: result|failed
# more tasks here
```
```
# Task return value  lters
Name        Description
------------------------------
failed      True if a registered value is a task that failed
changed     True if a registered value is a task that changed 
success     True if a registered value is a task that succeeded 
skipped     True if a registered value is a task that was skipped

```

#### Filters That Apply to File Paths 
```
# Consider this playbook fragment:
      vars:
        homepage: /usr/share/nginx/html/index.html
      tasks:
      - name: copy home page
        copy: src=files/index.html dest={{ homepage }}
```
```
# The basename filter will let us extract the index.html part of the filename from the full path, allowing us to write the playbook without repeating the filename:1
      vars:
        homepage: /usr/share/nginx/html/index.html
      tasks:
      - name: copy home page
        copy: src=files/{{ homepage | basename }} dest={{ homepage }}
```

#### Writing Your Own Filter
Ansible will look for custom filters in the filter_plugins directory, relative to the directory where your playbooks are.
```
# filter_plugins/surround_by_quotes.py
# From http://stackoverflow.com/a/15515929/742

def surround_by_quote(a_list):
    return ['"%s"' % an_element for an_element in a_list]

class FilterModule(object): def filters(self):
    return {'surround_by_quote': surround_by_quote}

#The FilterModule class is Ansible-specific code that makes the Jinja2 filter available to Ansible.
#filter plug-ins in the /usr/share/ansible_plugins/ lter_plugins direc‐ tory, or you can specify the directory by setting the ANSIBLE_FILTER_PLUGINS envi‐ ronment variable to the directory where your plug-ins are located. 
```

#### Lookups
- Ansible has a feature called lookups that allows you to read in configuration data from various sources and then use that data in your playbooks and template.
```
Name Description
file            Contents of a file
password        Randomly generate a password
pipe            Output of locally executed command
env             Environment variable
template        Jinja2 template after evaluation 
csvfile         Entry in a .csv  le
dnstxt          DNS TXT record 
redis_kv        Redis key lookup 
etcd            etcd key lookup
```
-  file 
```
# Using the  file lookup
- name: Add my public key as an EC2 key
  ec2_key: name=mykey key_material="{{ lookup('file', \
  '/Users/lorinhochstein/.ssh/id_rsa.pub') }}"
------
#  authorized_keys.j2
{{ lookup('file', '/Users/lorinhochstein/.ssh/id_rsa.pub') }}

#  to generate authorized_keys
- name: copy authorized_host file
  template: src=authorized_keys.j2 dest=/home/deploy/.ssh/authorized_keys

```

- pipe `The pipe lookup invokes an external program on the control machine and evaluates to the program’s output on standard out.`
```
  - name: get SHA of most recent commit
      debug: msg="{{ lookup('pipe', 'git rev-parse HEAD') }}"
---
The output would look something like this:
    TASK: [get the sha of the current commit] *************************************
    ok: [myserver] => {
        "msg": "e7748af0f040d58d61de1917980a210df419eae9"
    }
```

- env `The env lookup retrieves the value of an environment variable set on the control machine.`
```
  - name: get the current shell
      debug: msg="{{ lookup('env', 'SHELL') }}"
----- 
Since I use Zsh as my shell, the output looks like this when I run it:
    TASK: [get the current shell] *************************************************
    ok: [myserver] => {
        "msg": "/bin/zsh"
    }
```

- password  `lookup evaluates to a random password, and it will also write the pass‐ word to a file specified in the argument.`
```
# create a Post‐ gres user named deploy with a random password and write that password to deploy- password.txt on the control machine
- name: create deploy postgres user
      postgresql_user:
      name: deploy
      password: "{{ lookup('password', 'deploy-password.txt') }}"
```

- template `lets you specify a Jinja2 template file, and then returns the result of evaluating the template`
```
# message.j2
This host runs {{ ansible_distribution }}

And we defined a task like this:
    - name: output message from template
      debug: msg="{{ lookup('template', 'message.j2') }}"

# Then we'd see output that looks like this:
    TASK: [output message from template] ******************************************
    ok: [myserver] => {
        "msg": "This host runs Ubuntu\n"
    }
```

- csvfile  `reads an entry from a .csv file`
```
#  users.csv
username,email
lorin,lorin@ansiblebook.com
john,john@example.com
sue,sue@example.org

# If we wanted to extract Sue’s email address using the csvfile lookup plug-in, we would invoke the lookup plug-in like this:
    lookup('csvfile', 'sue file=users.csv delimiter=, col=1')

# If the user name we wanted to look up was stored in a variable named username, we could construct the argument string by using the + sign to concatenate the username string with the rest of the argument string:
    lookup('csvfile', username + ' file=users.csv delimiter=, col=1')

```

- dnxtxt `queries the DNS server for the TXT record associated with the host.`
*The dnstxt module requires that you install the dnspython Python package on the control machine.*
*The DNS protocol supports another type of record that you can associate with a host‐ name, called a TXT record. A TXT record is just an arbitrary string that you can attach to a hostname. Once you’ve associated a TXT record with a hostname, any‐ body can retrieve the text using a DNS client.*
```
    - name: look up TXT record
      debug: msg="{{ lookup('dnstxt', 'ansiblebook.com') }}"

# The output would look like this:
    TASK: [look up TXT record] ****************************************************
    ok: [myserver] => {
        "msg": "isbn=978-1491915325"
    }
```
`DNS service providers typically have web interfaces to let you perform DNS-related tasks such as creating TXT records.  `


- redis_kv      
*The redis_kv module requires that you install the redis Python package on the control machine.*       
*DNS service providers typically have web interfaces to let you perform DNS-related tasks such as creating TXT records.*      
You can use the redis_kv lookup to retrieve the value of a key. The key must be a string, as the module does the equivalent of call‐ ing the Redis GET command.    
```
# we had set the key weather to the value sunny, by doing something like this:
$ redis-cli SET weather sunny

# If we defined a task in our playbook that invoked the Redis lookup:
    - name: look up value in Redis
      debug: msg="{{ lookup('redis_kv', 'redis://localhost:6379,weather') }}"

# The output would look like this:
    TASK: [look up value in Redis] ************************************************
    ok: [myserver] => {
        "msg": "sunny"
    }
```

- etcd     
Etcd is a distributed key-value store, commonly used for keeping configuration data and for implementing service discovery    
```
# we had set the key weather to the value cloudy by doing something like this:
$ curl -L http://127.0.0.1:4001/v2/keys/weather -XPUT -d value=cloudy 
# If we defined a task in our playbook that invoked the etcd plug-in:
    - name: look up value in etcd
      debug: msg="{{ lookup('etcd', 'weather') }}"
# The output would look like this:
    TASK: [look up value in etcd] *************************************************
    ok: [localhost] => {
        "msg": "cloudy"
    }
```
`By default, the etcd lookup will look for the etcd server at http://127.0.0.1:4001, but you can change this by setting the ANSIBLE_ETCD_URL environment variable before invoking ansible-playbook.`


#### Writing Your Own Lookup Plug-in
自己自习看文档 ！

#### More Complicated Loops
```
# Looping constructs
Name                     Input                    Looping strategy
| with_items               | list                   |    Loop over list elements
| with_lines               | command to execute     |    Loop over lines in command output
| with_fileglob            | glob                   |    Loop over  filenames
| with_first_found         | list of paths          |    First  file in input that exists
| with_dict                | dictionary             |    Loop over dictionary elements 
| with_attened             | list of lists          |    Loop over  attened list
| with_indexed_items       | list                   |    Single iteration
| with_nested              | list                   |    Nested loop
| with_random_choice       | list                   |    Single iteration
| with_sequence            | sequence of integers   |    Loop over sequence
| with_subelements         | list of dictionaries   |    Nested loop
| with_together            | list of lists          |    Loop over zipped list 
| with_inventory_hostnames | host pattern           |    Loop over matching hosts
```

- with_lines     
`The with_lines looping construct lets you run an arbitrary command on your con‐ trol machine and iterate over the output, one line at a time`    
```
# Imagine you have a file that contains a list of names, and you want to send a Slack message for each name, something like this:
    Leslie Lamport
    Silvio Micali
    Shafi Goldwasser
    Judea Pearl
#  shows how you can use with_lines to read a file and iterate over its contents line by line.
#  Using with_lines as a loop
- name: Send out a slack message
  slack:
    domain: example.slack.com
    token: "{{ slack_token }}"
    msg: "{{ item }} was in the list"
  with_lines:
    - cat files/turing.txt
```

- with_fileglob       
`The with_fileglob construct is useful for iterating over a set of files on the control machine.`      
```
# shows how to iterate over files that end in .pub in the /var/keys direc‐ tory, as well as a keys directory next to your playbook. It then uses the file lookup plug-in to extract the contents of the file, which are passed to the authorized_key module.
# Using with_leglob to add keys
- name: add public keys to account
  authorized_key: user=deploy key="{{ lookup('file', item) }}"
  with_fileglob:
    - /var/keys/*.pub
    - keys/*.pub
```

- with_dict    
`The with_dict lets you iterate over a dictionary instead of a list.`      
```
if your host has an eth0 interface, then there will be an Ansible fact named ansible_eth0, with a key named ipv4 that contains a dictionary that looks something like this:
    {
        "address": "10.0.2.15",
        "netmask": "255.255.255.0",
        "network": "10.0.2.0"
    }
# We could iterate over this dictionary and print out the entries one at a time by doing:
     - name: iterate over ansible_eth0
        debug: msg={{ item.key }}={{ item.value }}
        with_dict: ansible_eth0.ipv4
# The output would look like this:
    TASK: [iterate over ansible_eth0] *********************************************
    ok: [myserver] => (item={'key': u'netmask', 'value': u'255.255.255.0'}) => {
        "item": {
            "key": "netmask",
            "value": "255.255.255.0"
    },
        "msg": "netmask=255.255.255.0"
    }
    ok: [myserver] => (item={'key': u'network', 'value': u'10.0.2.0'}) => {
        "item": {
            "key": "network",
            "value": "10.0.2.0"
        },
        "msg": "network=10.0.2.0"
    }
    ok: [myserver] => (item={'key': u'address', 'value': u'10.0.2.15'}) => {
        "item": {
            "key": "address",
            "value": "10.0.2.15"
        },
        "msg": "address=10.0.2.15"
    }

```



### Roles: Scaling Up Your Playbooks
```
In Ansible, the role is the primary mechanism for breaking apart a playbook into multiple files. This simplifies writing complex playbooks, and it also makes them eas‐ ier to reuse. Think of a role as something you assign to one or more hosts. For exam‐ ple, you’d assign a database role to the hosts that will act as database servers
```
- Basic Structure of a Role
```
# An Ansible role has a name, such as “database.” Files associated with the database role go in the roles/database directory, which contains the following files and directories.
roles/database/tasks/main.yml
    Tasks
roles/database/ les/
    Holds files to be uploaded to hosts
roles/database/templates/
    Holds Jinja2 template files
roles/database/handlers/main.yml
    Handlers
roles/database/vars/main.yml
    Variables that shouldn’t be overridden
roles/database/defaults/main.yml
    Default variables that can be overridden
roles/database/meta/main.yml
    Dependency information about a role
# Each individual file is optional; if your role doesn’t have any handlers, there’s no need to have an empty handlers/main.yml file.
```
```
# Where Does Ansible Look for My Roles?
# Ansible will look for roles in the roles directory alongside your . playbooks. It will also look for systemwide roles in /etc/ansible/roles. You can customize the system-wide location of roles by setting the roles_path setting in the defaults section of your ansi‐ ble.cfg file, as shown in Example 8-1.
#  overriding default roles path

[defaults]
roles_path = ~/ansible_roles

# You can also override this by setting the ANSIBLE_ROLES_PATH environment variable, as described in Appendix B.
```

#### Example: Database and Mezzanine Roles
- Using Roles in Your Playbooks
```
# Before we get into the details of how to define roles, let’s go over how to assign roles to hosts in a playbook.
# shows what our playbook looks like for deploying Mezzanine onto a sin‐ gle host, once we have database and Mezzanine roles defined.
#  mezzanine-single-host.yml

- name: deploy mezzanine on vagrant
  hosts: web
  vars_files:
    - secrets.yml
  roles:
    - role: database
      database_name: "{{ mezzanine_proj_name }}"
      database_user: "{{ mezzanine_proj_name }}"
    - role: mezzanine
      live_hostname: 192.168.33.10.xip.io
      domains:
        - 192.168.33.10.xip.io
        - www.192.168.33.10.xip.io

# When we use roles, we have a roles section in our playbook. The roles section expects a list of roles. 
```

#### Pre-Tasks and Post-Tasks
```
# Sometimes you want to run some tasks before or after you invoke your roles. Let’s say you wanted to update the apt cache before you deployed Mezzanine, and you wanted to send a notification to Slack channel after you deployed.

# Using pre-tasks and post-tasks
- name: deploy mezzanine on vagrant
  hosts: web
  vars_files:
    - secrets.yml
  pre_tasks:
    - name: update the apt cache
      apt: update_cache=yes
  roles:
    - role: mezzanine
      database_host: "{{ hostvars.db.ansible_eth1.ipv4.address }}"
      live_hostname: 192.168.33.10.xip.io
      domains:
        - 192.168.33.10.xip.io
        - www.192.168.33.10.xip.io
  post_tasks:
    - name: notify Slack that the servers have been updated
      local_action: >
        slack
        domain=acme.slack.com
        token={{ slack_token }}
        msg="web server {{ inventory_hostname }} configured"
```


#### A “Database” Role for Deploying the Database
```
# The job of our “database” role will be to install Postgres and create the required data‐ base and database user.
# Our database role involves the following files:
• roles/database/tasks/main.yml
• roles/database/defaults/main.yml • 
  roles/database/handlers/main.yml • 
  roles/database/fies/pg_hba.conf
• roles/database/fies/postgresql.conf

# This role includes two customized Postgres configuration files.

postgresql.conf
# Modifies the default listen_addresses configuration option so that Postgres will accept connections on any network interface. The default for Postgres is to accept connections only from localhost, which doesn’t work for us if we want our data‐ base to run on a separate host from our web application.

pg_hba.conf
# Configures Postgres to authenticate connections over the network using user‐ name and password.
```
```
# roles/database/tasks/main.yml
- name: install apt packages
  apt: pkg={{ item }} update_cache=yes cache_valid_time=3600
  sudo: True
  with_items:
    - libpq-dev
    - postgresql
    - python-psycopg2
- name: copy configuration file
  copy: >
    src=postgresql.conf dest=/etc/postgresql/9.3/main/postgresql.conf
    owner=postgres group=postgres mode=0644
  sudo: True
  notify: restart postgres
- name: copy client authentication configuration file
  copy: >
    src=pg_hba.conf dest=/etc/postgresql/9.3/main/pg_hba.conf
    owner=postgres group=postgres mode=0640
  sudo: True
  notify: restart postgres
- name: create a user
  postgresql_user:
    name: "{{ database_user }}"
    password: "{{ db_pass }}"
  sudo: True
  sudo_user: postgres
- name: create the database
  postgresql_db:
    name: "{{ database_name }}"
    owner: "{{ database_user }}"
    encoding: UTF8
    lc_ctype: "{{ locale }}"
    lc_collate: "{{ locale }}"
    template: template0
  sudo: True
  sudo_user: postgres


# roles/database/handlers/main.yml
- name: restart postgres
  service: name=postgresql state=restarted
  sudo: True


#  roles/database/defaults/main.yml
database_port: 5432
```
```
# Note that our list of tasks refers to several variables that we haven’t defined anywhere in the role:
    database_name
    database_user
    db_pass
    locale
# we pass in database_name and database_user when we invoke the role. I’m assuming that db_pass is defined in the secrets.yml file, which is included in the vars_files section. The locale variable is likely something that would be the same for every host, and might be used by multiple roles or play‐ books, so I defined it in the group_vars/all file in the code samples that accompany this book.
```
`If you think you might want to change the value of a variable in a role, use a default variable. If you don’t want it to change, then use a regular variable.`

##### A “Mezzanine” Role for Deploying Mezzanine      
`The job of our “mezzanine” role will be to install Mezzanine. This includes installing nginx as the reverse proxy and supervisor as the process monitor.`    
```
Here are the files that are involved:
     roles/mezzanine/defaults/main.yml
     roles/mezzanine/handlers/main.yml
     roles/mezzanine/tasks/django.yml
     roles/mezzanine/tasks/main.yml
     roles/mezzanine/tasks/nginx.yml
     roles/mezzanine/templates/gunicorn.conf.py.j2
     roles/mezzanine/templates/local_settings.py.filters.j2
     roles/mezzanine/templates/local_settings.py.j2
     roles/mezzanine/templates/nginx.conf.j2
     roles/mezzanine/templates/supervisor.conf.j2
     roles/mezzanine/vars/main.yml
```
```
It’s good practice to do this with role variables because Ansible doesn’t have any notion of namespace across roles. This means that variables that are defined in other roles, or elsewhere in a play‐ book, will be accessible everywhere. This can cause some unexpected behavior if you accidentally use the same variable name in two different roles.
```
```
# roles/mezzanine/vars/main.yml
# vars file for mezzanine
mezzanine_user: "{{ ansible_ssh_user }}"
mezzanine_venv_home: "{{ ansible_env.HOME }}"
mezzanine_venv_path: "{{ mezzanine_venv_home }}/{{ mezzanine_proj_name }}"
mezzanine_repo_url: git@github.com:lorin/mezzanine-example.git
mezzanine_proj_dirname: project
mezzanine_proj_path: "{{ mezzanine_venv_path }}/{{ mezzanine_proj_dirname }}"
mezzanine_reqs_path: requirements.txt
mezzanine_conf_path: /etc/nginx/conf
mezzanine_python: "{{ mezzanine_venv_path }}/bin/python"
mezzanine_manage: "{{ mezzanine_python }} {{ mezzanine_proj_path }}/manage.py"
mezzanine_gunicorn_port: 8000

#shows the default variables defined on our mezzanine role. In this case, we have only a single variable. When I write default variables, I’m less likely to prefix them because I might intentionally want to override them elsewhere.
```
```
# roles/mezzanine/defaults/main.yml
tls_enabled: True
```
```
# roles/mezzanine/tasks/main.yml
- name: install apt packages
  apt: pkg={{ item }} update_cache=yes cache_valid_time=3600
  sudo: True
  with_items:
    - git
    - libjpeg-dev
    - libpq-dev
    - memcached
    - nginx
    - python-dev
    - python-pip
    - python-psycopg2
    - python-setuptools
    - python-virtualenv
    - supervisor
- include: django.yml
- include: nginx.yml
```
```
#  roles/mezzanine/tasks/django.yml
- name: check out the repository on the host
  git:
    repo: "{{ mezzanine_repo_url }}"
    dest: "{{ mezzanine_proj_path }}"
    accept_hostkey: yes
- name: install required python packages
  pip: name={{ item }} virtualenv={{ mezzanine_venv_path }}
  with_items:
    - gunicorn
    - setproctitle
    - south
    - psycopg2
    - django-compressor
    - python-memcached
- name: install requirements.txt
  pip: >
    requirements={{ mezzanine_proj_path }}/{{ mezzanine_reqs_path }}
    virtualenv={{ mezzanine_venv_path }}
- name: generate the settings file
  template: src=local_settings.py.j2 dest={{ mezzanine_proj_path }}/local_settings.py
- name: sync the database, apply migrations, collect static content
  django_manage:
    command: "{{ item }}"
    app_path: "{{ mezzanine_proj_path }}"
    virtualenv: "{{ mezzanine_venv_path }}"
  with_items:
    - syncdb
    - migrate
    - collectstatic
- name: set the site id
  script: scripts/setsite.py
  environment:
    PATH: "{{ mezzanine_venv_path }}/bin"
    PROJECT_DIR: "{{ mezzanine_proj_path }}"
    WEBSITE_DOMAIN: "{{ live_hostname }}"
- name: set the admin password
  script: scripts/setadmin.py
  environment:
    PATH: "{{ mezzanine_venv_path }}/bin"
    PROJECT_DIR: "{{ mezzanine_proj_path }}"
    ADMIN_PASSWORD: "{{ admin_pass }}"
- name: set the gunicorn config file
  template: src=gunicorn.conf.py.j2 dest={{ mezzanine_proj_path }}/gunicorn.conf.py
- name: set the supervisor config file
  template: src=supervisor.conf.j2 dest=/etc/supervisor/conf.d/mezzanine.conf
  sudo: True
  notify: restart supervisor
- name: ensure config path exists
  file: path={{ mezzanine_conf_path }} state=directory
  sudo: True
  when: tls_enabled
- name: install poll twitter cron job
  cron: >
    name="poll twitter" minute="*/5" user={{ mezzanine_user }}
    job="{{ mezzanine_manage }} poll_twitter"
```
```
#  roles/mezzanine/tasks/nginx.yml
- name: set the nginx config file
  template: src=nginx.conf.j2 dest=/etc/nginx/sites-available/mezzanine.conf
  notify: restart nginx
  sudo: True
- name: enable the nginx config file
  file:
    src: /etc/nginx/sites-available/mezzanine.conf
    dest: /etc/nginx/sites-enabled/mezzanine.conf
    state: link
  notify: restart nginx
  sudo: True
- name: remove the default nginx config file
  file: path=/etc/nginx/sites-enabled/default state=absent
  notify: restart nginx
  sudo: True
- name: create tls certificates
  command: >
    openssl req -new -x509 -nodes -out {{ mezzanine_proj_name }}.crt
    -keyout {{ mezzanine_proj_name }}.key -subj '/CN={{ domains[0] }}' -days 3650
    chdir={{ mezzanine_conf_path }}
    creates={{ mezzanine_conf_path }}/{{ mezzanine_proj_name }}.crt
  sudo: True
  when: tls_enabled
  notify: restart nginx

# There’s one important difference between tasks defined in a role and tasks defined in a regular playbook, and that’s when using the copy or template modules.
# When invoking copy in a task defined in a role, Ansible will first check the rolename/files/ directory for the location of the file to copy. Similarly, when invoking template in a task defined in a role, Ansible will first check the rolename/templates directory for the location of the template to use.
```
```
#  roles/mezzanine/handlers/main.yml
- name: restart supervisor
  supervisorctl: name=gunicorn_mezzanine state=restarted
  sudo: True
- name: restart nginx
  service: name=nginx state=restarted
  sudo: True
```

#### Creating Role Files and Directories with ansible-galaxy

```
# 
$ ansible-galaxy init -p playbooks/roles web

# The -p flag tells ansible-galaxy where your roles directory is. If you don’t specify it, then the role files will be created in your current directory. Running the command creates the following files and directories:
 playbooks/roles/web/tasks/main.yml
 playbooks/roles/web/handlers/main.yml
 playbooks/roles/web/vars/main.yml
 playbooks/roles/web/defaults/main.yml
 playbooks/roles/web/meta/main.yml
 playbooks/roles/web/ les/
 playbooks/roles/web/templates/
 playbooks/roles/web/README.md
```

#### Dependent Roles
```
# the web role depends on the ntp role by creating a roles/web/meta/ main.yml file and listing it as a role, with a parameter, as shown in Example 8-14.
dependencies:
    - { role: ntp, ntp_server=ntp.ubuntu.com }

```
 更多看官方文档


#### Ansible Galaxy
```

$ ansible-galaxy install -p ./roles bennojoy.ntp

$ ansible-galaxy list

$ ansible-galaxy remove bennojoy.ntp

```



### Making Ansible Go Even Faster
#### SSH Multiplexing and ControlPersist
OpenSSH supports an optimization called SSH multiplexing, which is also referred to as ControlPersist.   
```
When you enable multiplexing:
    The first time you try to SSH to a host, OpenSSH starts a master connection.
    OpenSSH creates a Unix domain socket (known as the control socket) that is asso‐ ciated with the remote host.
    The next time you try to SSH to a host, OpenSSH will use the control socket to communicate with the host instead of making a new TCP connection.

The master connection stays open for a user-configurable amount of time, and then the SSH client will terminate the connection. Ansible uses a default of 60 seconds.
```

#### Manually Enabling SSH Multiplexing
```
# ssh/con g for enabling ssh multiplexing
Host myserver.example.com
  ControlMaster auto
  ControlPath /tmp/%r@%h:%p
  ControlPersist 10m
```

####  SSH Multiplexing Options in Ansible
```
# Ansible's SSH multiplexing options
Option          Value
-------------------------------------------------------
ControlMaster   auto
ControlPath     $HOME/.ansible/cp/ansible-ssh-%h-%p-%r
ControlPersist  60s
```
```
I’ve never needed to change Ansible’s default ControlMaster or ControlPersist val‐ ues. However, I have needed to change the value for the ControlPath option. That’s because the operating system sets a maximum length on the path of a Unix domain socket, and if the ControlPath string is too long, then multiplexing won’t work. Unfortunately, Ansible won’t tell you if the ControlPath string is too long; it will sim‐ ply run without using SSH multiplexing.
The workaround is to configure Ansible to use a shorter ControlPath. The official documentation recommends setting this option in your ansible.cfg file:
    [ssh_connection]
    control_path = %(directory)s/%%h-%%r
Ansible sets %(directory)s to $HOME/.ansible.cp, and the double percent signs (%%) are needed to escape these characters because percent signs are special characters for files in .ini format.
```
#### Pipelining
```
Recall how Ansible executes a task:
1. It generates a Python script based on the module being invoked.
2. Then it copies the Python script to the host.
3. Finally, it executes the Python script.

Ansible supports an optimization called pipelining, where it will execute the Python script by piping it to the SSH session instead of copying it. This saves time because it tells Ansible to use one SSH session instead of two.
```

#### Enabling Pipelining
```
# Pipelining is off by default because it can require some configuration on your remote hosts, but I like to enable it because it speeds up execution. To enable it, modify your ansible.cfg file as shown .
# ansible.cfg Enable pipelining
[defaults]
pipelining = True
```

#### Configuring Hosts for Pipelining
```
# For pipelining to work, you need to make sure that the requiretty is not enabled in your /etc/sudoers file on your hosts. Otherwise, you'll get errors.

# templates/disable-requiretty.j2
Defaults:{{ ansible_ssh_user }} !requiretty

# Then run the playbook , replacing myhosts with your hosts. Don’t forget to disable pipelining before you do this, or the playbook will fail with an error.

# disable-requiretty.yml
- name: do not require tty for ssh-ing user
  hosts: myhosts
  sudo: True
  tasks:
    - name: Set a sudoers file to disable tty
      template: >
        src=templates/disable-requiretty.j2
        dest=/etc/sudoers.d/disable-requiretty
        owner=root group=root mode=0440
        validate="visudo -cf %s"
# Note the use of validate="visudo -cf %s". See “Validating Files” on page 277 for a discussion of why it’s a good idea to use validation when modifying sudoers files.
```

#### Fact Caching
```
# If your play doesn’t reference any Ansible facts, you can turn off fact gathering for that play. Recall that you can disable fact gathering with the gather_facts clause in a play, for example:
    - name: an example play that doesn't need facts
      hosts: myhosts
      gather_facts: False
      tasks:
        # tasks go here:

# You can disable fact gathering by default by adding the following to your ansible.cfg file:
   [defaults]
    gathering = explicit
# If you write plays that do reference facts, you can use fact caching so that Ansible gathers facts for a host only once, even if you rerun the playbook or run a different playbook that connects to the same host.
# If fact caching is enabled, Ansible will store facts in a cache the first time it connects to hosts. For subsequent playbook runs, Ansible will look up the facts in the cache instead of fetching them from the remote host, until the cache expires.

# shows the lines you must add to your ansible.cfg file to enable fact cach‐ ing. The fact_caching_timeout value is in seconds, and the example uses a 24-hour (86,400 second) timeout.
#  ansible.cfg Enable fact caching
    [defaults]
    gathering = smart
# 24-hour timeout, adjust if needed 
    fact_caching_timeout = 86400
# You must specify a fact caching implementation
    fact_caching = ...
•   (three fact-caching implementations: JSON files • Redis • Memcached)
#Setting the gathering configuration option to “smart” in ansible.cfg tells Ansible to use smart gathering. This means that Ansible will only gather facts if they are not present in the cache or if the cache has expired.
```
```
# 注意:
As with all caching-based solutions, there’s always the danger of the cached data becoming stale. Some facts, such as the CPU architec‐ ture (stored in the ansible_architecture fact), are unlikely to change often. Others, such as the date and time reported by the machine (stored in the ansible_date_time fact), are guaranteed to change often.
If you decide to enable fact caching, make sure you know how quickly the facts used in your playbook are likely to change, and set an appropriate fact caching timeout value. If you want to clear the fact cache before running a playbook, pass the --flush-cache flag to ansible-playbook.
if you want to use fact caching, make sure your playbooks do not explicitly specify gather_facts: True or gather_facts: False. With smart gathering enabled in the configuration file, Ansible will gather facts only if they are not present in the cache.
```

#### JSON File Fact-Caching Backend
```
# ansible.cfg with JSON fact caching
    [defaults]
    gathering = smart
# 24-hour timeout, adjust if needed
    fact_caching_timeout = 86400
# JSON file implementation
    fact_caching = jsonfile
    fact_caching_connection = /tmp/ansible_fact_cache
# Use the fact_caching_connection configuration option to specify a directory where Ansible should write the JSON files that contain the facts. If the directory does not exist, Ansible will create it.
# Ansible uses the file modification time to determine whether the fact-caching time‐ out has occurred yet.
```

#### Redis Fact Caching Backend
```
#Redis is a popular key-value data store that is often used as a cache. To enable fact caching using the Redis backend, you need to:
#1. Install Redis on your control machine.
#2. Ensure the Redis service is running on the control machine.
#3. Install the Python Redis package.
#4. Modify ansible.cfg to enable fact caching with Redis.
# shows how to configure ansible.cfg to use Redis as the cache backend. Example 9-9. ansible.cfg with Redis fact caching
    [defaults]
    gathering = smart
# 24-hour timeout, adjust if needed
    fact_caching_timeout = 86400
    fact_caching = redis
    Ansible needs the Python Redis package on the control machine, which you can install using pip:3
    $ pip install redis
```
#### Memcached Fact Caching Backend
```
#Memcached is another popular key-value data store that is often used as a cache. To enable fact caching using the Memcached backend, you need to:
# 1. Install Memcached on your control machine.
# 2. Ensure the Memcached service is running on the control machine.
# 3. Install the Python Memcached Python package.
# 4. Modify ansible.cfg to enable fact caching with Memcached.
# shows how to configure ansible.cfg to use Memcached as the cache backend.
# ansible.cfg with Memcached fact caching
# [defaults]
 gathering = smart
# 24-hour timeout, adjust if needed
 fact_caching_timeout = 86400
 fact_caching = memcached
# Ansible needs the Python Memcached package on the control machine, which you can install using pip. You might need to sudo or activate a virtualenv, depending on how you installed Ansible on your control machine.
 $ pip install python-memcached
```

#### Parallelism
```
# For each task, Ansible will connect to the hosts in parallel to execute the tasks. But Ansible doesn’t necessarily connect to all of the hosts in parallel. Instead, the level of parallelism is controlled by a parameter, which defaults to 5. You can change this default parameter in one of two ways.

# Setting ANSIBLE_FORKS
    $ export ANSIBLE_FORKS=20
    $ ansible-playbook playbook.yml

# ansible.cfg Con guring number of forks
    [defaults]
    forks = 20
```

###  Custom Modules
#### Example: Checking That We Can Reach a Remote Server
- Using the Script Module Instead of Writing Your Own
    ```
    # we could create a script file called playbooks/scripts/can_reach.sh that accepts as arguments the name of a host, the port to connect to, and how long it should try to connect before timing out.
        can_reach.sh www.example.com 80 1
    # can_reach.sh
        #!/bin/bash
        host=$1
        port=$2
        timeout=$3
        nc -z -w $timeout $host $port
    
    # We can then invoke this by doing:
        - name: run my custom script
          script: scripts/can_reach.sh www.example.com 80 1
    ```

- can_reach as a Module 
    ```
    # let’s implement can_reach as a proper Ansible module, which we will be able to invoke like this:
        - name: check if host can reach the database server
          can_reach: host=db.example.com port=5432 timeout=1
    # This will check if the host can make a TCP connection to db.example.com on port 5432. It will time out after one second if it fails to make a connection.
    ```

- Where to Put Custom Modules
    ```
    Ansible will look in the library directory relative to the playbook. In our example, we put our playbooks in the playbooks directory, so we will put our custom module at playbooks/library/can_reach.
    ```
- How Ansible Invokes Modules
    - Generate a Standalone Python Script with the Arguments (Python Only)
        ```
        If the module is written in Python and uses the helper code that Ansible provides (described later), then Ansible will generate a self-contained Python script that injects helper code, as well as the module arguments.
        ```
    - Copy the Module to the Host
        ```
        Ansible will copy the generated Python script (for Python-based modules) or the local file playbooks/library/can_reach (for non-Python-based modules) to a tempo‐ rary directory on the remote host.
         If you are accessing the remote host as the ubuntu user, Ansible will copy the file to a path that looks like the following:
        /home/ubuntu/.ansible/tmp/ansible-tmp-1412459504.14-47728545618200/can_reach
        ```
    - Create an Arguments File on the Host (Non-Python Only)
        ```
        # If the module is not written in Python, Ansible will create a file on the remote host with a name like this:
            /home/ubuntu/.ansible/tmp/ansible-tmp-1412459504.14-47728545618200/arguments
        # If we invoke the module like this:
            - name: check if host can reach the database server
              can_reach: host=db.example.com port=5432 timeout=1
        # Then the arguments file will have the following contents:
            host=db.example.com port=5432 timeout=1
        # We can tell Ansible to generate the arguments file for the module as JSON, by adding the following line to playbooks/library/can_reach:
            # WANT_JSON
        # If our module is configured for JSON input, the arguments file will look like this:
            {"host": "www.example.com", "port": "80", "timeout": "1"}
        ```
    - Invoke the Module
        ```
        # Ansible will call the module and pass the argument file as arguments. If it’s a Python- based module, Ansible executes the equivalent of the following (with /path/to/ replaced by the actual path):
            /path/to/can_reach
        # If it’s a non-Python-based module, Ansible will look at the first line of the module to determine the interpreter and execute the equivalent of:
            /path/to/interpreter /path/to/can_reach /path/to/arguments
        # Assuming the can_reach module is implemented as a Bash script and starts with:
            #!/bin/bash
        # Then Ansible will do something like:
            /bin/bash /path/to/can_reach /path/to/arguments
        # But even this isn’t strictly true. What Ansible actually does is:
            /bin/sh -c 'LANG=en_US.UTF-8 LC_CTYPE=en_US.UTF-8 /bin/bash /path/to/can_reach  /path/to/arguments; rm -rf /path/to/ >/dev/null 2>&1'
        # You can see the exact command that Ansible invokes by passing -vvv to ansible- playbook.
        ```

#### Expected Outputs
```
# Ansible expects modules to output JSON. For example:
    {'changed': false, 'failed': true, 'msg': 'could not reach the host'}
```
- Output Variables Ansible Expects    
    Your module can return whatever variables you like, but Ansible has special treat‐ ment for certain returned variables:
    - changed       
        `All Ansible modules should return a changed variable. The changed variable is a Boolean that indicates whether the module execution caused the host to change state. When Ansible runs, it will show in the output whether a state change has happened. If a task has a notify clause to notify a handler, the notification will fire only if changed is true.`
    - failed      
        `If the module failed to complete, it should return failed=true. Ansible will treat this task execution as a failure and will not run any further tasks against the host that failed, unless the task has an ignore_errors or failed_when clause.
If the module succeeds, you can either return failed=false or you can simply leave out the variable.`
    - msg 
        ```
        # Use the msg variable to add a descriptive message that describes the reason why a module failed.
        # If a task fails, and the module returns a msg variable, then Ansible will output that variable slightly differently than it does the other variables. 
        # If a module returns:
            {"failed": true, "msg": "could not reach www.example.com:81"} 
        # Then Ansible will output the following lines when executing this task:
            failed: [vagrant1] => {"failed": true}
            msg: could not reach www.example.com:81
        ```

####  Implementing Modules in Python
```
If you implement your custom module in Python, Ansible provides the AnsibleMod ule Python class that makes it easier to:
    Parse the inputs
    Return outputs in JSON format
    Invoke external programs
```
```
# We'll create our module in Python by creating a can_reach file.
# can_reach

#! /usr/bin/python
def  can_reach(module, host, port, timeout):

    # Gets the path of an external program
    nc_path = module.get_bin_path('nc', required=True)
    args = [nc_path, "-z", "-w", str(timeout), host, str(port)]

    # Invokes an external program
    (rc, stdout, stderr) = moduel.run_command(args)
    return rc == 0 

def main():

    # Instantiates the AnsibleModule helper class
    module = AnsibleModule(

        # Specifies the permitted set of arguments
        argument_spec=dict(
            host=dict(required=True), # A required argument
            host=dict(required=True, type='int'), 
            timeout=dict(required=False, type='int', default=3) # An optional argument with a default value
        ),
        supports_check_mode = True  # Specify that this module supports check mode 
    )

    # In check mode, we take no action . Since this module never changes system state, we just return changed=False
    if module.check_mode:   # Test to see if moduels is running in check mode 
        module.exit_json(changed=False)   # Exit successfully passing a return value

    host = module.params['host'] # Extract an argument 
    port = module.params['port']
    timeout = module.params['timeout']

    if can_reach(module, host, port, timeout):
        module.exit_json(changed=False)
    else:
        msg = "Could not reach %s:%s" % (host, port)
        module.fail_json(msg=msg)   # Exit with failure, passing an error message

    from ansible.module_utils.basic import *    # "Imports" the  AnsibleModule helper class
    main()
```

#### Argument Options
- required `If True, argument is required`    
- default `Default value if argument is not required`
- choices `A list of possible values for argument`
    ```
    # The choices option allows you to restrict the allowed arguments to a predefined list. Consider the distros argument in the following example:
        distro=dict(required=True, choices=['ubuntu', 'centos', 'fedora']) 
    # If the user were to pass an argument that was not in the list, for example:
        distro=suse
    # This would cause Ansible to throw an error.
    ```
- aliases `Other names you can use as an alias for this argument`
    ```
    # The aliases option allows you to use different names to refer to the same argument. For example, consider the package argument in the apt module:
        module = AnsibleModule(
            argument_spec=dict(
        ...
                package = dict(default=None, aliases=['pkg', 'name'], type='list'),
            )
        )
    # Since pkg and name are aliases for the package argument, these invocations are all equivalent:
        - apt: package=vim
        - apt: name=vim
        - apt: pkg=vim
    ```

- type `Argument type. Allowed values: 'str', 'list', 'dict', 'bool', 'int', 'float'`


#### AnsibleModule Initializer Parameters     
- argument_spec `Dictionary that contains information about arguments`
    ```
    default: (none)
    The AnsibleModule initializer method takes a number of arguments. The only required argument is argument_spec.    
    ```
- bypass_checks `If true, don’t check any of the parameter constrains`
    ```
    default: False
    ```
- no_log `If true, don’t log the behavior of this module`
    ```
    default: False
    When Ansible executes a module on a host, the module will log output to the syslog, which on Ubuntu is at /var/log/syslog.
    If a module accepts sensitive information as an argument, you might want to disable this logging.
    To configure a module so that it does not write to syslog, pass the no_log=True parameter to the AnsibleModule initializer.
    ```

- check_invalid_arguments `If true, return error if user passed an unknown argument`
    ```
    default: True
    ```

- mutually_exlusive `List of mutually exclusive arguments`
    ```
    default: None
    The mutually_exclusive parameter is a list of arguments that cannot be specified during the same module invocation.
    For example, the lineinfile module allows you to add a line to a file. You can use the insertbefore argument to specify which line it should appear before, or the insertafter argument to specify which line it should appear after, but you can’t specify both.
    Therefore, this module specifies that the two arguments are mutually exclusive, like this:
        mutually_exclusive=[['insertbefore', 'insertafter']]
    ```

- required_together `List of arguments that must appear together`
    ```
    default: None
    ```

- required_one_of `List of arguments where at least one must be present`
    ```
    default: None
    # The required_one_of parameter is a list of arguments where at least one must be passed to the module.
    # For example, the pip module, which is used for installing Python packages, can take either the name of a package or the name of a requirements file that contains a list of packages. The module specifies that one of these arguments is required like this:
        required_one_of=[['name', 'requirements']]
    ```

- add_file_common_args `Supports the arguments of the file module`
    ```
    default: False

    # Many modules create or modify a file. A user will often want to set some attributes on the resulting file, such as the owner, group, and file permissions.
You could invoke the file module to set these parameters, like this: 
        - name: download a file
          get_url: url=http://www.example.com/myfile.dat dest=/tmp/myfile.dat
        - name: set the permissions
          file: path=/tmp/myfile.dat owner=ubuntu mode=0600

    # As a shortcut, Ansible allows you to specify that a module will accept all of the same arguments as the file module, so you can simply set the file attributes by passing the relevant arguments to the module that created or modified the file. 
        - name: download a file
          get_url: url=http://www.example.com/myfile.dat dest=/tmp/myfile.dat \
          owner=ubuntu mode=0600
    # To specify that a module should support these arguments:
        add_file_common_args=True
    # The AnsibleModule module provides helper methods for working with these arguments.
    ```

- supports_check_mode `If true, indicates module supports check mode`
    ```
    default: False
    ```

#### Returning Success or Failure
```
# Use the exit_json method to return success. You should always return changed as an argument, and it’s good practice to return msg with a meaningful message:
    module = AnsibleModule(...)
    ...
    module.exit_json(changed=False, msg="meaningful message goes here")
# Use the fail_json method to indicate failure. You should always return a msg param‐ eter to explain to the user the reason for the failure:
    module = AnsibleModule(...)
    ...
    module.fail_json(msg="Out of disk space")
```

#### Check Mode (Dry Run)
```
# Ansible supports something called “check mode,” which is enabled when passing the -C or --check flag to ansible-playbook. It is similar to the “dry run” mode sup‐ ported by many other tools.
# When Ansible runs a playbook in check mode, it will not make any changes to the hosts when it runs. Instead, it will simply report whether each task would have changed the host, returned successfully without making a change, or returned an error.
# Modules must be explicitly configured to support check mode. If you’re going to write your own module, I recommend you support check mode so that your module is a good Ansible citizen.

# Telling Ansible the module supports check mode
    module = AnsibleModule(
        argument_spec=dict(...),
        supports_check_mode=True)

# Checking if check mode is enabled
    module = AnsibleModule(...) ...
    if module.check_mode:
       # check if this module would make any changes
       would_change = would_executing_this_module_change_something()
       module.exit_json(changed=would_change)

# It is up to you, the module author, to ensure that your module does not modify the state of the host when running in check mode.
```

#### 
```
Near the top of your module, define a string variable called DOCUMENTATION that con‐ tains the documentation, and a string variable called EXAMPLES that contains example usage.
It's shows an example for the documentation section for our can_reach module.

# Example of module documentation

DOCUMENTATION = '''
---
module: can_reach
short_description: Checks server reachability
description:
 - Checks if a remote server can be reached
version_added: "1.8"
options:
  host:
    description:
      - A DNS hostname or IP address
    required: true
  port:
    description:
    - The TCP port number
    required: true
  timeout:
    description:
    - The amount of time try to connecting before giving up, in seconds
    required: false
    default: 3
  flavor:
    description:
    - This is a made-up option to show how to specify choices.
    required: false
    choices: ["chocolate", "vanilla", "strawberry"]
    aliases: ["flavour"]
    default: chocolate
requirements: [netcat]
author: Lorin Hochstein
notes:
  - This is just an example to demonstrate how to write a module.
  - You probably want to use the native M(wait_for) module instead.
'''

EXAMPLES = '''
# Check that ssh is running, with the default timeout
- can_reach: host=myhost.example.com port=22
# Check if postgres is running, with a timeout
- can_reach: host=db.example.com port=5432 timeout=1
'''
```

#### Debugging Your Module
```
# Clone the Ansible repo:
    $ git clone https://github.com/ansible/ansible.git --recursive
# Set up your environment variables so that you can invoke the module:
    $ source ansible/hacking/env-setup Invoke your module:
    $ ansible/hacking/test-module -m /path/to/can_reach -a "host=example.com port=81" 
# Since example.com doesn’t have a service that listens on port 81, our module should fail with a meaningful error message. And it does:
    * including generated source, if any, saving to:
      /Users/lorinhochstein/.ansible_module_generated
    * this may offset any line numbers in tracebacks/debuggers!
    ***********************************
    RAW OUTPUT
    {"msg": "Could not reach example.com:81", "failed": true}
    ***********************************
    PARSED OUTPUT
    {
        "failed": true,
        "msg": "Could not reach example.com:81"
    }
# As the output suggests, when you run this test-module, Ansible will generate a Python script and copy it to ~/.ansible_module_generated. This is a standalone script that you can execute directly if you like. The debug script will replace the following line:
    from ansible.module_utils.basic import *
# with the contents of the file lib/ansible/module_utils/basic.py, which can be found in the Ansible repository.
# This file does not take any arguments; rather, Ansible inserts the arguments directly into the file:
    MODULE_ARGS = 'host=example.com port=91'
```

#### Implementing the Module in Bash
```

#you’ll need to install jq on the host before invoking this module. Example 10-9 shows the complete Bash implementation of our module.
# can_reach module in Bash
    #!/bin/bash
    # WANT_JSON
    # Read the variables form the file
    host=`jq -r .host < $1`
    port=`jq -r .port < $1`
    timeout=`jq -r .timeout < $1`
    # Check if we can reach the host
    nc -z -w $timeout $host $port
    # Output based on success or failure
    if[$?-eq0];then
    echo '{"changed": false}'
    else
    echo "{\"failed\": true, \"msg\": \"could not reach $host:$port\"}" fi
# We added WANT_JSON in a comment to tell Ansible that we want the input to be in JSON syntax.
```

#### Example Modules
The best way to learn how to write Ansible modules is to read the source code for the modules that ship with Ansible. Check them out on GitHub: [modules core](https://github.com/ansible/ansible-modules-core) and [modules extras](https://github.com/ansible/ansible-modules-extras).
