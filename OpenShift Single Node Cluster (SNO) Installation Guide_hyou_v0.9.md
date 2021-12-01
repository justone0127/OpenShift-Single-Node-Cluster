## OpenShift Single Node Cluster (SNO) Installation Guide

OpenShift 4.9 버전부터 Single Node에 완전한 OpenShift를 배포할 수 있습니다. 완벽하게 지원되는 이 토폴로지는 3-Node Cluster와 Remote Worker 토폴로지를 결합하여 더 많은 Edge 환경에서 더 많은 고객 요구 사항을 충족할 수 있는 3가지 옵션을 제공합니다. 



Single Node가 Control 및 Worker Node 기능을 모두 제공하기 때문에 중앙 관리 사이트에서 Kubernetes를 채택하고 Edge 사이트에서 독립적인 Kubernetes 클러스터를 원하는 사용자는 더 작은 OpenShift footprint를 배포할 수 있으며 중앙 집중식 관리 클러스터에 대한 의존도를 최소화할 수 있습니다. 또한, 필요할 때 자율적으로 실행할 수 있습니다.



Multi Node OpenShift와 마찬가지로 Single Node 배포는 동일한 기술과 도구를 활용하고 운영체제, 기능이 풍부한 Kubernetes(클러스터 서비스 포함)를 포함한 컨테이너 스택 전체에 걸쳐 일관된 업그레이드 및 수명 주기 관리를 제공하여 OpenShift 경험을 확장합니다.



### 1. OpenShift Single Node Cluster Resource Requirement

- **Hardware requirements**

  | Profile | vCPU         | Memory      | Storage |
  | ------- | ------------ | ----------- | ------- |
  | Minimum | 8 vCPU cores | 32GB of RAM | 120GB   |

  >  SMT(동시 멀티스레딩) 또는 하이퍼스레딩이 활성화되지 않은 경우 vCPU 1개는 물리적 Core 1개와 동일합니다.
  >
  > BareMetal인 경우에는 가상 미디어로 부팅할 때 서버에 BMC(Baseboard Management Controller)가 있어야 합니다.

- **Networking** : 서버는 라우팅 가능한 네트워크에 연결되어 있지 않은 경우에는 인터넷에 접근거나 로컬 레지스트리에 접근할 수 있어야 합니다. 서버에는 Kubernetes API, Ingress route 및 클러스터 노드 도메인에 대한 DHCP 또는 고정 IP 주소가 있어야 합니다. (FQDN)

- **Required DNS records**

  | Usage          | FQDN                                      | Description                                                  |
  | -------------- | ----------------------------------------- | ------------------------------------------------------------ |
  | Kubernetes API | `api.<cluster_name>.<base_domain>`        | DNS A/AAAA 또는 CNAME 레코드를 추가합니다.                   |
  | Ingress route  | `*.apps.<cluster_name>.<base_domain>`     | 노드를 대상으로 하는 와일드카드 DNS A/AAAA 또는 CNAME 레코드를 추가합니다. |
  | Cluster node   | `<hostname>.<cluster_name>.<base_domain>` | DNS A/AAAA 또는 CNAME 레크도와 DNS PTR 레코드를 추가하여 노드를 식별합니다. |

  > 영구 주소 IP가 없으면 apiserver와 etcd간의 통신이 실패할 수 있습니다.



### 2. 테스트 목적 및 구축 환경

OpenShift 4.9 버전으로 Single Node Cluster를 구축하여 간단한 애플리케이션을 배포하여 정상 동작하는지 확인해 보도록 하겠습니다.

- **테스트 환경**

  | Machine   | OS          | vCPU | RAM   | Storage |
  | --------- | ----------- | ---- | ----- | ------- |
  | bastion   | RHEL 7.9    | 8    | 16 GB | 300 GB  |
  | bootstrap | RHCOS 4.9.8 | 8    | 32 GB | 300 GB  |
  | master    | RHCOS 4.9.8 | 8    | 32 GB | 300 GB  |

**2-1) User-Provisioned Infrastructuer (UPI)를 위해 준비해야 하는 부분**

- Bastion 구성을 위한 RHEL ISO & RHCOS iso 파일 다운로드 

  - RHEL 7.9 iso 다운로드

    https://access.redhat.com/downloads/content/69/ver=/rhel---7/7.9/x86_64/product-software

    위의 링크에서 8.4 iso 파일을 다운로드하여 bastion 서버를 구성합니다.

  - RHCOS iso 파일 다운로드

    - rhcos-live.x86_64.iso

      링크) https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.9/latest/rhcos-live.x86_64.iso

    - rhcos-4.9.0-x86_64-metal.x86_64.raw.gz

      링크) https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.9/latest/rhcos-4.9.0-x86_64-metal.x86_64.raw.gz

      > 위의 링크 클릭시 바로 다운로드가 시작됩니다.

  - **openshift-install**, **openshift-client** 파일 
    - openshift-install-linux-4.9.8.tar.gz
    - openshift-client-linux-4.9.8.tar.gz

- Load Balancer (Haproxy)

  Haporxy는 OCP 설치 시 Load Balancer 역할을 합니다. Haporxy는 master, infra, bootstrap node 간에 통신을 할 수 있는 front and backend protocol(Upstream)을 제공함으로써 OCP 설치시에 중요한 브로커 역할을 수행합니다.

- DNS 구성

  RHOCP 내부 DNS로 Node 간 FQDN Resolving으로 활용하게 되며 구성을 위한 named.conf 설정 변경, zone 파일 생성 및 DNS 서버 방화벽 오픈 작업을 수행합니다.



### 3. Subscription 설정 및 Repository 구성

**3-1) Subscription 등록**

```bash
$ rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release  
// Red Hat 패키지를 확인하려면 Red Hat GPG KEY를 import 해야함 (Optional)

$ subscription-manager register
Registering to: subscription.rhsm.redhat.com:443/subscription
Username: ${USERNAME} // access.redhat.com 아이디
Password: ${PASSWORD} // access.redhat.com 패스워드
The system has been registered with ID: 6b924884-039f-408e-8fad-8d6b304bc2b5
The registered system name is: localhost.localdomain
```

> 등록 시 redhat customer portal ID/PWD를 입력합니다.

**3-2) OpenShift 설치에 필요한 RPM 활성화**

```bash
$ subscription-manager list --available --matches '*OpenShift*' > aaa.txt

// OpenShift Subscription만 따로 뽑아서 그 중 하나를 선택하여 등록 합니다.

$ subscription-manager attach --pool=8a85f99c6c8b9588016c8be0f1b50ec1
Successfully attached a subscription for: OpenShift Employee Subscription

$ subscription-manager repos --disable="*"

$  subscription-manager repos \
    --enable="rhel-7-server-rpms" \
    --enable="rhel-7-server-extras-rpms" \
    --enable="rhel-7-server-ansible-2.9-rpms" \
    --enable="rhel-7-server-ose-4.9-rpms"
```



### 4. Bastion 서버 Hostname  변경

**4-1) hostname변경**

```bash
$ hostnamectl set-hostname bastion.single.demo.com
```

**4-2) 변경된 hostname 확인**

```bash
[root@bastion html]# hostnamectl
   Static hostname: bastion.single.demo.com
         Icon name: computer-vm
           Chassis: vm
        Machine ID: e5208f8b7a7a40a6aaa8fc3c600244f9
           Boot ID: 4bfab396fbd948adacfef5779efbb391
    Virtualization: kvm
  Operating System: Employee SKU
       CPE OS Name: cpe:/o:redhat:enterprise_linux:7.9:GA:server
            Kernel: Linux 3.10.0-1160.el7.x86_64
      Architecture: x86-64
```



### 5. httpd Install
**5-1) httpd install**

```bash
[root@bastion ~]# yum install -y httpd
```

**5-2) httpd port 변경**

> haproxy 구성 시에 80/443 port가 사용 되므로 겹치지 않도록 httpd port를 8080 (다른 Port)으로 변경

```bash
cd /etc/httpd/conf
sed -i 's/Listen 80/Listen 8080/g' httpd.conf
```

- 확인

  ```bash
  grep "Listen" httpd.conf
  ```

**5-3) httpd DocumentRoot 디렉토리 권한 설정 확인**

```bash
chmod -R +r /var/www/html/
restorecon -vR /var/www/html
```

**5-4) Port 방화벽 해제 및 서비스 시작**

```bash
firewall-cmd --permanent --add-port=8080/tcp
firewall-cmd --permanent --add-service=http
firewall-cmd --reload
systemctl enable httpd
systemctl start httpd
systemctl status httpd
```



### 6. DNS 구성

**6-1) bind, bind-utils Install**

``` bash
$ yum install -y bind bind-utils
```

**6-2) DNS Port 방화벽 해제**

```bash
firewall-cmd --perm --add-port=53/tcp
firewall-cmd --perm --add-port=53/udp
firewall-cmd --add-service dns --zone=internal --perm 
firewall-cmd --reload
```

**6-3) named.conf 수정**

> 파일 위치 : /etc/named.conf

- 외부 인터넷 접속이 필요한 경우, 상위 DNS 설정으로 외부 네트워크로 나가도록 설정이 가능합니다.

- 인터넷이 가능하지 않은 경우 해당 설정 부분이 사용되지 않습니다. 

- `etc/named.conf` 파일 수정

  ```bash
  cd /etc/
  cp named.conf named.conf.bak
  
  sed -i 's/listen-on port 53 { 127.0.0.1;/listen-on port 53 { any;/g' named.conf 
  sed -i 's/listen-on-v6 port 53 { ::1;/listen-on-v6 port 53 { none;/g' named.conf
  sed -i 's/allow-query     { localhost;/allow-query     { any;/g' named.conf
  ```

- 파일 내용 예시

  ```bash
  //
  // named.conf
  //
  // Provided by Red Hat bind package to configure the ISC BIND named(8) DNS
  // server as a caching only nameserver (as a localhost DNS resolver only).
  //
  // See /usr/share/doc/bind*/sample/ for example named configuration files.
  //
  // See the BIND Administrator's Reference Manual (ARM) for details about the
  // configuration located in /usr/share/doc/bind-{version}/Bv9ARM.html
  
  options {
          listen-on port 53 { any; };
          listen-on-v6 port 53 { none; };
          directory       "/var/named";
          dump-file       "/var/named/data/cache_dump.db";
          statistics-file "/var/named/data/named_stats.txt";
          memstatistics-file "/var/named/data/named_mem_stats.txt";
          recursing-file  "/var/named/data/named.recursing";
          secroots-file   "/var/named/data/named.secroots";
          allow-query     { any; };
  
          /*
           - If you are building an AUTHORITATIVE DNS server, do NOT enable recursion.
           - If you are building a RECURSIVE (caching) DNS server, you need to enable
             recursion.
           - If your recursive DNS server has a public IP address, you MUST enable access
             control to limit queries to your legitimate users. Failing to do so will
             cause your server to become part of large scale DNS amplification
             attacks. Implementing BCP38 within your network would greatly
             reduce such attack surface
          */
          recursion yes;
          forward only;
          
          forwarders {
                  8.8.8.8;   // 상위 DNS 설정 (외부 네트워크를 위한), 인터넷이 가능하지 않으면 설정 필요 없음, 서버 환경에 맞는 DNS 서버 IP를 입력해야 함
          };
  
          dnssec-enable yes;
          dnssec-validation yes;
  
          /* Path to ISC DLV key */
          bindkeys-file "/etc/named.root.key";
  
          managed-keys-directory "/var/named/dynamic";
  
          pid-file "/run/named/named.pid";
          session-keyfile "/run/named/session.key";
  };
  
  logging {
          channel default_debug {
                  file "data/named.run";
                  severity dynamic;
          };
  };
  
  zone "." IN {
          type hint;
          file "named.ca";
  };
  
  include "/etc/named.rfc1912.zones";
  include "/etc/named.root.key";
  ```

**6.4) named.rfc192.zones 파일 수정**

> 파일 위치 :  /etc/named.rfc1912.zones

- `/etc/named.rfc192.zones`의 맨 하단 부분에 추가 할 도메인 zone 파일 정보를 입력합니다.

  ```bash
  // named.rfc1912.zones:
  //
  // Provided by Red Hat caching-nameserver package
  //
  // ISC BIND named zone configuration for zones recommended by
  // RFC 1912 section 4.1 : localhost TLDs and address zones
  // and http://www.ietf.org/internet-drafts/draft-ietf-dnsop-default-local-zones-02.txt
  // (c)2007 R W Franks
  //
  // See /usr/share/doc/bind*/sample/ for example named configuration files.
  //
  
  zone "localhost.localdomain" IN {
          type master;
          file "named.localhost";
          allow-update { none; };
  };
  
  zone "localhost" IN {
          type master;
          file "named.localhost";
          allow-update { none; };
  };
  
  zone "1.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.ip6.arpa" IN {
          type master;
          file "named.loopback";
          allow-update { none; };
  };
  
  zone "1.0.0.127.in-addr.arpa" IN {
          type master;
          file "named.loopback";
          allow-update { none; };
  };
  
  zone "0.in-addr.arpa" IN {
          type master;
          file "named.empty";
          allow-update { none; };
  };
  
  zone "demo.com" IN {
          type master;
          file "/var/named/single.demo.com.zone";
          allow-update { none; };
  };
  
  zone "168.76.10.in-addr.arpa" IN {
          type master;
          file "/var/named/168.76.10.in-addr.rev";
          allow-update { none; };
  };
  ```

  **6-5) 정방향 zone 파일 생성**

  > 파일 위치 : /var/named/single.demo.com.zone

  ```bash
  $TTL 1D
  @   IN SOA  @ ns.demo.com. (
              1019951001  ; serial
              3H          ; refresh
              1H          ; retry
              1W          ; expiry
              1H )        ; minimum
  
  @           IN NS       ns.demo.com.
  @           IN A        DNS SERVER IP
  
  ; Ancillary services
  lb.single          IN A        Load Balancer IP
  
  ; Bastion or Jumphost
  bastion.single     IN A        Bastion Server IP
  ns     IN A                      DNS SERVER IP
  
  ; OCP Cluster
  bootstrap.single  IN A        Bootstrap IP
  master1.single    IN A        Master IP
  
  api.single         IN A    Load Balancer IP  ; external LB interface
  api-int.single     IN A    Load Balancer IP  ; internal LB interface
  *.apps.single      IN A    Load Balancer IP
  ```

  **6-6) 역방향 Zone 파일 생성**

  > 파일 위치 : /var/named/xxx.xx.xxx.in-addr.rev

  ```bash
  $TTL    86400
  @       IN      SOA   demo.com. ns.demo.com. (
                        179311     ; Serial
                        3H         ; Refresh
                        1H         ; Retry
                        1W         ; Expire
                        1H )       ; Minimum
  
  @      IN NS   ns.
  71     IN PTR  ns.
  71     IN PTR  bastion.single.demo.com.
  75     IN PTR  bootstrap.single.demo.com.
  72     IN PTR  master1.single.demo.com.
  71     IN PTR  api.single.demo.com.
  71     IN PTR  api-int.single.demo.com.
  ```

- zone 파일 권한 설정 확인

  ```bash
  [root@bastion named]# ls -lZ
  -rw-r-----. root  named system_u:object_r:named_zone_t:s0 xxx.xx.xx.in-addr.rev
  drwxrwx---. named named system_u:object_r:named_cache_t:s0 data
  drwxrwx---. named named system_u:object_r:named_cache_t:s0 dynamic
  -rw-r-----. root  named system_u:object_r:named_conf_t:s0 named.ca
  -rw-r-----. root  named system_u:object_r:named_zone_t:s0 named.empty
  -rw-r-----. root  named system_u:object_r:named_zone_t:s0 named.localhost
  -rw-r-----. root  named system_u:object_r:named_zone_t:s0 named.loopback
  -rw-r-----. root  named system_u:object_r:named_zone_t:s0 single.demo.com.zone
  drwxrwx---. named named system_u:object_r:named_cache_t:s0 slaves
  ```

- zone 파일 권한 변경 (예시)

  ```bash
  chown root:named ${filename}
  chcon -Rv -u system_u ${filename}
  ```

**6-5) DNS 서비스 등록 및 시작, 상태 확인**

- 서비스 등록 및 시작 & 상태 확인 명령어

  ```bash
  systemctl enable named
  systemctl start named
  systemctl status named 
  ```

**6-6) 정방향 & 역방향 DNS Resolving 확인**

- 정방향 DNS Resolving 확인

  ```bash
  [root@bastion named]# nslookup bootstrap.single.demo.com
  Server:         xx.xx.xxx.xx
  Address:        xx.xx.xxx.xx#53
  
  Name:   bootstrap.single.demo.com
  Address: bootstarp.ip
  ```

- 역방향 DNS Resolving 확인

  ```bash
  [root@bastion named]# dig -x ${bootstrap_IP} +short
  bootstrap.single.demo.com.
  ```

### 7. Load Balancer 구성 (Haproxy)

**7-1) haproxy Install**

```bash
$ yum install -y haproxy
```

**7-2) haporxy 설정**

> 파일 위치 : /etc/haproxy/haproxy.cfg

```bash
#---------------------------------------------------------------------
# Example configuration for a possible web application.  See the
# full configuration options online.
#
#   http://haproxy.1wt.eu/download/1.4/doc/configuration.txt
#
#---------------------------------------------------------------------

#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
    # to have these messages end up in /var/log/haproxy.log you will
    # need to:
    #
    # 1) configure syslog to accept network log events.  This is done
    #    by adding the '-r' option to the SYSLOGD_OPTIONS in
    #    /etc/sysconfig/syslog
    #
    # 2) configure local2 events to go to the /var/log/haproxy.log
    #   file. A line like the following can be added to
    #   /etc/sysconfig/syslog
    #
    #    local2.*                       /var/log/haproxy.log
    #
    log         127.0.0.1 local2

    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon

    # turn on stats unix socket
    stats socket /var/lib/haproxy/stats

#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------
defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 4000

#---------------------------------------------------------------------
# static backend for serving up images, stylesheets and such
#---------------------------------------------------------------------
backend static
    balance     roundrobin
    server      static 127.0.0.1:4441 check

#---------------------------------------------------------------------
# round robin balancing between the various backends
#---------------------------------------------------------------------
frontend openshift-api-server
    bind *:6443
    default_backend openshift-api-server
    mode tcp
    option tcplog

backend openshift-api-server
    balance source
    mode tcp
    server bootstrap ${bootstrap_ip}:6443 check
    server master1 ${master_ip}:6443 check

frontend machine-config-server
    bind *:22623
    default_backend machine-config-server
    mode tcp
    option tcplog

backend machine-config-server
    balance source
    mode tcp
    server bootstrap ${bootstrap_ip}:22623 check
    server master1 ${master_ip}:22623 check

frontend ingress-http
    bind *:80
    default_backend ingress-http
    mode tcp
    option tcplog

backend ingress-http
    balance source
    mode tcp
    server master1 ${master_ip}:80 check

frontend ingress-https
    bind *:443
    default_backend ingress-https
    mode tcp
    option tcplog

backend ingress-https
    balance source
    mode tcp
    server master1 ${master_ip}:443 check
```

> bootstrap의 경우 bootstrap 구성이 완료되면, 주석 처리 또는 해당 내용 제거 후 haproxy를 재기동합니다.

**7-3) 방화벽 Port 오픈**

```bash
* SELINUX 설정 (HTTP Listener로 정의되지 않은 Port에 대해 SELINUX 권한 허용)
semanage port -a -t http_port_t -p tcp 6443
semanage port -a -t http_port_t -p tcp 22623
```

```bash
firewall-cmd --permanent --add-port=22623/tcp
firewall-cmd --permanent --add-port=6443/tcp
firewall-cmd --permanent --add-service=https
firewall-cmd --permanent --add-service=http

firewall-cmd --permanent --add-port=80/tcp
firewall-cmd --permanent --add-port=443/tcp

firewall-cmd --add-port 22623/tcp --zone=internal --perm
firewall-cmd --add-port 6443/tcp --zone=internal --perm  
firewall-cmd --add-service https --zone=internal --perm  
firewall-cmd --add-service http --zone=internal --perm  
 
firewall-cmd --add-port 6443/tcp --zone=external --perm  
firewall-cmd --add-service https --zone=external --perm  
firewall-cmd --add-service http --zone=external --perm  
firewall-cmd --complete-reload
```

**7-4) firewall port 확인**

```bash
[root@bastion haproxy]# firewall-cmd --list-all
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: eth0 eth1
  sources:
  services: dhcpv6-client http https ssh
  ports: 8080/tcp 53/tcp 53/udp 22623/tcp 6443/tcp 80/tcp 443/tcp
  protocols:
  masquerade: yes
  forward-ports:
  source-ports:
  icmp-blocks:
  rich rules:
```

### 7-5) haproxy 서비스 등록 및 시작

```bash
systemctl enable haproxy
systemctl start haproxy 
systemctl status haproxy
```

### 8. Chrony 설정

모든 서버의 시간 동기화를 위해 Chrony 설정이 필요하며, 시간이 맞지 않을 경우 데이터 수집시 시간이 맞지 않아서 모니터링 대시보드가 제대로 보이지 않을 가능성이 있습니다.

**8-1) Chrony 설치**

Bastion 서버에 chrony 를 설치합니다.

```bash
$ yum install -y chrony 
```

**8-2) chrony 설정 파일 수정 (/etc/chrony.conf)**

```bash
# Use public servers from the pool.ntp.org project.
# Please consider joining the pool (http://www.pool.ntp.org/join.html).
server bastion.single.demo.com iburst // chrony 서버 설정

# Record the rate at which the system clock gains/losses time.
driftfile /var/lib/chrony/drift

# Allow the system clock to be stepped in the first three updates
# if its offset is larger than 1 second.
makestep 1.0 3

# Enable kernel synchronization of the real-time clock (RTC).
rtcsync

# Enable hardware timestamping on all interfaces that support it.
#hwtimestamp *

# Increase the minimum number of selectable sources required to adjust
# the system clock.
#minsources 2

# Allow NTP client access from local network.
allow xx.xxx.xx.x/24 // 내부 네트워크에서 이 서버를 타임서버로 참조하기 위한 설정

# Serve time even if not synchronized to a time source.
#local stratum 10
local stratum 3 // 값을 3으로 변경

# Specify file containing keys for NTP authentication.
#keyfile /etc/chrony.keys

# Specify directory for log files.
logdir /var/log/chrony

# Select which information is logged.
#log measurements statistics tracking
```

**8-3) chrony 서비스 등록 및 시작 (Basiton)**

```bash
$ systemctl enable chronyd 
$ systemctl start chronyd 
$ systemctl status chronyd 
```

**8-4) NTP Port 방화벽 해제** 

```bash
$ firewall-cmd --perm --add-service=ntp
$ firewall-cmd --reload
$ firewall-cmd --perm --add-port=123/tcp
$ firewall-cmd --perm --add-port=123/udp
$ firewall-cmd --reload
```

**8-5) chrony service 확인**

```bash
[root@bastion ~]# chronyc sources -v
210 Number of sources = 1

  .-- Source mode  '^' = server, '=' = peer, '#' = local clock.
 / .- Source state '*' = current synced, '+' = combined , '-' = not combined,
| /   '?' = unreachable, 'x' = time may be in error, '~' = time too variable.
||                                                 .- xxxx [ yyyy ] +/- zzzz
||      Reachability register (octal) -.           |  xxxx = adjusted offset,
||      Log2(Polling interval) --.      |          |  yyyy = measured offset,
||                                \     |          |  zzzz = estimated error.
||                                 |    |           \
MS Name/IP address         Stratum Poll Reach LastRx Last sample
===============================================================================
^* bastion.single.demo.com       3  10   377  184m  -2907ns[  -15us] +/-   21us
// *부분이 ?이면 안 된다.
// 다른 서버에서는 해당 부분이 hostname 대신 IP 또는 Hostname으로 보여질 것임
```

### 9. ssh-key 생성

설치를 진행할 bastion 서버에서 ssh key 생성과 ssh-agent에 key를 등록합니다.

- Cluster 설치 디버깅 또는 재해 복구를 위해서는 `ssh-agent` 및 설치 프로그램 모두 `ssh key`를 제공해야 함

- 운영 환경에서는 재해 복구 및 디버깅이 필요

- ==`ssh-key`를 사용하여 bastion node에서 다른 node로 `core` user로 SSH로 접속 할 수 있음== 

  ```bash
  [root@bastion ~]# ssh-keygen -t rsa -b 4096
  Generating public/private rsa key pair.
  Enter file in which to save the key (/root/.ssh/id_rsa):
  Created directory '/root/.ssh'.
  Enter passphrase (empty for no passphrase):
  Enter same passphrase again:
  Your identification has been saved in /root/.ssh/id_rsa.
  Your public key has been saved in /root/.ssh/id_rsa.pub.
  The key fingerprint is:
  SHA256:WZAdcAjzi01VWm5REhcPGQhR0+llDSuZPtFRah3QuDA root@bastion.ocp46.test.com
  The key's randomart image is:
  +---[RSA 4096]----+
  |      o.o=**O*%Oo|
  |       oo+.E.X+O=|
  |        o o X.*oo|
  |       + + o =.  |
  |      . S   o    |
  |             .   |
  |                 |
  |                 |
  |                 |
  +----[SHA256]-----+
  [root@bastion haproxy]# eval "$(ssh-agent -s)"
  Agent pid 19812
  [root@bastion haproxy]# ssh-add ~/.ssh/id_rsa
  Identity added: /root/.ssh/id_rsa (/root/.ssh/id_rsa)
  ```

### 10. Install 사전 확인

**10-1) IP 확인**

```bash
[root@bastion ~]# ip addr
```

**10-2) DNS 서비스 확인**

```bash
$ systemctl status named
```

**10-3) Haproxy 서비스 확인**

```bash
$ systemctl status haproxy
```

**10-4) chrony 서비스 확인**

``` bash
$ systemctl status chronyd
```



### 11. install-config.yaml 생성

**11-1) Install에 필요한 파일 다운로드**

- rhcos iso 파일, openshift-install, openshift-client 다운로드  (/var/www/html)

  ```bash
  wget -P /var/www/html https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.9/latest/rhcos-4.9.0-x86_64-metal.x86_64.raw.gz
  wget -P /var/www/html https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.9.8/openshift-install-linux-4.9.8.tar.gz
  wget -P /var/www/html https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.9.8/openshift-client-linux-4.9.8.tar.gz 
  ```

**11-2) 파일 압축 해제**

- openshift-install, openshift-client 파일 압축 해제

  ```bash
  tar -xzf /var/www/html/openshift-client-linux-4.9.8.tar.gz -C /usr/local/bin/
  tar -xzf /var/www/html/openshift-install-linux-4.9.8.tar.gz -C /usr/local/bin/
  ```

**11-3) OpenShift CLI를 위한 bash 환경 변수 설정**

- 환경 변수 설정

  ```bash
  oc completion bash >/etc/bash_completion.d/openshift
  ```

**11-4) `rhcos-4.9.0-x86_64-metal.x86_64.raw.gz`파일 이름 변경**

설치 시 원활한 진행을 위해 파일 이름을 짧게 변경하여 사용하시는 것이 좋습니다.

```bash
mv rhcos-4.9.0-x86_64-metal.x86_64.raw.gz bios.raw.gz
```

**11-5) install-config.yaml 생성**

OpenShift를 설치할 디렉토리를 생성한 후 `install-config.yaml`을 작성합니다.

본 가이드의 설치 디렉토리는 `/root/ocp49` 이며, 설치 실패 시 디렉토리 재 사용은 금지입니다. 재 사용하고 싶으신 경우 기존 디렉토리를 백업 또는 다른 이름으로 변경하고 디렉토리를 새로 생성하여 진행해야 합니다.

`install-config.yaml`은 OpenShift Cluster 구성을 위한 ignition 파일을 생성하는 데 필요하며, 해당 파일은 ignition 파일을 생성하고 삭제되므로 반드시 백업을 한 후 작업을 하는 것이 좋습니다.

- 생성된 ignition config 파일은 24시간 동안 유효한 인증서를 포함하고 있어서 반드시 24시간 내에 OpenShift Cluster 구성을 완료해야 합니다.

- 또한, 설치가 완료된 이후에도 가급적이면 24시간 내에 master node는 종료하지 않는 것이 좋습니다.

- 24시간 이후에 master node의 인증서가 갱신 되기 때문입니다.

  - install-config.yaml

    ```yaml
    apiVersion: v1
    baseDomain: demo.com
    compute:
    - hyperthreading: Enabled
      name: worker
      replicas: 0
      skipMachinePools: true
    controlPlane:
      hyperthreading: Enabled
      name: master
      replicas: 1
    metadata:
      name: single
    networking:
      clusterNetwork:
      - cidr: 10.128.0.0/14
        hostPrefix: 23
      networkType: OpenShiftSDN
      serviceNetwork:
      - 172.30.0.0/16
    platform:
      none: {}
    fips: false
    pullSecret: '{"auths":{"cloud.openshift.com":{"auth":"b3BlbnNoaWZ0LXJlbGVhc2UtZGV2K3JobnN1cHBvcnRoeW91MXNhOW9md2d2cTVzNnpiZWc1cHBwMzNzdnBuOlZERUpCT01GSkNXUUcySk0yWEFGNDJYS0NXUUpXOUVPV1o4MTI3Nk45UElVT0YzUVRLU1hPTUEwMVMySUpXWEQ=","email":"hyou@redhat.com"},"quay.io":{"auth":"b3BlbnNoaWZ0LXJlbGVhc2UtZGV2K3JobnN1cHBvcnRoeW91MXNhOW9md2d2cTVzNnpiZWc1cHBwMzNzdnBuOlZERUpCT01GSkNXUUcySk0yWEFGNDJYS0NXUUpXOUVPV1o4MTI3Nk45UElVT0YzUVRLU1hPTUEwMVMySUpXWEQ=","email":"hyou@redhat.com"},"registry.connect.redhat.com":{"auth":"NTI4MzgyMTF8dWhjLTFTYTlPRldHdlE1czZ6YkVnNVBwcDMzU1ZQTjpleUpoYkdjaU9pSlNVelV4TWlKOS5leUp6ZFdJaU9pSTBZek5sTVRJMFpUQmpZbU0wTWpKbFlUSXhObVkyT1dNMk1qQTROR1EyWWlKOS5vVEtoLXh3dUVtTUx1by12T0NmTHRwbkprQlBTVm1xdUtZUERYT3lndXdmMHVfWU1VTWlhcDU5bzR5LVFhYmthUl9JVThWaTFqeHo0QmpTOXFiNGk4S2NNYzVDb0c4SDJiY0VnV0JWNDFXcTNScGhfSzA1WmdPWDRhaFJRU1NIN2NGYUt5d3l0eW9LWU9tREVZTkZybGdrZjhpUnF3ZjN0WnpFYTU3dWdOTTQ1ajZYUzlqZHhYR2Jab2pGWGdsUHZ5T1pHMjI3eHZoMkR6bk5WdmYwMVE3MXBlcV9aU3VEYmpYYkFKQ3E3LURIMnhaYkMzZXBMOHBnN1R4ZDlrSFZuek9WSHJtRnQzUDVyeEVQTHhEcUlpQVlUQmE5LXJwOWpwdHl5a1VoYW1vbEhSYW50MkhEZ0Vac0lidDVjcFg4YTEtYVNsT2NaQUF2R1FRam1ZV1FDTzRZRFhpWXlINUNVMWY1ZnFiUkxva0NpeXUtVHNDTE9TdU9qRnN6Z3VYbzIzaU5QUVlDMmtfWWFJWENBa1VVYnVIQm5LcUhPQXVsa1NvRk1uanlzU0l6SmZWMXdvcFpMM050bFVSVnlsek5fbVJpWVJJcmxlZHRCN01Vdy1Tb2drMlpSLXUyREFHTEIxSmd6ZFQteF9qZnRpWVpZc0lETUk2VWlGb0J3ZjVLaWZycXVTYklqUWRPdC01MEw5VjQ3ZGZ6M3pxcDd0OUNfSEFzLTZBbVNvVHdETW9McWJVTW5VejlHbVdEdk15eHlCZ1U5NTlUX2RaTmhKV3pQUHlRR2JuME9heXRxNEpnYjBVc3JEdTItY2RwanBLQ2NmRnZBal93RE9odlkxYkVNSDlXWkR4WThyY3NVcGJ2MGhLNjJrYUctN0p6RGdVQjFuZGRWOWYyYmVHaw==","email":"hyou@redhat.com"},"registry.redhat.io":{"auth":"NTI4MzgyMTF8dWhjLTFTYTlPRldHdlE1czZ6YkVnNVBwcDMzU1ZQTjpleUpoYkdjaU9pSlNVelV4TWlKOS5leUp6ZFdJaU9pSTBZek5sTVRJMFpUQmpZbU0wTWpKbFlUSXhObVkyT1dNMk1qQTROR1EyWWlKOS5vVEtoLXh3dUVtTUx1by12T0NmTHRwbkprQlBTVm1xdUtZUERYT3lndXdmMHVfWU1VTWlhcDU5bzR5LVFhYmthUl9JVThWaTFqeHo0QmpTOXFiNGk4S2NNYzVDb0c4SDJiY0VnV0JWNDFXcTNScGhfSzA1WmdPWDRhaFJRU1NIN2NGYUt5d3l0eW9LWU9tREVZTkZybGdrZjhpUnF3ZjN0WnpFYTU3dWdOTTQ1ajZYUzlqZHhYR2Jab2pGWGdsUHZ5T1pHMjI3eHZoMkR6bk5WdmYwMVE3MXBlcV9aU3VEYmpYYkFKQ3E3LURIMnhaYkMzZXBMOHBnN1R4ZDlrSFZuek9WSHJtRnQzUDVyeEVQTHhEcUlpQVlUQmE5LXJwOWpwdHl5a1VoYW1vbEhSYW50MkhEZ0Vac0lidDVjcFg4YTEtYVNsT2NaQUF2R1FRam1ZV1FDTzRZRFhpWXlINUNVMWY1ZnFiUkxva0NpeXUtVHNDTE9TdU9qRnN6Z3VYbzIzaU5QUVlDMmtfWWFJWENBa1VVYnVIQm5LcUhPQXVsa1NvRk1uanlzU0l6SmZWMXdvcFpMM050bFVSVnlsek5fbVJpWVJJcmxlZHRCN01Vdy1Tb2drMlpSLXUyREFHTEIxSmd6ZFQteF9qZnRpWVpZc0lETUk2VWlGb0J3ZjVLaWZycXVTYklqUWRPdC01MEw5VjQ3ZGZ6M3pxcDd0OUNfSEFzLTZBbVNvVHdETW9McWJVTW5VejlHbVdEdk15eHlCZ1U5NTlUX2RaTmhKV3pQUHlRR2JuME9heXRxNEpnYjBVc3JEdTItY2RwanBLQ2NmRnZBal93RE9odlkxYkVNSDlXWkR4WThyY3NVcGJ2MGhLNjJrYUctN0p6RGdVQjFuZGRWOWYyYmVHaw==","email":"hyou@redhat.com"},"ext-registry.rhocp4.nirs.go.kr:5000":{"auth":"YWRtaW46cjNkaDR0MSE="}}}'
    sshKey: 'ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQCfdX2bXY7B6M4NJpzXFM8e4nYtETt2VLlNIUr4Ki6/Hzu6JFVti3nSYTizW8dzZc7bf/ro2cnpRYZtzpGwiOzaiUpNy/Wr+sdl1HzcRNOvxCOHiPMkBxAhnrG1EFSHgpv1f+5BosBlCPQh6oh58i4OyHE78+5IqHyCCtNSm028WYphTj45D6FbFo6hx65UpR+66HepPiYRELhbydpN1IM9C95qNLUWW07jwqQBc1M7j+nYTWSBnwyrjD25CEmrscbXqVPeEqXUtsLl3oH/vvuexWngtNoLabFszyMjBOYj+kE7PrUIHxatj9yNILAGJyi+q4eMUKH7fISjqa94gzlCMaPDhIJYK7Vk7QQNBE5g/j6FgeekUzQBDzjRuW0P/zGirlBHF88LImXpjIrUYxQpBKtrViTcKOPLebjHC3Op9kAqEfyPYFA7xdR2rYl5M4DhJoH6t/hOU52B2umbtl4lcSCvAqxRoVftXGO5SiSgNPP/yhnWSR5xR9+SHy4eReiZD6H53mUTD7TqMCdd4camV9pbFMdDtoyjMJFIVFQNBfdYvbr6iy7m+IfefLpCJFNU91aIg5KJqiJmLTNHfB9bp+DabbFlhlu5cSaIticqRvDx92+MUfzvIJEbfpZIby2jBc3oAb6F5UXziJwWvVgwIwQkEnRpoM0yOcSZXU31pQ== root@bastion.single.demo.com'
    ```



### 12. Ignition 파일 생성

**12-1) install-config.yaml 생성**

-  생성된 ignition config 파일은 24시간 동안 유효한 인증서를 포함하고 있어서 반드시 24시간 내에 OpenShift Cluster 구성을 완료해야 합니다.

- ignition 파일을 생성하면 `install-config.yaml`은 삭제되므로 작업 전에 백업을 하고 진행합니다.

  ```bash
  [root@bastion ocp]# cp install-config.yaml install-config.yaml_bak
  ```

**12-2) Kubernetes manifest file 생성**

Single Node Cluster의 경우 Master가 Worker Node의 역할까지 포함하고 있으므로 `masterSchedulable` 값을 변경하지 않고 진행합니다.

- Kubernetes manifest file 생성

  ```bash
  $ openshift-install create manifests --dir=/root/ocp49
  ```

**12-3) ignition config 파일 생성**

- ignition config 파일 생성

  ```bash
  [root@bastion ocp]# openshift-install create ignition-configs --dir=/root/ocp49
  INFO Consuming Openshift Manifests from target directory
  INFO Consuming Master Machines from target directory
  INFO Consuming OpenShift Install (Manifests) from target directory
  INFO Consuming Worker Machines from target directory
  INFO Consuming Common Manifests from target directory
  INFO Ignition-Configs created in: /root/ocp49 and /root/ocp49/auth
  ```

- ignition 파일을 `/var/www/html` 위치로 복사

  ```bash
  $ cp *.ign /var/www/html
  $ chmod 755 /var/www/html/*.ign
  ```



### 13. 노드 구성

Single Node Cluster 구성 시에도 초기 구성을 위해 bootstrap node를 시작으로 구성됩니다. 이후 master node를 설치 할 때, TAB Key를 눌러서 다음과 같이 명령어를 입력하고 설치를 시작하며, 명령어는 한 줄로 입력되어야 합니다.

- 설치 명령어 예시

  ```bash
  coreos.inst.insecure coreos.inst.install_dev=sda coreos.inst.image_url=http://${BASTION_IP}:8080/bios.raw.gz
  coreos.inst.ignition_url=http://${BASTION_IP}:8080/ign/bootstrap.ign
  ip=${BOOTSTRAP_IP}::${DNS_SERVER_IP}:{$NETMASK}:bootstrap.single.demo.com:ens3:none nameserver=${DNS_SERVER_IP}
  ```

- bootstrap node가 정상적으로 올라오면, bootstrap 서버에 접속하여 로그를 확인할 수 있습니다. 이후 master node를 설치 및 구성합니다.

- KUBECONFIG 환경 변수 설정

  ```bash
  export KUBECONFIG=/root/ocp49/auth/kubeconfig
  ```

- bootstrap complete install status monitoring command

  ``` bash
  openshift-install wait-for bootstrap-complete --dir=/root/ocp49 --log-level debug
  ```

  - bootstrap mornitoring log message

    ``` bash
    [root@bastion ~]# openshift-install wait-for bootstrap-complete --dir=/root/ocp49 --log-level debug
    openshift-install wait-for bootstrap-complete --dir=/root/ocp49 --log-level debug
    DEBUG OpenShift Installer 4.9.8
    DEBUG Built from commit 1c538b8949f3a0e5b993e1ae33b9cd799806fa93
    INFO Waiting up to 20m0s for the Kubernetes API at https://api.single.demo.com:6443...
    INFO API v1.22.2+c8538fc up
    INFO Waiting up to 30m0s for bootstrapping to complete...
    DEBUG Bootstrap status: complete
    INFO It is now safe to remove the bootstrap resources
    DEBUG Time elapsed per stage:
    DEBUG Bootstrap Complete: 5m14s
    INFO Time elapsed: 5m14s
    ```

- boostrap 및 node  SSH 접속 방법

  ```bash
  $ core ssh@bootstrap // 다른 노드의 경우 다른 노드 호스트명으로 접속
  ```

- bootstrap node log 확인

  ```bash
  $ journalctl -b -f -u release-image.service -u bootkube.service
  ```

### 14. Initial Cluster Operator Configuration

Cluster 구성이 시작되면 Operator가 정상적으로 설치되고 있는지 확인할 수 있습니다.

- Watch the cluster components come online:

  ```bash
  [root@bastion ~]# oc get co
  NAME                                       VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE   MESSAGE
  authentication                             4.9.8     True        False         False      90s
  baremetal                                  4.9.8     True        False         False      25m
  cloud-controller-manager                   4.9.8     True        False         False      33m
  cloud-credential                           4.9.8     True        False         False      52m
  cluster-autoscaler                         4.9.8     True        False         False      29m
  config-operator                            4.9.8     True        False         False      31m
  console                                    4.9.8     True        False         False      64s
  csi-snapshot-controller                    4.9.8     True        False         False      30m
  dns                                        4.9.8     True        False         False      26m
  etcd                                       4.9.8     True        False         False      28m
  image-registry                             4.9.8     True        False         False      24m
  ingress                                    4.9.8     True        False         False      22m
  insights                                   4.9.8     True        False         False      24m
  kube-apiserver                             4.9.8     True        False         False      25m
  kube-controller-manager                    4.9.8     True        False         False      29m
  kube-scheduler                             4.9.8     True        False         False      29m
  kube-storage-version-migrator              4.9.8     True        False         False      30m
  machine-api                                4.9.8     True        False         False      29m
  machine-approver                           4.9.8     True        False         False      29m
  machine-config                             4.9.8     True        False         False      23m
  marketplace                                4.9.8     True        False         False      29m
  monitoring                                 4.9.8     True        False         False      15m
  network                                    4.9.8     True        False         False      31m
  node-tuning                                4.9.8     True        False         False      29m
  openshift-apiserver                        4.9.8     True        False         False      24m
  openshift-controller-manager               4.9.8     True        False         False      30m
  openshift-samples                          4.9.8     True        False         False      25m
  operator-lifecycle-manager                 4.9.8     True        False         False      29m
  operator-lifecycle-manager-catalog         4.9.8     True        False         False      30m
  operator-lifecycle-manager-packageserver   4.9.8     True        False         False      27m
  service-ca                                 4.9.8     True        False         False      31m
  storage                                    4.9.8     True        False         False      31m
  ```

- openshift installation complete message

  `bootstrap` node 설치가 완료되면 install-complete command를 사용하여 전체 구성에 대한 상황을 모니터링 할 수 있습니다.

  ```bash
  [root@bastion ~]# openshift-install wait-for install-complete --dir=/root/ocp49 --log-level debug
  DEBUG OpenShift Installer 4.9.8
  DEBUG Built from commit 1c538b8949f3a0e5b993e1ae33b9cd799806fa93
  DEBUG Loading Install Config...
  DEBUG   Loading SSH Key...
  DEBUG   Loading Base Domain...
  DEBUG     Loading Platform...
  DEBUG   Loading Cluster Name...
  DEBUG     Loading Base Domain...
  DEBUG     Loading Platform...
  DEBUG   Loading Networking...
  DEBUG     Loading Platform...
  DEBUG   Loading Pull Secret...
  DEBUG   Loading Platform...
  DEBUG Using Install Config loaded from state file
  INFO Waiting up to 40m0s for the cluster at https://api.single.demo.com:6443 to initialize...
  DEBUG Cluster is initialized
  INFO Waiting up to 10m0s for the openshift-console route to be created...
  DEBUG Route found in openshift-console namespace: console
  DEBUG OpenShift console route is admitted
  INFO Install complete!
  INFO To access the cluster as the system:admin user when using 'oc', run 'export KUBECONFIG=/root/ocp49/auth/kubeconfig'
  INFO Access the OpenShift web-console here: https://console-openshift-console.apps.single.demo.com
  INFO Login to the console with user: "kubeadmin", and password: "tUYyX-DHVbL-JskDu-Ncg2W"
  INFO Time elapsed: 0s
  ```

  설치가 완료되면 OpenShift Console, API 접속 정보 및 기본 계정인 `kubeadmin` 계정의 패스워드를 함께 확인 할 수 있습니다.


### 15. Logging in to the Cluster

Cluster kubeconfig 파일을 export해서 기본 시스템 사용자로 Cluster에 로그인 할 수 있습니다. kubeconfig 파일에는 CLI 클라이언트를 올바른 Cluster 및 API 서버에 연결하는 데 사용하는 Cluster에 대한 정보가 포함되어 있습니다. 이 파일은 Cluster에 따라 다르며 OpenShift Container Platform 설치 중에 생성됩니다.

- KUBECONFIG 환경 변수 설정

  ```bash
  export KUBECONFIG=/root/ocp49/auth/kubeconfig
  ```

- oc 명령이 정상적으로 실행되는지 확인

  ```bash
  $ oc whoami
  system:admin
  ```

  > 해당 `oc whoami` 명령은 Cluster 구성이 완료되어야 정상적으로 확인 가능합니다.



### 16. Console 접속

Console 접속을 위해서 hosts 파일에 내용을 추가합니다.

- hosts 파일 내용 추가

  ```bash
  ${Haproxy_IP} console-openshift-console.apps.single.demo.com 
  ${Haproxy_IP} oauth-openshift.apps.single.demo.com 
  ```

- Console 접속

  kubeadmin 계정으로 로그인하면 다음과 같이 Single Node로 구성된 Cluster를 확인할 수 있습니다.

  ![01_single_node_cluster_console](https://github.com/justone0127/OpenShift-Single-Node-Cluster/blob/main/01_single_node_cluster_console.png)



  





