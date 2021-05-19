---
title: "Installing K3s with embedded etcd on RaspberryPi 4"
date: 2021-05-19
draft: false

tags: [k8s]
catergories: []
keywords: []
---

Today we will use [`k3sup`](https://github.com/alexellis/k3sup) to install k3s onto our Raspberry Pi 4s. To install this tool on macOS we can simply use Homebrew:

```shell
brew install k3sup
```

Next, we will use [Balena Etcher](https://etcherpc.com/). This is a handy tool for flashing images to usb devices and such. It's avaliable on Homebrew under the cask `balenaetcher`.

```shell
brew install balenaetcher
```

To install Ubuntu on our Raspberry Pi, head over to the [Ubuntu Raspberry Pi downloads page](https://ubuntu.com/download/raspberry-pi), and grab the latest `.iso`. Plug your SD card into your computer, start up Etcher, and follow the instructions to flash the image onto your SD card. 

The next thing we want to do is to update the `user-data`. This manages the operation of [`cloud-init`](https://cloudinit.readthedocs.io/en/latest/index.html), which creates your user, with appropriate SSH `authorized_keys` and such on the first boot. My file looks like this:

```yaml
#cloud-config
hostname: pixie0
ssh_pwauth: false

users:
- name: k3s
  ssh_import_id:
  - gh:jmmaloney4
  sudo: ALL=(ALL) NOPASSWD:ALL
  shell: /bin/bash
  lock_passwd: true
```

This disables password-authenticated ssh, and creates one user named `k3s` with ssh `authorized_keys` from the GitHub account `jmmaloney4`, and passwordless sudo permissions. There are lots of other cloud-init options, but these are the basics needed to get an installation of Ubuntu up and running.

After modifying the `user-data` file on the root of the newly-flashed SD card, eject it, insert it into your Raspberry Pi and boot it up. It may take a few minutes for your ssh keys to get copied to `authorized_keys`, so don't be alarmed if you cannot immediately log into your pi.

Next we need to configure the machine a little bit. We will use `ansible` to do this. The first task is to fix the kernel cmdline to enable some features required by Kubernetes. 

```yaml
- name: Fix RaspberryPi cmdline
  hosts: all
  become: true
  tasks:
  - name: Fix cmdline
    register: fix_cmdline
    ansible.builtin.command:
      argv:
      - "bash"
      - "-c"
      - |
        mv /boot/firmware/cmdline.txt /boot/firmware/cmdline.txt.bup;
        head -n 1 /boot/firmware/cmdline.txt.bup | tr -d '\n' | cat - <(echo ' cgroup_memory=1 cgroup_enable=cpuset cgroup_enable=memory') > /boot/firmware/cmdline.txt;
        touch /boot/firmware/.CMDLINE_FIXED"
      creates: /boot/firmware/.CMDLINE_FIXED
  - name: Reboot
    ansible.builtin.reboot:
    when: fix_cmdline is changed
```

The first task `fix_cmdline` ensures that the `/boot/firmware/cmdline.txt` file has `cgroup_memory=1 cgroup_enable=cpuset cgroup_enable=memory` present. The tricky part is that it all has to be one line, and there doesn't seem to be a good way to append to the same line built in to ansible. So theres a little shell sorcery. The second task just ensures that the Raspberry Pis reboot, in order to enact the changes, but only when necessary.

Next we need to disable the swap. This is to give K8s full management of the memory available on the device, thus pods can be scheduled apropriately, and not where they will be constantly swapped out. 

```yaml
- name: Disable Swap
  hosts: all
  become: true
  tasks:
  - name: Remove swap from /etc/fstab
    mount:
      name: "{{ item }}"
      fstype: swap
      state: absent
    with_items:
      - swap
      - none
  - name: Disable swap
    command: swapoff -a
    when: ansible_swaptotal_mb > 0
```

This play will remove the swap line from `/etc/fstab`, and disable swap.

Finally, we can install Docker.

```yaml
- name: Install/Upgrade and configure Docker
  hosts: all
  become: true
  tasks:
  - name: Install/Upgrade Docker
    ansible.builtin.apt:
      name: docker.io
      update_cache: yes
      state: latest
  - name: Start Docker service
    ansible.builtin.systemd:
      name: docker
      enabled: yes
      state: started
  - name: Add user to docker group
    ansible.builtin.user: 
      name: "{{ ansible_user }}"
      groups: docker
      append: yes
```

This play installs the `docker.io` apt package, ensures the `docker` service is started, and will continue to start on each reboot, and then adds our `k3s` user to the `docker` group, which is important to ensure that docker commands can be executed without a password.

Throw each of these plays into a playbook file called `rpi_init.yaml` like so:

```yaml
- name: Fix RaspberryPi cmdline
  ...

- name: Disable Swap
  ...

- name: Install/Upgrade and configure Docker
  ...
```

Then setup an `inventory.toml` file by literally listing each ip address you play to use line by line:

```
192.168.1.20
192.168.1.21
192.168.1.22
```

Then run the ansible playbook like so:

```
ansible-playbook -u k3s -i inventory.toml rpi_init.yaml
```

After your pi reboots, run the following command using k3sup:

```
k3sup install \
  --cluster \
  --k3s-channel v1.21.0+k3s1 \ # Or whatever is latest [see](https://github.com/k3s-io/k3s/releases)
  --user k3s --ip 192.168.1.20
```

This command will initilize the first server node. To add additional nodes as other etcd/controlplane nodes (You need an odd integer greater than three) run:

```
k3sup join \
  --server \
  --k3s-channel v1.21.0+k3s1 \
  --user k3s --ip 192.168.1.21 \
  --server-user k3s --server-ip 192.168.1.20
```

And to add them as non-etcd/worker nodes just drop the `--server`:

```
k3sup join \
  --k3s-channel v1.21.0+k3s1 \
  --user k3s --ip 192.168.1.23 \
  --server-user k3s --server-ip 192.168.1.20
```

When adding new nodes, you may replace the `--server-ip` with any of the etcd/controlplane nodes.

That's it! There should be a file called `kubeconfig` in your local directory. Run 

```
mv ./kubeconfig ~/.kube/config
kubectl get nodes --watch
```

And watch as your nodes register themselves to the cluster.