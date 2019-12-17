# Loghive
Syslog Server Solution using ELK

After years of using commercial products for syslog solutions I decided I wanted to build something simple and free.

I believe the solution is scalable for small through to large environments.

You can running everything on a single server for a small setup or you can scale out across multiple servers to
provide the resilience and separation your environment may require.

The solution comprises of:

1. Elasticsearch
2. Logstash
3. Kibana

<img src="https://oldhamuk.github.io/loghive.github.io/images/basicflow.png"/>



## Install Guide

The following is a guide to installing everything required to setup Loghive. Please pay attention to the IP Addresses that your are required to enter so the local firewall permits the required traffic on each server.

This guide is for installing each component on separate servers, If you plan to running everything on a single server then you can exclude the 'sudo firewall-cmd' commands list in each section below with the exception of Kibana as those rules will be required to access the Kibana site.


### Elasticsearch

sudo mkdir -p /ouk/install/loghive/  
sudo yum update -y  
sudo yum install wget net-tools java-1.8.0-openjdk -y  
sudo bash -c 'cat > /etc/yum.repos.d/elasticsearch.repo << EOF  
[elasticsearch-7.x]  
name=Elasticsearch repository for 7.x packages  
baseurl=https://artifacts.elastic.co/packages/7.x/yum  
gpgcheck=1  
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch  
enabled=1  
autorefresh=1  
type=rpm-md  
EOF'  
sudo yum update -y  
sudo rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch  
sudo yum install elasticsearch -y  
sudo systemctl start elasticsearch  
sudo systemctl enable elasticsearch  
sudo curl -X GET "localhost:9200"  
sudo firewall-cmd --permanent --zone=public --add-rich-rule='rule family=ipv4 source address=LOGSTASH-IP/32 port port=9200 protocol=tcp  accept'  
sudo firewall-cmd --permanent --zone=public --add-rich-rule='rule family=ipv4 source address=LOGSTASH-IP/32 port port=9300 protocol=tcp  accept'  
sudo firewall-cmd --permanent --zone=public --add-rich-rule='rule family=ipv4 source address=KIBANA-IP/32 port port=9200 protocol=tcp  accept'  
sudo firewall-cmd --permanent --zone=public --add-rich-rule='rule family=ipv4 source address=KIBANA-IP/32 port port=9300 protocol=tcp  accept'  
sudo firewall-cmd --reload  

*** EDIT: /etc/elasticsearch/elasticsearch.yml with the following: ***

---- Cluster ----

cluster.name: loghive  

---- Node ----

node.name: ${HOSTNAME}  
node.master: true  
node.data: true  

---- Network ----

network.host: 0.0.0.0  

---- Discovery ----

discovery.zen.ping.unicast.host: ["ELASTICSEARCH-IP"]  
cluster.initial_master_nodes: ELASTICSEARCH-IP  

---- Various ----

indices.query.bool.max_clause_count: 8192  
search.max_buckets: 100000  

*** SAVE THE FILE ***

sudo systemctl restart elasticsearch  



*********************************
******  Install Completed  ******
*********************************




### Logstash

sudo mkdir -p /ouk/install/loghive/
sudo yum update -y
sudo yum install wget git net-tools java-1.8.0-openjdk -y
sudo bash -c 'cat > /etc/yum.repos.d/logstash.repo << EOF
[logstash-7.x]
name=Elastic repository for 7.x packages
baseurl=https://artifacts.elastic.co/packages/7.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
EOF'
sudo rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
sudo yum update -y
sudo yum install logstash -y
sudo cp -r logstash.service.d /etc/systemd/system/logstash.service.d/
sudo firewall-cmd --permanent --zone=public --add-rich-rule='rule family=ipv4 source address=10.0.0.0/8 port port=514 protocol=tcp  accept'
sudo firewall-cmd --permanent --zone=public --add-rich-rule='rule family=ipv4 source address=172.16.0.0/12 port port=514 protocol=tcp  accept'
sudo firewall-cmd --permanent --zone=public --add-rich-rule='rule family=ipv4 source address=192.168.0.0/16 port port=514 protocol=tcp  accept'
sudo firewall-cmd --permanent --zone=public --add-rich-rule='rule family=ipv4 source address=10.0.0.0/8 port port=514 protocol=udp  accept'
sudo firewall-cmd --permanent --zone=public --add-rich-rule='rule family=ipv4 source address=172.16.0.0/12 port port=514 protocol=udp  accept'
sudo firewall-cmd --permanent --zone=public --add-rich-rule='rule family=ipv4 source address=192.168.0.0/16 port port=514 protocol=udp  accept'
sudo firewall-cmd --reload
sudo setcap cap_net_bind_service=+epi /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.232.b09-0.el7_7.x86_64/jre/bin/java
sudo bash -c 'cat > /etc/logstash/conf.d/syslog.conf << EOF
input {
  tcp {
    port => 514
    type => syslog
  }
  udp {
    port => 514
    type => syslog
  }
}
filter {
syslog_pri { }
}
output {
elasticsearch { hosts => ["ELASTICSEARCH-IP:9200"]
index => "elastilog-hdc-%{+YYYY.MM.dd}" }
}
EOF'
sudo bash -c 'cat > /etc/ld.so.conf.d/java.conf << EOF
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.232.b09-0.el7_7.x86_64/jre/lib/amd64/jli/
EOF'
sudo /sbin/ldconfig
sudo systemctl enable logstash
sudo systemctl start logstash

*********************************
******  Install Completed  ******
*********************************




### Kibana

** Note: In this section you will need to replace YOURSERVERNAME.YOURDOMAIN in the nginx config, the ELASTICSEARCH-IP with the IP of your elasticsearch server in the kibana.yml config.


sudo mkdir -p /ouk/install/loghive/
sudo yum update -y
sudo yum install wget net-tools epel-release -y
sudo yum install tcpdump nginx -y
sudo systemctl start nginx
sudo firewall-cmd --permanent --zone=public --add-service=http
sudo firewall-cmd --permanent --zone=public --add-service=https
sudo firewall-cmd --reload
sudo systemctl enable nginx
sudo bash -c 'cat > /etc/nginx/conf.d/loghive.conf << EOF
server {
    listen 80;
    server_name loghive YOURSERVERNAME.YOURDOMAIN;
    auth_basic "Restricted Access";
    auth_basic_user_file /etc/nginx/htpasswd.users;
    location / {
        proxy_pass http://localhost:5601;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
EOF'
sudo bash -c 'cat > /etc/yum.repos.d/kibana.repo << EOF
[kibana-7.x]
name=Kibana repository for 7.x packages
baseurl=https://artifacts.elastic.co/packages/7.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
EOF'
sudo rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
sudo yum update -y
sudo yum install kibana -y
sudo bash -c 'cat > /etc/kibana/kibana.yml << EOF
elasticsearch.hosts: ["http://ELASTICSEARCH-IP:9200"]
EOF'
sudo systemctl enable kibana
sudo systemctl start kibana
sudo nginx -t
sudo systemctl restart nginx
sudo setsebool httpd_can_network_connect 1 -P

"Setting password for the user 'admin' to gain access to the Kibana portal. Repeat this set chaging the username from admin to your desired name to create extra logins."

admin:`openssl passwd -apr1`" | sudo tee -a /etc/nginx/htpasswd.users
sudo systemctl restart nginx


*********************************
******  Install Completed  ******
*********************************
