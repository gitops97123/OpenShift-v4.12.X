   
# Haproxy Configuration Node. 

#### Step 1: Create DNF (yum) Server using rhel-dvd. 
 
mount cdrom to /mnt directory. 

    [root@haproxy ~]# mount /dev/cdrom /mnt
    mount: /mnt: WARNING: device write-protected, mounted read-only.

Create yum configuration repository. 
    
    [root@haproxy ~]# cat /etc/yum.repos.d/abc.repo
    [AppStream]
    enabled = 0
    gpgcheck = 0
    baseurl = file:///mnt/AppStream
    name = app stream
    
    [baseos]
    enabled = 0
    gpgcheck = 0
    baseurl = file:///mnt/BaseOS
    name = base

Install haproxy package. 

    [root@haproxy ~]# yum install haproxy -y 

Configuration haproxy.cfg. (Replace the content below in your configuration)  

    [root@haproxy ~]# cat /etc/haproxy/haproxy.cfg
    # Global settings
    #---------------------------------------------------------------------
    global
        maxconn     20000
        log         /dev/log local0 info
        chroot      /var/lib/haproxy
        pidfile     /var/run/haproxy.pid
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
        log                     global
        mode                    http
        option                  httplog
        option                  dontlognull
        option http-server-close
        option redispatch
        option forwardfor       except 127.0.0.0/8
        retries                 3
        maxconn                 20000
        timeout http-request    10000ms
        timeout http-keep-alive 10000ms
        timeout check           10000ms
        timeout connect         40000ms
        timeout client          300000ms
        timeout server          300000ms
        timeout queue           50000ms
    
    # Enable HAProxy stats
    listen stats
        bind :9000
        stats uri /stats
        stats refresh 10000ms
    
    # Kube API Server
    frontend k8s_api_frontend
        bind :6443
        default_backend k8s_api_backend
        mode tcp
    
    backend k8s_api_backend
        mode tcp
        balance source
        server      bootstrap 10.0.0.201:6443 check
        server      master1 10.0.0.202:6443 check
        server      master2 10.0.0.203:6443 check
        server      master3 10.0.0.204:6443 check
    
    # OCP Machine Config Server
    frontend ocp_machine_config_server_frontend
        mode tcp
        bind :22623
        default_backend ocp_machine_config_server_backend
    
    backend ocp_machine_config_server_backend
        mode tcp
        balance source
        server      bootstrap 10.0.0.201:22623 check
        server      master1 10.0.0.202:22623 check
        server      master2 10.0.0.203:22623 check
        server      master3 10.0.0.204:22623 check
    
    # OCP Ingress - layer 4 tcp mode for each. Ingress Controller will handle layer 7.
    frontend ocp_http_ingress_frontend
        bind :80
        default_backend ocp_http_ingress_backend
        mode tcp
    
    backend ocp_http_ingress_backend
        balance source
        mode tcp
        server      master1 10.0.0.202:80 check
        server      master2 10.0.0.203:80 check
        server      master3 10.0.0.204:80 check
    
    frontend ocp_https_ingress_frontend
        bind *:443
        default_backend ocp_https_ingress_backend
        mode tcp
    
    backend ocp_https_ingress_backend
        mode tcp
        balance source
        server      master1 10.0.0.202:443 check
        server      master2 10.0.0.203:443 check
        server      master3 10.0.0.204:443 check

Allow SELinux policy. 
    
    [root@haproxy ~]# setsebool -P haproxy_connect_any on

Start and enable haproxy service. 

    [root@haproxy ~]# systemctl restart haproxy

    [root@haproxy ~]# systemctl status haproxy
    ● haproxy.service - HAProxy Load Balancer
       Loaded: loaded (/usr/lib/systemd/system/haproxy.service; disabled; vendor preset: disabled)
       Active: active (running) since Fri 2023-10-13 09:39:08 IST; 4s ago
      Process: 23102 ExecStartPre=/usr/sbin/haproxy -f $CONFIG -f $CFGDIR -c -q $OPTIONS (code=exited, status=0/SUCCESS)
     Main PID: 23107 (haproxy)
        Tasks: 2 (limit: 100500)
       Memory: 4.9M
       CGroup: /system.slice/haproxy.service
               ├─23107 /usr/sbin/haproxy -Ws -f /etc/haproxy/haproxy.cfg -f /etc/haproxy/conf.d -p /run/haproxy.pid
               └─23109 /usr/sbin/haproxy -Ws -f /etc/haproxy/haproxy.cfg -f /etc/haproxy/conf.d -p /run/haproxy.pid
    
    Oct 13 09:39:08 haproxy.lab.prodevans.com haproxy[23107]: Proxy stats started.
    Oct 13 09:39:08 haproxy.lab.prodevans.com haproxy[23107]: Proxy k8s_api_frontend started.
    Oct 13 09:39:08 haproxy.lab.prodevans.com haproxy[23107]: Proxy k8s_api_backend started.
    Oct 13 09:39:08 haproxy.lab.prodevans.com haproxy[23107]: Proxy ocp_machine_config_server_frontend started.
    Oct 13 09:39:08 haproxy.lab.prodevans.com haproxy[23107]: Proxy ocp_machine_config_server_backend started.
    Oct 13 09:39:08 haproxy.lab.prodevans.com haproxy[23107]: Proxy ocp_http_ingress_frontend started.
    Oct 13 09:39:08 haproxy.lab.prodevans.com haproxy[23107]: Proxy ocp_http_ingress_backend started.
    Oct 13 09:39:08 haproxy.lab.prodevans.com haproxy[23107]: Proxy ocp_https_ingress_frontend started.
    Oct 13 09:39:08 haproxy.lab.prodevans.com haproxy[23107]: Proxy ocp_https_ingress_backend started.
    Oct 13 09:39:08 haproxy.lab.prodevans.com systemd[1]: Started HAProxy Load Balancer.

