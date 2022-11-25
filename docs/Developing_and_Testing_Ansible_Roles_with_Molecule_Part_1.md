# Developing and Testing Ansible Roles with Molecule - Part 1

Notes based on: 
* [Developing and Testing Ansible Roles with Molecule and Podman - Part 1][molecule-and-podman-part-1]
* [Developing and Testing Ansible Roles with Molecule and Podman - Part 2][molecule-and-podman-part-2]

# Getting Started

1. Create a dedicated Python environment for our Molecule installation

```bash
$ mkdir molecule-blog
$ cd molecule-blog
$ python3 -m venv molecule-venv
$ source molecule-venv/bin/activate
(molecule-venv) $ pip install "molecule[lint]"

```
> **Note:** that installed Molecule with the `lint` option. By using this option, pip also installed the `yamllint` and `ansible-lint` tools that allow you to use Molecule to  perform static code analysis of your role, ensuring it complies with Ansible coding standards.

Installation downloads all of the dependencies including Ansible. Verify the installed version:

```bash
$ molecule --version
molecule 3.0.4
   ansible==2.9.10 python==3.6

```

# Initializing a New Ansible Role


> **Note:** By default, Molecule uses the Docker driver to execute tests. Want to execute tests using **podman**, need to specify the driver name using the option `--driver-name=podman` when initializing the role with molecule. 

> **Note:** [Molecule docker driver missing error][molecule-docker-error] need to install `pip install molecule-docker`



Initialize the new role `mywebapp` with this command: 

## Creating a new role

Example: `$ molecule init role acme.my_new_role --driver-name docker`

```bash
$   molecule init role -d docker webrole.webapp
INFO     Initializing new role webapp...
Using /etc/ansible/ansible.cfg as config file
- Role webapp was created successfully
localhost | CHANGED => {"backup": "","changed": true,"msg": "line added"}
INFO     Initialized role in /mnt/c/Users/rnall/Desktop/repos/IaC/Ansible/molecule-blog/webapp successfully.

```

Created the structure for new role in a directory named `mywebapp`

```bash
.
├── defaults
│   └── main.yml
├── files
├── handlers
│   └── main.yml
├── meta
│   └── main.yml
├── molecule
│   └── default
│       ├── converge.yml
│       ├── INSTALL.rst
│       ├── molecule.yml
│       └── verify.yml
├── README.md
├── tasks
│   └── main.yml
├── templates
├── tests
│   ├── inventory
│   └── test.yml
└── vars
    └── main.yml
```

Molecule includes its configuration files under the `molecule` subdirectory. When initializing a new role, Molecule adds a single scenario named `default`.

Verify the basic configuration in the file `molecule/default/molecule.yml`:

```bash
---
dependency:
  name: galaxy
driver:
  name: podman
platforms:
  - name: instance
    image: docker.io/pycontribs/centos:8
    pre_build_image: true
provisioner:
  name: ansible
verifier:
  name: ansible
```

Default platform `instance` using the container image `docker.io/pycontribs/centos:8` that is changed later.

Include the lint configuration at the end:

`$ vi molecule/default/molecule.yml`

```yml
...
verifier:
  name: ansible
lint: |
  set -e
  yamllint .
  ansible-lint .
``` 

 Run `molecule lint` from the project root to lint the entire project:

`$ molecule lint`

> **Note:** Need to install `pip install ansible-lint` **CRITICAL Lint failed with error code 127**

The command returns a few errors because the file `meta/main.yml` is missing some required values. Fix these issues by editing the file `meta/main.yml` and adding `author`, `company`, `license`, `platforms`, and removing the blank line at the end. Without comments - for brevity - the `meta/main.yaml` looks like this:


```yml
galaxy_info:
  author: Ricardo Gerardi
  description: Mywebapp role deploys a sample web app 
  company: Red Hat 
 
  license: MIT 
 
  min_ansible_version: 2.9
 
  platforms:
  - name: rhel
    versions:
    - 8 
  - name: ubuntu
    versions:
    - 20.04
 
  galaxy_tags: []
 
dependencies: []
```

Re-lint the project and verify that there are no errors this time.

`$ molecule lint`

"Basic molecule configuration is in place"

# Setting up Instances

By default, Molecule defines a single instance named `instance` using the `Centos:8` image. According to our requirements, should ensure role works with RHEL 8 and Ubuntu 20.04. This role starts the Apache web server as a system service and need to use container images that enable `systemd`.

Red Hat provides an official Universal Base Image for RHEL 8, which enables `systemd`: 

`registry.access.redhat.com/ubi8/ubi-init`

For Ubuntu, there’s no official `systemd` enabled images so need to use an image maintained by **Jeff Geerling** from the Ansible open-source community:

`geerlingguy/docker-ubuntu2004-ansible`

To enable the `systemd` instances, modify the `molecule/default/molecule.yml` configuration file, remove the `centos:8` instance and add the two new instances.

`$ vi molecule/default/molecule.yml`


```yml
---
dependency:
  name: galaxy
driver:
  name: podman
platforms:
  - name: rhel8
    image: registry.access.redhat.com/ubi8/ubi-init
    tmpfs:
      - /run
      - /tmp
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:ro
    capabilities:
      - SYS_ADMIN
    command: "/usr/sbin/init"
    pre_build_image: true
  - name: ubuntu
    image: geerlingguy/docker-ubuntu2004-ansible
    tmpfs:
      - /run
      - /tmp
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:ro
    capabilities:
      - SYS_ADMIN
    command: "/lib/systemd/systemd"
    pre_build_image: true
provisioner:
  name: ansible
verifier:
  name: ansible
lint: |
  set -e
  yamllint .
  ansible-lint .

```

Mounting the temporary filesystem `/run` and `/tmp`, as well as the `cgroup` volume for each instance. Enabling the `SYS_ADMIN` capability, as they are required to run a container with Systemd.

If following this on a RHEL 8 machine with SELinux enabled. Set the `container_manage_cgroup` boolean to true to allow containers to run Systemd. 

`sudo setsebool -P container_manage_cgroup 1`


Molecule uses an Ansible Playbook to provision these instances.Modify and add parameters for provisioning by modifying the `provisioner` dictionary in the `molecule/default/molecule.yml` configuration file. It accepts the same configuration options provided in an Ansible configuration file `ansible.cfg`.

 > For example, update the provisioner configuration by adding a `defaults` section. Set the Python interpreter to `auto_silent` to prevent warnings. Enable the `profile_tasks`, `timer`, and `yaml` callback plugins to output profiling information with the playbook output. 
 
 > **Note:** Then, add the `ssh_connection` section and disable SSH pipelining because it does not work with Podman.

```yml
provisioner:
  name: ansible
  config_options:
    defaults:
      interpreter_python: auto_silent
      callback_whitelist: profile_tasks, timer, yaml
    ssh_connection:
      pipelining: false
```

Save the configuration file and create the instances by running `molecule create` from the role root directory:


`$ molecule create`

Molecule runs the provisioning playbook and creates both instances. Check the instances by running `molecule list`:


```bash 
INFO     Running default > list
                ╷             ╷                  ╷               ╷         ╷
  Instance Name │ Driver Name │ Provisioner Name │ Scenario Name │ Created │ Converged
╶───────────────┼─────────────┼──────────────────┼───────────────┼─────────┼───────────╴
  rhel8         │ docker      │ ansible          │ default       │ true    │ false
  ubuntu        │ docker      │ ansible          │ default       │ true    │ false
                ╵             ╵                  ╵               ╵         ╵
```

 In case a test fails, or an error causes an irreversible change that requires you to start over, delete these instances by running `molecule destroy` and recreate them with `molecule create` at any time.




[//]: Links
[molecule-and-podman-part-1]: https://www.ansible.com/blog/developing-and-testing-ansible-roles-with-molecule-and-podman-part-1
[molecule-docker-error]: https://spatial-labs.dev/posts/202104241553-molecule-docker-error
[molecule-and-podman-part-2]: https://www.ansible.com/blog/developing-and-testing-ansible-roles-with-molecule-and-podman-part-2