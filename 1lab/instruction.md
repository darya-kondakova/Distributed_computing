## VM VirtualBox Linux
```
head  192.168.33.2
node1 192.168.33.3  
node2 192.168.33.4
```


## Hosts
```
sudo vi /etc/hosts  

    127.0.0.1       localhost
    127.0.1.1       head
    192.168.33.2    head
    192.168.33.3    node1
    192.168.33.4    node2
```


## GCC
```
sudo apt update
sudo apt install build-essential
```


## NFS
https://www.digitalocean.com/community/tutorials/how-to-set-up-an-nfs-mount-on-ubuntu-20-04-ru  
https://andreyex.ru/ubuntu/kak-ustanovit-i-nastroit-server-nfs-v-ubuntu-20-04/  

**server (head)**
```
sudo apt update
sudo apt install nfs-kernel-server
```
```
sudo vi /etc/exports  

    /home   192.168.33.0/24(rw,sync,no_root_squash,no_subtree_check)
    /opt    192.168.33.0/24(rw,sync,no_root_squash,no_subtree_check)
```
```
sudo exportfs -ar
sudo exportfs -v
```
**client (node)**
```
sudo apt update
sudo apt install nfs-common

sudo mount 192.168.33.2:/home /home
sudo mount 192.168.33.2:/opt /opt
```
```
sudo vi /etc/fstab  

    192.168.33.2:/home /home	  nfs		defaults,timeo=900,retrans=5,_netdev	0 0
    192.168.33.2:/opt /opt	  nfs		defaults,timeo=900,retrans=5,_netdev	0 0
```


## OpenMPI
https://edu.itp.phys.ethz.ch/hs12/programming_techniques/openmpi.pdf  
https://download.open-mpi.org/release/open-mpi/v4.1/openmpi-4.1.2.tar.bz2  

**на head**
```
mkdir /opt/src
cd /opt/src
sudo wget https://download.open-mpi.org/release/open-mpi/v4.1/openmpi-4.1.2.tar.bz2
sudo tar -jxf openmpi-4.1.2.tar.bz2
cd openmpi-4.1.2
sudo ./configure --prefix=$HOME/opt/openmpi 
sudo make all 
sudo make install
sudo rm -rf src/
```
**на всех узлах**
```
sudo vi /etc/environment  

    PATH="...:/opt/openmpi/bin"
    LD_LIBRARY_PATH="/opt/openmpi/lib"
```

## Test OpenMPI 
```
./hosts_names  

    head slots=2
    node1 slots=2
    node2 slots=2
```
```
mpicc -o hello hello.cpp
mpirun --hostfile hosts_names -np 6 ./hello
```

## SSH
```
sudo apt install openssh-server
```
**на head**
```
ssh-keygen -t rsa
ssh-copy-id darya@192.168.33.3
ssh-copy-id darya@192.168.33.4
```

Если не работает вход без пароля:  
- почистить `authorized_keys` на **node'ах**;  
- почистить `known_hosts` на **head**;  
- создать ключ заново, скопировать на **node'ы**.


## Slurm
https://blog.llandsmeer.com/tech/2020/03/02/slurm-single-instance.html  
https://network.cmu.ac.th/wiki/index.php/Slurm_Install  
https://slurm.schedmd.com/quickstart_admin.html 
```
sudo apt install munge
```
```
sudo apt install mariadb-server
sudo mysql -u root
create database slurm_acct_db;
create user 'slurm'@'localhost';
set password for 'slurm'@'localhost' = password('slurmdbpass');
grant usage on *.* to 'slurm'@'localhost';
grant all privileges on slurm_acct_db.* to 'slurm'@'localhost';
flush privileges;
exit;
```
```
sudo apt install slurmd slurm-client slurmctld  

sudo vi /etc/slurm-llnl/slurm.conf

    # сгенирировать файл здесь - https://slurm.schedmd.com/configurator.html 

    # в конце расписать каждый узел 
    NodeName=head CPUs=1 RealMemory=512 Sockets=1 CoresPerSocket=4 ThreadsPerCore=1 State=UNKNOWN
    NodeName=node1 CPUs=1 RealMemory=512 Sockets=1 CoresPerSocket=4 ThreadsPerCore=1 State=UNKNOWN
    NodeName=node2 CPUs=1 RealMemory=512 Sockets=1 CoresPerSocket=4 ThreadsPerCore=1 State=UNKNOWN
    PartitionName=debug Nodes=node[1-2] Default=YES MaxTime=INFINITE State=UP
```
```
scp /etc/slurm-llnl/slurm.conf darya@node1:/etc/slurm-llnl/slurm.conf
scp /etc/slurm-llnl/slurm.conf darya@node2:/etc/slurm-llnl/slurm.conf
```
```
sudo vi /etc/slurm-llnl/cgroup.conf  

    CgroupAutomount=yes
    CgroupReleaseAgentDir="/etc/slurm/cgroup"
    ConstrainCores=yes
    ConstrainDevices=yes
    ConstrainRAMSpace=yes
```
```
sudo systemctl restart slurmctld
sudo systemctl restart slurmd
sinfo
```

## Munge

**на head**
```
sudo create-munge-key
```

скопировать ключ `/etc/munge/munge.key` на node-ы сохранив права:
```
sudo scp /etc/munge/munge.key darya@node1:/etc/munge/munge.key
sudo scp /etc/munge/munge.key darya@node2:/etc/munge/munge.key
```

**на node’ах**
```
sudo chown munge:munge /etc/munge/munge.key
```

после синхронизации ключей перезапустить munge **везде**:
```
sudo systemctl restart munge
```

slurm нужно собирать `with pmix` 

создать папку `/var/spool/slurmctld` для пользователя `slurm`:
```
sudo mkdir /var/spool/slurmctld
sudo chown slurm:slurm /var/spool/slurmctld
```

`slurm.conf` нужно делать в конфигураторе внутри виртуалки:
- сгенирировать файл [здесь](https://slurm.schedmd.com/configurator.html) 
- скачать html файл на машину 
- переместить содержимое html файла в `/etc/slurm-llnl/`

`slurmd` запускается только при прописанном файле `cgroup.conf`

собираем pmix (openpmi) 3.2.3
для сборки pmi и slurm:
```
sudo apt install libmunge-dev libhwloc-dev libevent-dev
```

сборка pmi:
```
./configure --prefix=<pmix3.2.3-folder>/installhere
make all install
```

сборка slurm:
```
./configure --prefix=/usr/local --sysconfdir=/etc/slurm-llnl --with-pmix=<pmix3.2.3-folder>/installhere --with-munge
sudo make install
```

если STATE down:
```
sudo scontrol update nodename=node2 state=idle
cd
```

**на всех**
```
sudo systemctl start slurmd 
```

**на head**
```
sudo systemctl start slurmctld
```

## Создание пользователей
```
ls -l /home
sudo useradd -m user1
sudo passwd user1
```
```
sudo userdel -r user1 # удалить пользователя
sudo usermod -aG group1 user1 # добавить пользователя в группу
sudo deluser user1 group1 # удалить пользователя из группы
```

## Копирование пользователей
https://www.8host.com/blog/passwd-i-adduser-upravlenie-parolyami-na-servere-linux/  
http://www.sea-lab.ru/it-arcticles/perenos_polzovateley_so_starogo_servera_linux_na_noviy.php  

**откуда копируем**
```
mkdir move/
export UGIDLIMIT=1000
awk -v LIMIT=$UGIDLIMIT -F: '($3>=LIMIT) && ($3!=65534)' /etc/passwd > move/passwd.mig
awk -v LIMIT=$UGIDLIMIT -F: '($3>=LIMIT) && ($3!=65534)' /etc/group > move/group.mig
awk -v LIMIT=$UGIDLIMIT -F: '($3>=LIMIT) && ($3!=65534) {print $1}' /etc/passwd | tee - |egrep -f - /etc/shadow > move/shadow.mig
cp /etc/gshadow move/gshadow.mig
scp -r move/* darya@node1:/path/to/location
scp -r move/* darya@node2:/path/to/location
```

**куда копируем**
```
mkdir newsusers.bak
sudo cp /etc/passwd /etc/shadow /etc/group /etc/gshadow newsusers.bak
cd /path/to/location
sudo cat passwd.mig >> sudo /etc/passwd
sudo cat group.mig >> sudo /etc/group
sudo cat shadow.mig >> sudo /etc/shadow
sudo /bin/cp gshadow.mig /etc/gshadow
reboot
```

## LDAP
**server (head)**
```
apt-get install slapd ldap-utils
dpkg-reconfigure slapd
    Omit OpenLDAP server configuration? <No> 
    Database backend to use: HDB
    Do you want the database to be removed when slapd is purged? <No>
    Move old database? <Yes>
    Allow LDAPv2 protocol? <No>
```
[...продолжение](https://www.youtube.com/watch?v=8FOEIC4eMEA&list=PLLSuIcxrb1x7saUcMLJCCUex2X45HDQS0&index=8)  

**client (node)**  
[...продолжение](https://www.youtube.com/watch?v=DBL_VdR-eVs&list=PLLSuIcxrb1x7saUcMLJCCUex2X45HDQS0&index=9)


## Ganglia
https://www.youtube.com/watch?v=pxIfGQ_hrpY&list=PLLSuIcxrb1x7saUcMLJCCUex2X45HDQS0&index=7  

**head**
```
sudo apt-get install ganglia-monitor rrdtool gmetad ganglia-webfrontend -y
sudo cp /etc/ganglia-webfrontend/apache.conf /etc/apache2/sites-enabled/ganglia.conf
```
```
sudo vi /etc/ganglia/gmetad.conf  

	data_source “head” 192.168.33.2
```
```
sudo vi /etc/ganglia/gmond.conf  

    globals {
        deaf = no
    }
    ...
    cluster {
        name = "head"
    }
    ...
    udp_send_channel {
        # mcast_join = 239.2.11.71
        host = 192.168.33.2
    }
    udp_recv_channel {
        # mcast_join = 239.2.11.71
        # bind = 239.2.11.71
    }
```
```
service ganglia-monitor restart
service gmetad restart
service apache2 restart
```

**node**
```
sudo apt-get install ganglia-monitor -y
```
```
sudo vi /etc/ganglia/gmond.conf  

    globals {
        deaf = yes
    }
    cluster {
        name = "head"
    }
    udp_send_channel {
        # mcast_join = 239.2.11.71
        host = 192.168.33.2
    }
    # udp_recv_channel {
    #     ...
    # }
```
```
service ganglia-monitor restart
```
