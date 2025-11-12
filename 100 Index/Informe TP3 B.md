# Informe TP3 B

**Integrantes**: Lopez Romero Gabriel

---

## Características de la VM

Habiendo accedido a *Cloud Shell*, vamos a ver los resultados de ciertos comandos que permiten analizar la arquitectura de la computadora con la que estamos trabajando:

### lscpu

```bash
gabylopezromero13@cloudshell:~ (woven-nimbus-478020-c7)$ lscpu
Architecture:                x86_64
  CPU op-mode(s):            32-bit, 64-bit
  Address sizes:             46 bits physical, 48 bits virtual
  Byte Order:                Little Endian
CPU(s):                      2
  On-line CPU(s) list:       0,1
Vendor ID:                   GenuineIntel
  Model name:                Intel(R) Xeon(R) CPU @ 2.20GHz
    CPU family:              6
    Model:                   79
    Thread(s) per core:      2
    Core(s) per socket:      1
    Socket(s):               1
    Stepping:                0
    BogoMIPS:                4399.99
    Flags:                   fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ss ht syscall nx pdpe1gb rdtscp lm constant_
                             tsc rep_good nopl xtopology nonstop_tsc cpuid tsc_known_freq pni pclmulqdq ssse3 fma cx16 pcid sse4_1 sse4_2 x2apic movbe popcnt aes xsave avx
                              f16c rdrand hypervisor lahf_lm abm 3dnowprefetch pti ssbd ibrs ibpb stibp fsgsbase tsc_adjust bmi1 hle avx2 smep bmi2 erms invpcid rtm rdseed
                              adx smap xsaveopt arat md_clear arch_capabilities
Virtualization features:     
  Hypervisor vendor:         KVM
  Virtualization type:       full
Caches (sum of all):         
  L1d:                       32 KiB (1 instance)
  L1i:                       32 KiB (1 instance)
  L2:                        256 KiB (1 instance)
  L3:                        55 MiB (1 instance)
NUMA:                        
  NUMA node(s):              1
  NUMA node0 CPU(s):         0,1
Vulnerabilities:             
  Gather data sampling:      Not affected
  Indirect target selection: Mitigation; Aligned branch/return thunks
  Itlb multihit:             Not affected
  L1tf:                      Mitigation; PTE Inversion
  Mds:                       Mitigation; Clear CPU buffers; SMT Host state unknown
  Meltdown:                  Mitigation; PTI
  Mmio stale data:           Vulnerable: Clear CPU buffers attempted, no microcode; SMT Host state unknown
  Reg file data sampling:    Not affected
  Retbleed:                  Mitigation; IBRS
  Spec rstack overflow:      Not affected
  Spec store bypass:         Mitigation; Speculative Store Bypass disabled via prctl
  Spectre v1:                Mitigation; usercopy/swapgs barriers and __user pointer sanitization
  Spectre v2:                Mitigation; IBRS; IBPB conditional; STIBP conditional; RSB filling; PBRSB-eIBRS Not affected; BHI SW loop, KVM SW loop
  Srbds:                     Not affected
  Tsa:                       Not affected
  Tsx async abort:           Mitigation; Clear CPU buffers; SMT Host state unknown
  Vmscape:                   Not affected
```

### free

```bash
gabylopezromero13@cloudshell:~ (woven-nimbus-478020-c7)$ free
               total        used        free      shared  buff/cache   available
Mem:         8138108      616216     6534840        1140     1230216     7521892
Swap:              0           0           0
```


### uname

```bash
Linux
```


### lsb_release -a

```bash
gabylopezromero13@cloudshell:~ (woven-nimbus-478020-c7)$ lsb_release -a
No LSB modules are available.
Distributor ID: Ubuntu
Description:    Ubuntu 24.04.3 LTS
Release:        24.04
Codename:       noble
```


### df -h

```bash
gabylopezromero13@cloudshell:~ (woven-nimbus-478020-c7)$ df -h
Filesystem                         Size  Used Avail Use% Mounted on
overlay                             95G   54G   42G  57% /
tmpfs                               64M     0   64M   0% /dev
/dev/sda1                           95G   54G   42G  57% /root
/dev/disk/by-id/google-home-part1  4.8G   84K  4.6G   1% /home
/dev/root                          2.0G  1.2G  762M  62% /usr/lib/modules
shm                                 64M     0   64M   0% /dev/shm
```


### ifconfig

```bash
gabylopezromero13@cloudshell:~ (woven-nimbus-478020-c7)$ ifconfig
docker0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1460
        inet 172.17.0.1  netmask 255.255.0.0  broadcast 172.17.255.255
        ether 6e:b6:95:55:80:c0  txqueuelen 0  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1460
        inet 10.88.0.3  netmask 255.255.0.0  broadcast 10.88.255.255
        ether a6:2c:e4:21:69:16  txqueuelen 1000  (Ethernet)
        RX packets 5831  bytes 1210283 (1.2 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 4356  bytes 746436 (746.4 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 817  bytes 98126 (98.1 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 817  bytes 98126 (98.1 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```


Podemos notar que es un servidor Linux con especificaciones bastante sencillas