### elk7.15.2部署.

#### 一、环境检查

```javascript
echo "vm.swappiness = 0" >>/etc/sysctl.conf
echo "vm.max_map_count = 655360" >>/etc/sysctl.conf
echo "net.core.somaxconn = 2048" >>/etc/sysctl.conf
echo "*       soft    nofile  655360" >>/etc/security/limits.conf
echo "*       hard    nofile  655360" >>/etc/security/limits.conf
sed -i 's/#DefaultLimitNPROC=/DefaultLimitNPROC=655360/' /etc/systemd/system.conf
sed -i 's/#DefaultLimitNPROC=/DefaultLimitNPROC=655360/' /etc/systemd/user.conf
```

#### 二、安装jdk、es、kibana.

```javascript
#elasticsearch源
cat <<EOF | sudo tee /etc/yum.repos.d/elasticsearch.repo
[elasticsearch]
name=Elasticsearch repository for 7.x packages
baseurl=https://artifacts.elastic.co/packages/7.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=0
autorefresh=1
type=rpm-md
EOF

#kibana源
cat <<EOF | sudo tee /etc/yum.repos.d/kibana.repo
[kibana-7.x]
name=Kibana repository for 7.x packages
baseurl=https://artifacts.elastic.co/packages/7.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
EOF

#安装
yum install java-1.8.0-openjdk elasticsearch kibana logstash filebeat -y
```

#### 三、配置

```javascript
[root@node1 ~]# grep -Ev '^#|^$' /etc/elasticsearch/elasticsearch.yml 
cluster.name: node1
node.name: node1
path.data: /var/lib/elasticsearch
path.logs: /var/log/elasticsearch
network.host: 0.0.0.0
http.port: 9200
discovery.seed_hosts: ["172.27.0.6"]
cluster.initial_master_nodes: ["node1"]


[root@node1 ~]# grep -Ev '^#|^$' /etc/kibana/kibana.yml
server.port: 5601
server.host: "0.0.0.0"
elasticsearch.hosts: ["http://172.27.0.6:9200"]



#输入
cat <<EOF | sudo tee /etc/logstash/conf.d/02-beats-input.conf
input {
  beats {
    port => 5044
  }
}
EOF

#文件
cat <<EOF | sudo tee /etc/logstash/conf.d/10-syslog-filter.conf
filter {
  if [fileset][module] == "system" {
    if [fileset][name] == "auth" {
      grok {
        match => { "message" => ["%{SYSLOGTIMESTAMP:[system][auth][timestamp]} %{SYSLOGHOST:[system][auth][hostname]} sshd(?:\[%{POSINT:[system][auth][pid]}\])?: %{DATA:[system][auth][ssh][event]} %{DATA:[system][auth][ssh][method]} for (invalid user )?%{DATA:[system][auth][user]} from %{IPORHOST:[system][auth][ssh][ip]} port %{NUMBER:[system][auth][ssh][port]} ssh2(: %{GREEDYDATA:[system][auth][ssh][signature]})?",
                  "%{SYSLOGTIMESTAMP:[system][auth][timestamp]} %{SYSLOGHOST:[system][auth][hostname]} sshd(?:\[%{POSINT:[system][auth][pid]}\])?: %{DATA:[system][auth][ssh][event]} user %{DATA:[system][auth][user]} from %{IPORHOST:[system][auth][ssh][ip]}",
                  "%{SYSLOGTIMESTAMP:[system][auth][timestamp]} %{SYSLOGHOST:[system][auth][hostname]} sshd(?:\[%{POSINT:[system][auth][pid]}\])?: Did not receive identification string from %{IPORHOST:[system][auth][ssh][dropped_ip]}",
                  "%{SYSLOGTIMESTAMP:[system][auth][timestamp]} %{SYSLOGHOST:[system][auth][hostname]} sudo(?:\[%{POSINT:[system][auth][pid]}\])?: \s*%{DATA:[system][auth][user]} :( %{DATA:[system][auth][sudo][error]} ;)? TTY=%{DATA:[system][auth][sudo][tty]} ; PWD=%{DATA:[system][auth][sudo][pwd]} ; USER=%{DATA:[system][auth][sudo][user]} ; COMMAND=%{GREEDYDATA:[system][auth][sudo][command]}",
                  "%{SYSLOGTIMESTAMP:[system][auth][timestamp]} %{SYSLOGHOST:[system][auth][hostname]} groupadd(?:\[%{POSINT:[system][auth][pid]}\])?: new group: name=%{DATA:system.auth.groupadd.name}, GID=%{NUMBER:system.auth.groupadd.gid}",
                  "%{SYSLOGTIMESTAMP:[system][auth][timestamp]} %{SYSLOGHOST:[system][auth][hostname]} useradd(?:\[%{POSINT:[system][auth][pid]}\])?: new user: name=%{DATA:[system][auth][user][add][name]}, UID=%{NUMBER:[system][auth][user][add][uid]}, GID=%{NUMBER:[system][auth][user][add][gid]}, home=%{DATA:[system][auth][user][add][home]}, shell=%{DATA:[system][auth][user][add][shell]}$",
                  "%{SYSLOGTIMESTAMP:[system][auth][timestamp]} %{SYSLOGHOST:[system][auth][hostname]} %{DATA:[system][auth][program]}(?:\[%{POSINT:[system][auth][pid]}\])?: %{GREEDYMULTILINE:[system][auth][message]}"] }
        pattern_definitions => {
          "GREEDYMULTILINE"=> "(.|\n)*"
        }
        remove_field => "message"
      }
      date {
        match => [ "[system][auth][timestamp]", "MMM  d HH:mm:ss", "MMM dd HH:mm:ss" ]
      }
      geoip {
        source => "[system][auth][ssh][ip]"
        target => "[system][auth][ssh][geoip]"
      }
    }
    else if [fileset][name] == "syslog" {
      grok {
        match => { "message" => ["%{SYSLOGTIMESTAMP:[system][syslog][timestamp]} %{SYSLOGHOST:[system][syslog][hostname]} %{DATA:[system][syslog][program]}(?:\[%{POSINT:[system][syslog][pid]}\])?: %{GREEDYMULTILINE:[system][syslog][message]}"] }
        pattern_definitions => { "GREEDYMULTILINE" => "(.|\n)*" }
        remove_field => "message"
      }
      date {
        match => [ "[system][syslog][timestamp]", "MMM  d HH:mm:ss", "MMM dd HH:mm:ss" ]
      }
    }
  }
}
EOF



#输出
cat <<EOF | sudo tee /etc/logstash/conf.d/30-elasticsearch-output.conf
output {
  elasticsearch {
    hosts => ["localhost:9200"]
    manage_template => false
    index => "%{[@metadata][beat]}-%{[@metadata][version]}-%{+YYYY.MM.dd}"
  }
}
EOF

#修改filebeat配置
vi /etc/filebeat/filebeat.yml
171 # ================================== Outputs ===================================
172 
173 # Configure what output to use when sending the data collected by the beat.
174 
175 # ---------------------------- Elasticsearch Output ----------------------------
176 #output.elasticsearch:
177   # Array of hosts to connect to.
178 #  hosts: ["localhost:9200"]
179 
180   # Protocol - either `http` (default) or `https`.
181   #protocol: "https"
182 
183   # Authentication credentials - either API key or username/password.
184   #api_key: "id:api_key"
185   #username: "elastic"
186   #password: "changeme"
187 
188 # ------------------------------ Logstash Output -------------------------------
189 output.logstash:
190   # The Logstash hosts
191   hosts: ["localhost:5044"]
```

#### 四、启动

```javascript
systemctl start elasticsearch
systemctl enable elasticsearch

#添加配置文件密码
vi /etc/elasticsearch/elasticsearch.yml
http.cors.enabled: true
http.cors.allow-origin: "*"
http.cors.allow-headers: Authorization
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true

#设置es密码
sh /usr/share/elasticsearch/bin/elasticsearch-setup-passwords interactive

vi /etc/kibana/kibana.yml
elasticsearch.username: "kibana_system"
elasticsearch.password: "Test@12345"

curl -XGET　-u elastic:Test@12345 '127.0.0.1:9200/_cluster/health?pretty'    #查看状态是否为green.
curl -XGET　-u elastic:Test@12345 '127.0.0.1:9200/_cat/nodes?v'              #查看node.
curl -XGET　-u elastic:Test@12345 '127.0.0.1:9200/_cat/allocation?v&pretty'  #查看存储空间.
systemctl start kibana
systemctl enable kibana
systemctl start logstash
systemctl enable logstash
systemctl start logstash
systemctl enable logstash

#测试配置文件是否正确.
sudo -u logstash /usr/share/logstash/bin/logstash --path.settings /etc/logstash -t


#模块开启.
filebeat modules enable system
filebeat modules list
#添加到es中.
sudo filebeat setup --template -E output.logstash.enabled=false -E 'output.elasticsearch.hosts=["localhost:9200"]'
sudo filebeat setup -e -E output.logstash.enabled=false -E output.elasticsearch.hosts=['localhost:9200'] -E setup.kibana.host=localhost:5601
```

#### 五、访问5601

```javascript
http://127.0.0.1:5601/app/discover    #查看日志.
```


#### 六、Loki日志收集.

```javascript
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm upgrade --install loki grafana/loki-stack \
  --namespace=loki-stack \
  --create-namespace \
  --set grafana.enabled=true \
  --set loki.persistence.enabled=true \
  --set loki.persistence.storageClassName=csi-cephfs-sc \
  --set loki.persistence.size=5Gi
  
  
  
kubectl get secret --namespace loki-stack loki-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
kubectl -n loki-stack patch svc loki-grafana -p '{"spec": {"type": "NodePort"}}'
```


####  kubernetes efk官方部署.
```javascript
git clone https://github.com/kubernetes/kubernetes.git
cd kubernetes/cluster/addons/fluentd-elasticsearch
kubectl apply -f .
kubectl proxy --address='172.27.0.5' --port=5601 --accept-hosts='^*$'
http://172.27.0.5:5601/api/v1/namespaces/logging/services/kibana-logging/proxy
```
