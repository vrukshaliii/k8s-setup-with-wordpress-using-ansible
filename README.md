# K8S Setup with Wordpress using Ansible

<img src=https://github.com/Vrukshali-26/k8s-setup-with-wordpress-using-ansible/blob/main/Images/Untitled%20-%20Paint%2015-03-2021%2015_47_17%20(2).png />
<hr>

# What is Kubernetes ?
<img src=https://miro.medium.com/max/735/1*uzmctPmzqUBA427ODBQBIQ.png  align=right width="250px" />
👉🏻 Kubernetes is a portable, extensible, open-source platform for managing containerized workloads and services, that facilitates both declarative configuration and automation.
It has a large, rapidly growing ecosystem. Kubernetes services, support, and tools are widely available. 

## Why you need Kubernetes and what it can do ?
Containers are a good way to bundle and run your applications. In a production environment, you need to manage the containers that run the applications and ensure that there is no downtime. For example, if a container goes down, another container needs to start. Wouldn’t it be easier if this behavior was handled by a system?
That’s how Kubernetes comes to the rescue! Kubernetes provides you with a framework to run distributed systems resiliently. It takes care of scaling and failover for your application, provides deployment patterns, and more. For example, Kubernetes can easily manage a canary deployment for your system. <br />

###  🎡 Kubernetes provides you with: <br />
👉🏻 Service discovery and load balancing  <br />
👉🏻 Storage orchestration   <br />
👉🏻 Automated rollouts and rollbacks   <br />
👉🏻 Automatic bin packing  <br />
👉🏻 Self-healing    <br />
👉🏻 Secret and configuration management   <br />

<hr>

# What is Ansible?
<img src=https://miro.medium.com/max/875/1*vPjuuh6j7zXPE_HURgGenA.png align=right width="250px" /> <br />
Ansible is a configuration management system written in Python using a declarative markup language to describe configurations. It is used to automate software configuration and deployment. <br />

<hr>

# MYSQL :
<img src=https://buddymantra.com/wp-content/uploads/2020/12/mysqlw.jpg align=right width="250px" />
A relational database organizes data into one or more data tables in which data types may be related to each other; these relations help structure the data. SQL is a language programmers use to create, modify and extract data from the relational database, as well as control user access to the database. In addition to relational databases and SQL, an RDBMS like MySQL works with an operating system to implement a relational database in a computer's storage system, manages users, allows for network access and facilitates testing database integrity and creation of backups. <br />

<hr>

# WordPress :
<img src=https://miro.medium.com/max/398/1*6W4k9iFfN01p0mQ3kEK32A.jpeg  align=right />
WordPress is a free and open-source content management system (CMS) written in PHP and paired with a MySQL or MariaDB database. <br />

<hr>
<br />

# Configure K8S Multi Node Cluster over AWS Cloud using Ansible :
 
### 1. Launching EC2 Instances using Ansible :

---> <b>Create Role</b> : `ansible-galaxy init ec2`

---> <b>In tasks file</b> : <br />
```- name: "Launching EC2 Instance"
   ec2:
         key_name: "{{ key_name }}"
         instance_type: "{{ instance_type }}"
         image: "{{ image_id }}"
         wait: yes
         count: "{{ count }}"
         #         instance_tags:
         #       name: "sample_os"
         vpc_subnet_id: "{{ subnet_id }}"
         assign_public_ip: yes
         state: present
         region: "{{ region }}"
         group_id: "{{ sg_group_id }}"
         aws_access_key: "{{ aws_access_key }}"
         aws_secret_key: "{{ aws_secret_key }}"
         instance_tags:
             Name: "{{ item }}"
  loop: "{{ OS_name }}"
```
<br />

---> <b>In vars file</b> : <br />

```aws_access_key: "aws_access_key"
aws_secret_key: "aws_secret_key"
key_name: "testing"
image_id: "ami-038f1ca1bd58a5790"
count: 1
subnet_id: "subnet-52bc140d"
region: "us-east-1"
sg_group_id: "sg-0639b0dd0a69545ea"
instance_type: "t2.micro"
OS_name:
     - "K8S_Master"
     - "K8S_Node1"
     - "K8S_Node2"
```

<br />

---> <b>In playbook setup.yml</b> : <br />
```
- hosts: localhost
  gather_facts: False
  #vars_files: secret.yml
  roles:
  - name: "EC2 Launch"
    role: /root/task23/k8s/ec2/
```

---> <b>Run the playbook</b> : <br />
 
```
ansible-playbook setup.yml
```

<img src=https://github.com/Vrukshali-26/k8s-setup-with-wordpress-using-ansible/blob/main/Images/root%40localhost_~%2015-03-2021%2008_18_05.png />

<hr>

### 2. Setting Up Master Node and Worker Nodes

##### In Master Node,

---> <b>Create Role</b> : `ansible-galaxy init k8s_master` <b4 />

---> <b>In tasks file</b> : <br />
```
- name: "Creating Repo for Kubernetes"
  copy:
          src: kubernetes.repo
          dest: /etc/yum.repos.d/kubernetes.repo

- name: "Installing Software"
  package:
          name: "{{ item }}"
          state: present
  loop: "{{ package_name }}"

- name: "Starting services"
  service:
          name:  "{{ item }}"
          state: started
  loop: "{{ package }}"

- name: "Changing driver to systemd"
  copy:
          src: daemon.json
          dest: /etc/docker/daemon.json

- name: "Restart Docker Services"
  service:
          name: docker
          state: restarted

- name: "Pulling Images"
  shell: kubeadm config images pull

- name: "Bridge to 1"
  shell: echo "1" > /proc/sys/net/bridge/bridge-nf-call-iptables

- name: "kubeadm init"
  shell: kubeadm init --pod-network-cidr=10.244.0.0/16 --ignore-preflight-errors=NumCPU --ignore-preflight-errors=Mem 
  ignore_errors: yes

- name: "Creating .kube directory"
  file:
    path: $HOME/.kube
    state: directory

- name: "Copying file"
  shell: cp  -i  /etc/kubernetes/admin.conf  $HOME/.kube/config

- name: "Changing Owner Permissions"
  shell: chown $(id -u):$(id -g) $HOME/.kube/config

- name: "Setting up Flannel"
  shell: kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
  changed_when: False

- name: "Token Creation"
  shell: kubeadm  token create --print-join-command
  register: token

- name: "Printing Token"
  debug:
         var: token.stdout

```

---> <b>In vars file</b> : <br />
```
package_name:
        - "docker"
        - "kubelet"
        - "kubeadm"
        - "kubectl"
        - "iproute-tc"

package:
        - "docker"
        - "kubelet"
```
<br />

##### In Worker Nodes,

---> <b>Create Role</b> : `ansible-galaxy init k8s_nodes` <b4 />

---> <b>In tasks file</b> : <br />
```
- name: "Creating Repo for Kubernetes"
  copy:
          src: kubernetes.repo
          dest: /etc/yum.repos.d/kubernetes.repo

- name: "Installing Software"
  package:
          name: "{{ item }}"
          state: present
  loop: "{{ package_name }}"

- name: "Starting services"
  service:
          name:  "{{ item }}"
          state: started
  loop: "{{ package }}"

- name: "Changing driver to systemd"
  copy:
          src: daemon.json
          dest: /etc/docker/daemon.json

- name: "Restart Docker Services"
  service:
          name: docker
          state: restarted

- name: "Bridge to 1"
  shell: echo "1" > /proc/sys/net/bridge/bridge-nf-call-iptables

- name: "Using token"
  shell: "{{ token }}"
```
<br />

---> <b>In vars file</b> : <br />
```
package_name:
        - "docker"
        - "kubelet"
        - "kubeadm"
        - "kubectl"
        - "iproute-tc"

package:
        - "docker"
        - "kubelet"
```
<br />

---> <b>In playbook k8s_setup.yml</b> : <br />
```
- hosts: "tag_Name_K8S_Master"
  roles:
  - name: "K8S Master"
    role: /root/k8s/k8s_master



- hosts: ["tag_Name_K8S_Node1", "tag_Name_K8S_Node2"]
  vars_prompt:
    - name: token
      prompt: "Enter token :"
      private: no

  roles:
  - name: "K8S_Nodes"
    role: /root/k8s/k8s_nodes
```
<br />

---> <b>Run the playbook</b> : <br />
 
```
ansible-playbook k8s_setup.yml
```

<img src=https://github.com/Vrukshali-26/k8s-setup-with-wordpress-using-ansible/blob/main/Images/root%40localhost_~_task23_k8s%2015-03-2021%2012_26_21.png />
<br />
<img src=https://github.com/Vrukshali-26/k8s-setup-with-wordpress-using-ansible/blob/main/Images/root%40localhost_~_task23_k8s%2015-03-2021%2012_26_28.png />
<br />
<img src=https://github.com/Vrukshali-26/k8s-setup-with-wordpress-using-ansible/blob/main/Images/root%40localhost_~_task23_k8s%2015-03-2021%2012_26_37.png />
<br />

<hr>

### 3. Launching WordPress and MySQL DataBase Pods

---> <b>Create Role</b> : `ansible-galaxy init wordpress` <b4 />

---> <b>In tasks file</b> : <br />
```
- name: "Launching MYSQL DB Pod"
  shell: kubectl run "{{ db_pod }}" --image "{{ sql_image }}" --env=MYSQL_ROOT_PASSWORD={{ mysql_root_password }} --env=MYSQL_DATABASE={{ database_name }}  --env=MYSQL_USER={{ mysql_user }} --env=MYSQL_PASSWORD={{ mysql_password }}


- name: "Launching WordPress Pod"
  shell: kubectl run "{{ wp_pod }}" --image "{{ wp_image }}"

- name: "Exposing Pod"
  shell: kubectl expose pod "{{ wp_pod }}" --type=NodePort --port=80

- name: "Get svc"
  shell: kubectl get svc
  register: svc

- name: "Print"
  debug:
          var: svc.stdout

- name: "Database IP"
  shell: "kubectl get pods -o wide"
  register: db_ip

- debug:
    var: "db_ip.stdout_lines"
```

<br />

---> <b>In vars file</b> : <br />
```
db_pod: "mydb1"
wp_pod: "mywp1"
sql_image: "mysql:5.7"
wp_image: "wordpress:5.1.1-php7.3-apache"
mysql_root_password: "redhat"
database_name: "mywpdb"
mysql_user: "vrush"
mysql_password: "redhat"
```

---> <b>In playbook wordpress_setup.yml</b> : <br />
```
- hosts: "tag_Name_K8S_Master"
  roles:
  - name: "WordPress Application"
    role: /root/task23/k8s/wordpress
```

---> <b>Run the playbook</b> : <br />
 
```
ansible-playbook wordpress_setup.yml
```

<img src=https://github.com/Vrukshali-26/k8s-setup-with-wordpress-using-ansible/blob/main/Images/root%40localhost_~_task23_k8s%2015-03-2021%2012_26_37.png />
<br />
<img src=https://github.com/Vrukshali-26/k8s-setup-with-wordpress-using-ansible/blob/main/Images/root%40localhost_~_task23_k8s%2015-03-2021%2012_26_42.png />
<br />

### On Browser :

<img src=https://github.com/Vrukshali-26/k8s-setup-with-wordpress-using-ansible/blob/main/Images/Dashboard%20%E2%80%B9%20lw-india%20%E2%80%94%20WordPress%20-%20Google%20Chrome%2015-03-2021%2012_28_13.png />

<br />

# Finally our Task is completed successfully !!!!😄✌🏻 

<img src=https://miro.medium.com/max/393/1*UMxteW13wWr5K_vZroctUg.png />
