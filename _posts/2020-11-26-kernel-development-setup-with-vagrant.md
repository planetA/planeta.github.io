---
title: Kernel development setup with Vagrant
date: 2020-11-26 23:02
categories: Programming
tags: linux kernel
---

Developing the Linux kernel is cumbersome, starting from the very beginning.
Typical development cycle of writing, compiling, deploying[^1], testing,
debugging, and again writing is hard for any large project, but the kernel
definitely stands out.

In this article, I want to describe a simple development setup that uses Vagrant
and Ansible to setup two virtual machines to quickly test out code changes to
the Linux kernel. This setup can be used by novice kernel developers to dive
into the actual process quicker.

Vagrant manages virtual machines similarly to how docker-machine manages
containers. Instead of creating VMs and installing the OS manually, one can use
Vagrant to fetch and instantiate pre-made images. Vagrant will setup network,
provide hostnames, and create ssh tunnels into the machines. All of this happens
with few lines of a configuration file.

As a second step, Ansible will configure the VMs by installing additional
packages, upgrading the kernel and kernel modules. In comparison to normal
bash-over-ssh, Ansible offers clear structure, concise syntax, error handling
and caching the results. The last one, I would consider as a killer feature.

Of course, the same features can be implemented in bash, but the programmer
would need to put in own work. Bash is not that great with code reuse. To ensure
that expensive commands do not rerun needlessly, I often end up with a bunch of
scripts for different "stages" of the deployment process. Then, I manually
choose a stage I want to rerun. With Ansible, I have a single entry point
starting from the moment when Vagrants instantiates the system image.

In this article, I target my own usage scenario, although it can be adapted to
other uses with little effort. Specifically, I develop a kernel-level driver and
user-level library runtime, both related to RDMA-networks. The libraries are
used by a benchmark, which I install from github as part of workflow.

The goal of this article is enable setup, where one can instantiate virtual
machines from scratch using one command, namely:

```bash
vagrant up --provision
```

Connecting to the VMs is also simple:

```bash
vagrant ssh <machine-name>
```

The repo with the configuration is available on
[github](https://github.com/planetA/kernel-vargrant).

# Prerequisites

For this setup to work, one needs to install Vagrant, Ansible, and QEMU. For my
Debian testing installation, these included (but not limited to) following
packages:

 - vagrant
 - vagrant-libvirt
 - libvirt-daemon
 - ansible
 - qemu-system-x86
 
Ansible requires additional plugins:

```bash
ansible-galaxy collection install ansible.posix
ansible-galaxy collection install community.general
```

I maintain source code of the Linux kernel and RDMA-libraries repository outside
of this setup. I provide the locations of the repositories in the configuration.

As an optional step, I create a libvirt storage pool in the home partition,
because my root directory has very limited capacity. Libvirt is a lower-level
(in comparison to Vagrant) virtualisation toolset to manage VMs and containers.
Roughly libvirt relates to Vagrant, like docker to docker-machine.

I created the pool as follows:

```bash
mkdir -p /home/libvirt/images/
chmod 0711 /home/libvirt/images/
virsh pool-define-as --name home --type dir --target /home/libvirt/images/
virsh pool-start home
```

# Vagrant setup

[Vagrant](https://www.vagrantup.com/intro) setup is very simple. After
installing vagrant, initialise a new project in a directory of you choice:

```bash
vagrant init
```

This will create a configuration file, that I update as follows:

```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|

  config.vm.provider :libvirt do |libvirt|
    libvirt.storage_pool_name = "home"
  end
  config.vm.box = "generic/debian10"

  # Disable automatic box update checking. If you disable this, then
  # boxes will only be checked for updates when the user runs
  # `vagrant box outdated`. This is not recommended.
  config.vm.box_check_update = false

  N = 2
  (1..N).each do |machine_id|
    config.vm.define "nadu#{machine_id}" do |machine|
      machine.vm.hostname = "nadu#{machine_id}"
      machine.vm.network "private_network", ip: "192.168.33.#{10+machine_id}"

      if machine_id == N

        machine.vm.provision :ansible do |ansible|
          ansible.limit = "all"
          ansible.groups = {
            "master" => ["nadu1"],
            "workers" => ["nadu[1:#{N}]"]
          }
          ansible.playbook = "provisioning/install.yaml"
        end
      end
    end
  end
end
```

Some explanation to config. I set the provider to libvirt, so that Vagrant uses
KVM over libvirt, instead of VirtualBox. Oracle seems not to maintain a
VirtualBox package for my Debian/Testing and the old package does not work
smoothly. I explicitly request libvirt to use a specific storage pool "home".

Then, I set the name of the image I want to use. This can be any image from
Vagrant's [website](https://app.vagrantup.com/boxes/search). Pay attention, that
the image you choose supports the virtualisation method specified before,
libvirt in my case. Also, I disable automatic updates. This option comes with
the default Vagrant configuration file. I thought it makes sense.

Finally, the config instantiates the virtual machines in a loop. The loop
assigns hostnames, configures network and calls Ansible. Option "limit" allows
ansible in parallel on different VMs. Option "group" declares ansible groups.

You still need to setup ansible playbook, but already now you can start the VMs
using this command:

```bash
vagrant up
# ... or also to trigger ansible
vagrant up --provision
```

And connect to the VMs using this command:

```bash
vagrant ssh nadu1
```

# Kernel code

There are actually multiple ways to compile kernel code: out-of-tree, preparing
a deb- or rpm-package, or full kernel compilation. I want to build upon a recent
kernel (vanilla master) and have possibility to modify in-tree modules, and even
compiled-in code. Moreover building a deb-package requires the kernel repository
to be absolutely clean of any files that are not part of the repo. This
requirement was quite annoying for me, so I chose to build the kernel normally
(just make).

Now, I need to decide how to deploy the compiled code into the VMs. It would be
too slow to deploy code changes inside the VMs and compile code there.
Additionally, this would result into work duplication. So, I compile the kernel
on my host system, and then send changes to the VM.

Given that introduction, here is how I compile the kernel:

```bash
# Switch to my branch
git rebase master
# Resolve the conflicts
make oldconfig
make menuconfig
make -j$(nproc)

export INSTALL_PATH=$(readlink -f ~/<provisioning>/roles/rdma_install/files/kernel/boot)
export INSTALL_MOD_PATH=$(readlink -f ~/<provisioning>/roles/rdma_install/files/kernel)

mkdir -p $INSTALL_PATH
make install
make modules_install 
```

First, I update config file to the kernel version, then I, potentially, set new
options I need, and compile the kernel. After compilation finished, I install
the kernel and the modules into `../linux-install/`.

I could save mod time of the kernel, and then check it after recompilation. If
the time after make changes, then the kernel has been updated, so I need to
reinstall it.

# Ansible

[Ansible](https://docs.ansible.com/ansible/latest/index.html) is an automation
tool, that on surface replaces bash-over-ssh. In comparison to bash-over-ssh,
Ansible offers enough convenience feature to make it worthwhile to learn a new
tool. In its simplest form, Ansible will execute configuration following a
YAML-file provided by the user.

To integrate Ansible with Vagrant, I create playbook "install.yaml" inside
directory "provisioning". Additionally, for my example, I use feature of Ansible
called "roles". Roles allow to organise common functionality into a dedicated
directory structured in a predefined way. In short, Ansible roles work as
libraries for programming languages. As results, the overall structure of the
provisioning directory looks as follows:

```
provisioning
├── config.yaml
├── install.yaml
└── roles
    ├── kernel_install
    │   ├── files
    │   │   ├── kernel
    │   │   │   ├── boot
    │   │   │   ├── lib
    │   │   │   └── usr
    │   │   └── ssh
    │   │       ├── config
    │   │       ├── id_rsa
    │   │       └── id_rsa.pub
    │   ├── tasks
    │   │   └── main.yml
    │   └── templates
    │       └── hosts.j2
    └── rdma_install
        ├── defaults
        │   └── main.yml
        └── tasks
            └── main.yml
```

The topmost files inside the provisioning directory describe the playbook. File
"config.yaml" references the repositories I use for development:

```yaml
---
kernel_src: ~/src/linux-2.6
rdma_core_src: ~/src/rdma-core
```

Vagrant directly calls playbook file "install.yaml":

```yaml
{%raw%}
---
- hosts: master
  connection: local
  vars_files: "{{ playbook_dir }}/config.yaml"
  tasks:
    - name: Install kernel locally 
      command:
        argv:
          - make
          - -j5
          - install
      args:
        chdir: "{{ kernel_src }}"
      delegate_to: localhost
      environment:
        INSTALL_PATH: "{{ playbook_dir  }}/roles/kernel_install/files/kernel/boot"
    - name: Install kernel modules locally 
      command:
        argv:
          - make
          - -j5
          - modules_install
      args:
        chdir: "{{ kernel_src }}"
      delegate_to: localhost
      environment:
        INSTALL_PATH: "{{playbook_dir}}/roles/kernel_install/files/kernel/boot"
        INSTALL_MOD_PATH: "{{playbook_dir}}/roles/kernel_install/files/kernel/"
    - name: Install headers
      command:
        argv:
          - make
          - "INSTALL_HDR_PATH={{ playbook_dir  }}/roles/kernel_install/files/kernel/usr"
          - headers_install
      args:
        chdir: "{{ kernel_src }}"
      delegate_to: localhost
      environment:
        INSTALL_PATH: "{{playbook_dir }}/roles/kernel_install/files/kernel/boot"
        INSTALL_MOD_PATH: "{{playbook_dir}}/roles/kernel_install/files/kernel/"
- hosts: workers
  vars_files: "{{ playbook_dir }}/config.yaml"
  roles:
    - kernel_install
    - rdma_install
- hosts: nadu1
  tasks:
    - name: Start server
      command: ib_send_bw -b
      async: 120
      poll: 0
      register: send_bw_srv
- hosts: nadu2
  tasks:
    - name: Start client
      command: ib_send_bw -b nadu1
- hosts: nadu1
  tasks:
    - name: Check back the server
      async_status:
        jid: "{{ send_bw_srv.ansible_job_id }}"
      register: srv_result
      until: srv_result.finished
      retries: 10
      delay: 5
{%endraw%}
```

Describing syntax of Ansible is out of scope of this article, instead I only
describe what the tasks are doing.

I assume that the Linux kernel is already compiled and do not try to recompile
it. Instead, I start by "installing" Linux kernel, kernel modules, and headers
into the "files" directory of the "kernel_install" role. Later, this role uses
the installed files to move them into the VMs.

I compile the Linux kernel on the host system, because that Linux kernel is a
large project that takes a lot of time to compile, especially inside a VM.
Moreover, I would need to the same large task twice, because I have two VMs.
Luckily, compiling the Linux kernel is very easy to do on another platform: it
literally has no dependencies, except the CPU architecture.

In contrast, rdma-core libraries are much smaller and have dependencies that
cannot be ignored. Therefore, I decided, it is easier to compile the rdma-core
libraries directly in VMs, without too much loss of performance.

After preparing the kernel, I let the roles run on each of the VMs. Finally, I
have a series of tasks to run the simplest possible test to check if RDMA
communication over SoftRoCE still works.

Next runs "kernel_install" role. The role starts by connecting newly created

As a first step, the role add corresponding entries into the hosts file:

```yaml
- name: Install hostfile
  become: true
  template:
    src: hosts.j2
    dest: /etc/hosts
    mode: '0644'
    backup: true
```

The host file is filled from a template:

```
127.0.0.1	localhost
127.0.1.1	{% raw %}{{ inventory_hostname }}	{{ inventory_hostname }} {% endraw %}

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters

192.168.33.11 nadu1
192.168.33.12 nadu2
192.168.33.13 nadu3
192.168.33.14 nadu4
```

Next, the role install ssh keys:

```yaml
{%raw%}
- name: Copy private ssh and config
  copy:
    src: ssh/
    dest: .ssh
    mode: '0600'
- name: Enable ssh keys
  authorized_key:
    user: vagrant
    key: '{{ item }}'
    state: present
  with_file:
    - ssh/id_rsa.pub
{%endraw%}
```

Doing this on a production system would be terribly insecure, but these are VMs
that are accessible only from my laptop. And uploading private keys would be
even worse, so these are the keys that I specifically generated before and do
not use anywhere else.

Next I install kernel and kernel modules:

```yaml
- name: Install kernel
  become: true
  ansible.posix.synchronize:
    src: kernel/boot/
    dest: /boot/
    checksum: true
  register: kernel
- name: Install kernel headers
  become: true
  ansible.posix.synchronize:
    src: kernel/usr/include/
    dest: /usr/include/
- name: Install kernel modules
  become: true
  ansible.posix.synchronize:
    src: kernel/lib/modules/
    dest: /lib/modules/
    checksum: true
  register: modules
- name: Display modules result
  debug:
      var: modules.stdout_lines
```

If the kernel actually has been changed, I update GRUB and reboot the system:

```yaml
- name: Update GRUB
  become: true
  command: update-grub
  when: kernel.changed or modules.changed
  register: grub
- name: Reboot
  become: true
  shell: /sbin/shutdown -r now 'Rebooting box to update system libs/kernel as needed'
  async: 1
  poll: 1
  ignore_errors: true
  when: grub.changed
- name: Wait for system to become reachable again
  wait_for_connection:
      delay: 1
      timeout: 60
- name: Verify new update (optional)
  command: uname -mrs
  register: uname_result
- name: Display new kernel version
  debug:
      var: uname_result.stdout_lines
```

Now, it is the time to install RDMA userspace. This role starts by installing
the required packages.

```yaml
{%raw%}
- name: Install all required dev packages
  become: true
  apt:
    pkg:
      - libnl-route-3-dev
      - pandoc
      - docutils-common
      - iproute2
      - libmnl-dev
      - lldb
      - strace
      - ltrace
- name: Download iproute2
  unarchive:
    src: "{{ iproute2_url | quote }}"
    dest: "{{ home_dir }}"
    remote_src: yes
- name: Compile iproute2
  shell: "./configure && make -j2"
  args:
    chdir: "{{home_dir}}/{{iproute2_version}}"
    creates: "{{home_dir}}/{{iproute2_version}}/rdma/rdma"
- name: Install iproute2
  become: true
  shell: "make install && touch installed || rm installed"
  args:
    chdir: "{{home_dir}}/{{iproute2_version}}"
    creates: "{{home_dir}}/{{iproute2_version}}/installed"
{%endraw%}
```

I can install most of the packages from the repository, but I need a newer
version of iproute2 than the one that is provided by Debian/stable.

Next, I copy, compile and install rdma-core repository I am working on. When
copying the repo into the VM, I want to ignore build and git directories. Before
compiling, I invoke CMake, then install the libraries system-wide.

```yaml
{%raw%}
- name: Sync rdma-core repo
  ansible.posix.synchronize:
    src: "{{rdma_core_src}}/"
    dest: "{{rdma_core_src_vm}}"
    delete: true
    rsync_opts:
      - "--exclude=.git"
      - "--exclude=build"
- name: Create build directory
  file:
    path: "{{rdma_core_src_vm}}/build"
    state: directory
- name: Run cmake
  command:
    argv:
      - cmake
      - -DCMAKE_INSTALL_PREFIX=/usr
      - ..
  args:
    chdir: "{{rdma_core_src_vm}}/build"
    # creates: "{{rdma_core_src_vm}}/build/Makefile"
- name: Compile rdma-core
  command:
    argv:
      - make
      - -j5
  args:
    chdir: "{{ rdma_core_src_vm }}/build"
- name: Install rdma-core
  become: true
  command:
    argv:
      - make
      - install
  args:
    chdir: "{{ rdma_core_src_vm }}/build"
{%endraw%}
```

After that I load SoftRoCE and SoftiWarp drivers, although, I did not find a way
how to make SoftiWarp drivers work. Here, I call rdma tool form iproute2 package
I installed before.

```yaml
{%raw%}
- name: Load drivers
  become: true
  community.general.modprobe:
    name: rdma_rxe
    state: present
- name: Add SoftRoCE device
  become: true
  shell: /usr/sbin/rdma link show rxe0/1 || /usr/sbin/rdma link add rxe0 type rxe netdev eth1
- name: Add SoftiWarp device
  become: true
  shell: /usr/sbin/rdma link show siw0/1 || /usr/sbin/rdma link add siw0 type siw netdev eth1
{%endraw%}
```

Finally, I install RDMA performance tools. This time, I use autotools for configuration.

```yaml
{%raw%}
- name: Get perftest tools
  unarchive:
    src: "{{ perftest_url | quote }}"
    dest: "{{home_dir}}"
    remote_src: yes
- name: Configure perftest
  shell: "./autogen.sh && ./configure"
  args:
    chdir: "{{home_dir}}/{{perftest_dir}}"
    creates: "{{home_dir}}/{{perftest_dir}}/Makefile"
- name: Compile perftest
  shell: "make -j"
  args:
    chdir: "{{home_dir}}/{{perftest_dir}}"
    creates: "{{home_dir}}/{{perftest_dir}}/ib_send_bw"
- name: Install perftest
  become: true
  shell: "make install"
  args:
    chdir: "{{home_dir}}/{{perftest_dir}}"
    creates: "/usr/bin/ib_send_bw"
{%endraw%}
```

I keep all the sources in home directory, instead of /tmp, to keep these
packages through reboots.

# Conclusion

The main advantage of this setup, is that I can build the whole system
practically with one command. Additionally, now the whole setup is automatised
enough, so that I could, for example, run automatic git bisect.

Although, other usage scenarios will require to modify the scripts, this setup
can be a starting point for creating other kernel development flows. For
example, in future, I plan to reuse Ansible configuration to deploy and test
driver prototypes on real kernels.

I will be happy for any feedback to my email.


[^1]: For some projects, deployiment during development is basically a nop.
