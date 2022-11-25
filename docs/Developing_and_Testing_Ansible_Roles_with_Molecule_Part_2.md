# Developing and Testing Ansible Roles with Molecule - Part 2

Notes based on: 
* [Developing and Testing Ansible Roles with Molecule and Podman - Part 1][molecule-and-podman-part-1]
* [Developing and Testing Ansible Roles with Molecule and Podman - Part 2][molecule-and-podman-part-2]


This basic role deploys a web application supported by the Apache web server. It must support Red Hat Enterprise Linux (RHEL) 8 and Ubuntu 20.04.

# Developing the Ansible Role with Molecule

Apache package `httpd` using the `package` Ansible module. Edit the file `tasks/main.yaml` and include this task:

```bash
---
# tasks file for mywebapp
- name: Ensure httpd installed
  package:
    name: "httpd"
    state: present
```

Save the file and `converge` the instances by running `molecule converge`. The “converge” command applies the current version of the role to all the running container instances. Molecule `converge` does not restart the instances if they are already running. It tries to converge those instances by making their configuration match the desired state described by the role currently testing.

```bash
$ molecule converge

... TRUNCATED OUTPUT ... 
   TASK [mywebapp : Ensure httpd installed] ***************************************
    *********
fatal: [ubuntu]: FAILED! => {"changed": false, "msg": "No package matching 'httpd' is available"}
    changed: [rhel8]
... TRUNCATED OUTPUT ... 

```

Notice that the current version worked well on the RHEL8 instance, but failed for the Ubuntu instance. The tasks failed because Ubuntu does not have a package named `httpd`. For that platform, the package name is `apache2`.

Modify the role to include variables with the correct package name for each platform. Start with RHEL8 by adding a file `RedHat.yaml` under the `vars` sub-directory with this content:

```yml
---
httpd_package: httpd
```

`vars/Debian.yaml` for Ubuntu:

```yml
---
httpd_package: apache2
```

`tasks/main.yaml` file to include these variable files according to the OS family identified by Ansible via the [system fact variable][playbooks_vars_facts] `ansible_os_family`.  Have to include a task to update the package cache for systems in the “Debian” family since their package manager caches. Update the install task to use the variable `httpd_package` that you defined in the variables files:


```yml
- name: Include OS-specific variables.
  include_vars: "{{ ansible_os_family }}.yaml"
 
- name: Update Package Cache (apt/Ubuntu)
  apt:
    update_cache: yes
  changed_when: false
  when: ansible_os_family == "Debian"
 
- name: Ensure httpd installed
  package:
    name: "{{ httpd_package }}"
    state: present
```

```bash
$ molecule converge
... TRUNCATED OUTPUT ... 
   TASK [mywebapp : Ensure httpd installed] ***************************************
    *********
    ok: [rhel8]
    changed: [ubuntu]
... TRUNCATED OUTPUT ...

```

Package was already installed in the RHEL8 instance, Ansible returned the status `OK` and it did not make any changes. It installed the package correctly in the Ubuntu instance this time.

> **Note:** Issues with `systemd` on WSL. *Error:* [System has not been booted with systemd as init system (PID 1)][cant-operate] images need to be built with `systemd` enabled. 
>
> [Testing your Ansible roles with Molecule][ansible-roles-molecule]: use `volumes` to mount `/sys/fs/cgroup` inside the image, and set the `--privileged` flag, because without these, `systemd` will not run correctly inside the container. 
>
> **Fixed:** using `sysvinit:` instead of `service:` in tasks/main.yml. 

```bash
- name: Ensure {{httpd_service}} svc is started
  sysvinit:
      name: "{{ httpd_service }}"
      state: started
      enabled: yes
  when: ansible_os_family == "Debian"
```


Add service name variables to the playbooks and variable files. Start with RHEL8:

```yml
---
httpd_package: httpd
httpd_service: httpd
```

edit the file “vars/Debian.yaml” for Ubuntu:

```yml
---
httpd_package: apache2
httpd_service: apache2
```

 At the end of the `tasks/main.yml` file:

```yml
- name: Ensure {{httpd_service}} svc is started
  service:
    name: "{{ httpd_service }}"
    state: started
    enabled: yes
  when: ansible_os_family == "RedHat"  

- name: Ensure {{httpd_service}} svc is started
  sysvinit:
      name: "{{ httpd_service }}"
      state: started
      enabled: yes
  when: ansible_os_family == "Debian"
```

```bash
$ molecule converge
... TRUNCATED OUTPUT ...    
   TASK [mywebapp : Ensure httpd svc started] *************************************
    *********
    changed: [ubuntu]
    changed: [rhel8]
... TRUNCATED OUTPUT ...
```

Each platform requires the HTML files owned by different groups. Add new variables to each variable file to define the group name `vars/RedHat.yaml`:

```yml
---
httpd_package: httpd
httpd_service: httpd
httpd_group: apache
```

`vars/Debian.yaml` for Ubuntu:

```yml
---
httpd_package: apache2
httpd_service: apache2
httpd_group: www-data
```

Dnd of the `tasks/main.yml` file:

```yml
- name: Ensure HTML Index
  copy:
    dest: /var/www/html/index.html
    mode: 0644
    owner: root
    group: "{{ httpd_group }}"
    content: "{{ web_content }}"
```

Add a default `web_content` value to this variable in case the user `defaults/main.yml`:

```yml
---
# defaults file for mywebapp
web_content: There's a web server here
```

`converge` the instances one more time to add the content:


```bash
$ molecule converge
... TRUNCATED OUTPUT ... 
   TASK [mywebapp : Ensure HTML Index] ********************************************
    *********
    changed: [rhel8]
    changed: [ubuntu]
... TRUNCATED OUTPUT ...
```

Manually verify that the role worked by using the molecule login command to log into one of the instances and running the `curl` command to get the content:


```bash
$ molecule login -h rhel8
[root@2ce0a0ea8692 /]# curl http://localhost
There's a web server here 
[root@2ce0a0ea8692 /]# exit
```

# Verifying the Role with Molecule

Basic verifier playbook `molecule/default/verify.yml` as a starting point. Ansible’s `uri` module to obtain the content from the running web server and the “assert” module to ensure it’s the correct content:

Verify the results by running `molecule verify`:

```bash
$ molecule verify
... TRUNCATED OUTPUT ... 
   TASK [Ensure content type is text/html] ****************************************
    Saturday 27 June 2020  10:03:18 -0400 (0:00:03.131)       0:00:07.255 *********
    ok: [rhel8] => {
        "changed": false,
        "msg": "All assertions passed"
    }
    ok: [ubuntu] => {
        "changed": false,
        "msg": "All assertions passed"
    }
... TRUNCATED OUTPUT ... 
Verifier completed successfully.
```

Change the default values for the test by editing the converge playbook to update the `web_content` variable:

```yml
---
- name: Converge
  hosts: all
  tasks:
    - name: "Include httpd.webapp"
      include_role:
        name: "httpd.webapp"
      vars:
         web_content: "New content for testing only"
```


Update the `expected_content` variable in the verifier playbook:


```yml
---
# This is an example playbook to execute Ansible tests.
 
- name: Verify
  hosts: all
  vars:
    expected_content: "New content for testing only"
  tasks:
```

**Converge** the instances one more time to update the web server content, then **verify** the results:


```bash
$ molecule converge
... TRUNCATED OUTPUT ... 
   TASK [mywebapp : Ensure HTML Index] ********************************************
    *********
    changed: [rhel8]
    changed: [ubuntu]
... TRUNCATED OUTPUT ... 

$ molecule verify
... TRUNCATED OUTPUT ... 
   TASK [Debug results] ***********************************************************
    *********
    ok: [rhel8] => {
        "this.content": "New content for testing only"
    }
    ok: [ubuntu] => {
        "this.content": "New content for testing only"
    }
... TRUNCATED OUTPUT ... 
Verifier completed successfully.

```

# Automating the Complete Test Workflow

`molecule converge`,  aides with role development, the goal with `molecule test` is to provide an automated and repeatable environment to ensure the role works according to its specifications. Therefore, the test process destroys and re-creates the instances for every test.

Default, `molecule test` executes these steps in order:

1. Install required dependencies
2. Lint the project
3. Destroy existing instances
4. Run a syntax check
5. Create instances
6. Prepare instances (if required)
7. Converge instances by applying the role tasks
8. Check the role for idempotence
9. Verify the results using the defined verifier
10. Destroy the instances

Change these steps by adding the `test_sequence` dictionary with the required steps to the Molecule configuration file. For additional information, consult the official [documentation][scenario].

Execute the test scenario:

```bash
$ molecule test
... TRUNCATED OUTPUT ... 
--> Test matrix
    
└── default
    ├── dependency
    ├── lint
    ├── cleanup
    ├── destroy
    ├── syntax
    ├── create
    ├── prepare
    ├── converge
    ├── idempotence
    ├── side_effect
    ├── verify
    ├── cleanup
    └── destroy
    
--> Scenario: 'default'
... TRUNCATED OUTPUT ... 
```

If the test workflow fails at any point, the command returns a status code different than zero. You can use that return code to automate the process or integrate Molecule with CI/CD workflows.



[//]: Links
[molecule-and-podman-part-1]: https://www.ansible.com/blog/developing-and-testing-ansible-roles-with-molecule-and-podman-part-1
[molecule-docker-error]: https://spatial-labs.dev/posts/202104241553-molecule-docker-error
[molecule-and-podman-part-2]: https://www.ansible.com/blog/developing-and-testing-ansible-roles-with-molecule-and-podman-part-2
[playbooks_vars_facts]: https://docs.ansible.com/ansible/devel/playbook_guide/playbooks_vars_facts.html
[cant-operate]: https://askubuntu.com/questions/1379425/system-has-not-been-booted-with-systemd-as-init-system-pid-1-cant-operate
[ansible-roles-molecule]: https://www.jeffgeerling.com/blog/2018/testing-your-ansible-roles-molecule

[scenario]: https://molecule.readthedocs.io/en/latest/configuration.html#scenario