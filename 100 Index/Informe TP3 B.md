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


---

## top

Ahora vamos a analizar los procesos que esta VM está ejecutando

```bash
top - 20:42:04 up 17 min,  0 user,  load average: 0.02, 0.15, 0.21
Tasks:  23 total,   1 running,  22 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.8 us,  0.8 sy,  0.0 ni, 98.2 id,  0.0 wa,  0.0 hi,  0.2 si,  0.0 st 
MiB Mem :   7947.4 total,   6389.4 free,    592.3 used,   1203.2 buff/cache     
MiB Swap:      0.0 total,      0.0 free,      0.0 used.   7355.1 avail Mem 

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND                                                                                            
    261 root      20   0 1729312  47736  34852 S   0.3   0.6   0:01.21 containerd                                                                                         
      1 root      20   0    4324   3252   2968 S   0.0   0.0   0:00.08 bash                                                                                               
      8 syslog    20   0  222140   3780   2440 S   0.0   0.0   0:00.01 rsyslogd                                                                                           
     24 root      20   0   40080  29060  10884 S   0.0   0.4   0:00.71 python                                                                                             
     25 root      20   0    6196   2724   2484 S   0.0   0.0   0:00.00 logger                                                                                             
     70 root      10 -10   12020   4116   3004 S   0.0   0.1   0:00.01 sshd                                                                                               
    219 root      20   0 1973044  75152  54428 S   0.0   0.9   0:00.48 dockerd                                                                                            
    263 root      20   0 1231944   8032   6856 S   0.0   0.1   0:00.01 editor-proxy                                                                                       
    264 root      20   0   16320   6352   5428 S   0.0   0.1   0:00.01 sudo                                                                                               
    274 root      20   0 1226100   2672   1976 S   0.0   0.0   0:00.00 tmux-agent                                                                                         
    280 root      10 -10   13992   9700   8324 S   0.0   0.1   0:00.03 sshd                                                                                               
    353 root      10 -10   13992   9808   8436 S   0.0   0.1   0:00.03 sshd                                                                                               
    468 root      20   0    2696   1608   1512 S   0.0   0.0   0:00.00 sleep                                                                                              
    479 gabylop+  10 -10   14252   6408   4752 S   0.0   0.1   0:00.00 sshd                                                                                               
    480 gabylop+  10 -10   14252   6432   4776 S   0.0   0.1   0:00.02 sshd                                                                                               
    481 gabylop+  10 -10    7340   3628   3380 S   0.0   0.0   0:00.00 bash                                                                                               
    484 gabylop+  10 -10    7604   4352   3760 S   0.0   0.1   0:00.00 bash                                                                                               
    487 gabylop+  10 -10    7340   3612   3360 S   0.0   0.0   0:00.00 bash                                                                                               
    488 gabylop+  10 -10    7340   3596   3336 S   0.0   0.0   0:00.00 start-shell.sh                                                                                     
    492 gabylop+  10 -10    6880   3444   3144 S   0.0   0.0   0:00.00 tmux                                                                                               
    494 gabylop+  10 -10    7280   2864   2092 S   0.0   0.0   0:00.03 tmux                                                                                               
```

Podemos notar que no hay nada extraño. La mayoría son servicios normales de cualquier instalación de Linux, como bash (lo que estamos utilizando ahora mismo), sshd, sudo, docker. Tambien vemos que está corriendo python.


---

## Probando ejercicios

Finalmente, vamos a hacer una prueba corriendo uno de los ejercicios desarrollados en la materia para analizar el rendimiento en la VM de Google.