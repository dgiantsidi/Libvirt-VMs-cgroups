# cgroups-and-libvirt-VMs

## Step 1: Starting a VM using virt-manager

- We create a VM using virt-manager interface. Then, a cgroup is created for this VM. In general, libvirt creates a single unique cgroup for each created/started VM under the mount point /sys/fs/cgroup/$CONTROLLER-NAME/machine.slice/. The cgroup created is defined in machine-qemu/{driver}\x2d{domain_id}\x2d{vm name}.scope
  - example in guest digian-vm: 
  ```
  	  +- machine-qemu\x2d330\x2ddigian\x2dvm.scope
            |   |
            |   +- emulator 	* qemu threads and others *
 	        |   +- vcpu0   	* thread of virtual cpu 0 *
	        |   +- vcpu1	        * thread of virtual cpu 1 *
	        |   +- vcpu2
	        |   +- vcpu3

  ```
  -References: https://libvirt.org/cgroups.html#resourceAPIs
  ```
Non-systemd cgroups layout 
On hosts which do not use systemd, each consumer has a corresponding cgroup named $VMNAME.libvirt-{qemu,lxc}. Each consumer is associated with exactly one partition, which also have a corresponding cgroup usually named $PARTNAME.partition. The exceptions to this naming rule are the three top level default partitions, named /system (for system services), /user (for user login sessions) and /machine (for virtual machines and containers). By default every consumer will of course be associated with the /machine partition.
```  
  
## Step 2: Exploring the cgroup
  
  ```
  cd /sys/fs/cgroup/cpuset/machine/test-cgroups.libvirt-qemu
  ls
  ```
  
  ```
  Output:
  
  cgroup.clone_children  cpuset.cpus  cpuset.mem_exclusive cpuset.memory_pressure  cpuset.mems  emulator  vcpu0
  cgroup.procs  cpuset.effective_cpus  cpuset.mem_hardwall  cpuset.memory_spread_page  cpuset.sched_load_balance
  notify_on_release  vcpu1  cpuset.cpu_exclusive  cpuset.effective_mems  cpuset.memory_migrate    
  cpuset.memory_spread_slab  cpuset.sched_relax_domain_level  tasks
  ```
  
  ```
  cat tasks
  ```
  
  ```
  Output:
  (none)
  ```
 - in each of the cgroups (vcp0, vcp1, emulator) we can retrieve cpuset.cpus in order to see which cpus are dedicated to these tasks
 - editing such files with the command `echo $mask > cpuset.cpus` we can also change cpu affinity
  
  ## Step 3: cgroups VS vcpupin/emulatorpin (virt-manager commands)
  
  ### Do virt manager commands update cgroups' files ?
  
 - Retrieve emulator-threads cpu affinity
  ```
  sudo virsh emulatorpin test-cgroups  
  
  output :
  emulator: CPU Affinity
	----------------------------------
	       *: 0-2

  ```
  
 - Change emulator-threads cpu affinity
  ```
  sudo virsh emulatorpin test-cgroups 2-3
 
  sudo virsh emulatorpin test-cgroups  
  
  output :
  emulator: CPU Affinity
	----------------------------------
	       *: 2-3
  ```
  
 - What happened in cgroup ?
  
  ```
  cat emulator/cpuset.cpus 
  2-3
  ```
    -  So, it updated..
    
 - Retrieve virtual cpus cpu affinity
  ```
  sudo virsh vcpuinfo test-cgroups
  VCPU:           0
	CPU:            3
	State:          running
	CPU time:       122.1s
	CPU Affinity:   ---y
	
	VCPU:           1
	CPU:            3
	State:          running
	CPU time:       43.9s
	CPU Affinity:   ---y
  ```
  
 - Change emulator-threads cpu affinity
 ```
 sudo virsh vcpupin test-cgroups 1 0
 sudo virsh vcpupin test-cgroups 0 1
 ```
 
 ```
  sudo virsh vcpuinfo test-cgroups
  VCPU:           0
	CPU:            1
	State:          running
	CPU time:       122.1s
	CPU Affinity:   ---y
	
	VCPU:           1
	CPU:            0
	State:          running
	CPU time:       43.9s
	CPU Affinity:   ---y
  ```
 - .. and cgroups are .. ?
  ```
  cat vcpu0/cpuset.cpus 
  1
  ```
  
  ```
  cat vcpu1/cpuset.cpus 
  0
  ```
  
  ### Do virt-manager’s commands report the updated cpumask when we edit cgroups ?
  
  ```
  root@digian-vm:/sys/fs/cgroup/cpuset/machine/test-cgroups.libvirt-qemu# cat vcpu1/cpuset.cpus 
  
  Output: 
  0
  ```
  
  - Change cpuset.cpus
  
  ```
  root@digian-vm:/sys/fs/cgroup/cpuset/machine/test-cgroups.libvirt-qemu/vcpu1# echo 3 > cpuset.cpus 
  
  root@digian-vm:/sys/fs/cgroup/cpuset/machine/test-cgroups.libvirt-qemu/vcpu1# cat cpuset.cpus 
  
  Output:
  3
  ```
 
 - virsh commands 
 
 ```
 root@digian-vm:/sys/fs/cgroup/cpuset/machine/test-cgroups.libvirt-qemu/vcpu1# sudo virsh vcpuinfo test-cgroups 
 
 Output:
 VCPU:           0
CPU:            1
State:          running
CPU time:       123.1s
CPU Affinity:   -y--

VCPU:           1
CPU:            3
State:          running
CPU time:       44.6s
CPU Affinity:   ---y

 ```
