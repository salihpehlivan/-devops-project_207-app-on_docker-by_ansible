## Part 1 - Launch Ansible instance

- Create 4 `Red Hat Enterprise Linux 8 (HVM), SSD Volume Type - ami-0b0af3577fe5e3532 (64-bit x86) / ami-01fc429821bf1f4b4 (64-bit Arm)` ami. 

- ansible_control (t2.medium) (tag: Name=ansible_control)


## Part 2 - Prepare the scene

Connect the ansible_control node
   
   - Copy the student files

   - Run the commands below to install Python3 and Ansible. 

```bash
sudo yum update -y
```

```bash
sudo yum install -y python3 
```

```bash
pip3 install --user ansible
```

- Check Ansible's installation with the command below.

```bash
ansible --version
```

- Create ansible directory and change directory to this directory.

```bash
mkdir ansible
cd ansible
```

- Create `ansible.cfg` files.

```
[defaults]
host_key_checking = False
inventory=inventory_aws_ec2.yml
interpreter_python=auto_silent
private_key_file=/home/ec2-user/last_key_pair_211020.pem 
remote_user=ec2-user
```

- copy pem file from local to home directory of ec2-user.

```bash
scp -i <pem-file> <pem-file> ec2-user@<public-ip of ansible_control>:/home/ec2-user
```

- change pem file`s user permission to read-only.

```bash
chmod 400 last_key_pair_211020.pem
```


## Part 3 - Launch Postgresql, Nodejs, React instances

- go to AWS Management Console and select the IAM roles:

- click the  "create role" then create a role with "AmazonEC2FullAccess"

- go to EC2 instance Dashboard, and select the control-node instance

- select actions -> security -> modify IAM role

- select the role thay you have jsut created for EC2 full access and save it.

- install "boto3"

```bash
pip3 install --user boto3
```

```bash
pip3 install --user boto
```

- Tag the postgresql, nodejs and react instances as below in the following YAML file `launch_servers.yml`

```
Name=ansible_postgresql
Name=ansible_nodejs
Name=ansible_react
```

- Tag ansible_control manually and tag ansible_postgresql, ansible_nodejs, ansible_react instances as below in following YAML file `launch_servers.yml`

```
stack=ansible_project
```

- Tag ansible_postgresql, ansible_nodejs, ansible_react instances as below in following YAML file `launch_servers.yml`

```
environment=development
```

- Create `launch_servers.yml` file under the ansible directory. 

```yaml
- name: Create Ec2 instances
  hosts: localhost
  gather_facts: false
  tasks:
  - name: security group rule of Postgre-SQL
    ec2_group:
      name: sec-grp-postgresql
      description: project207 sg for Postgre-SQL
      vpc_id: vpc-0f8cca5fb42fe191f
      region: us-east-1
      rules:
        - proto: tcp
          ports:
            - 22
            - 5432
          cidr_ip: 0.0.0.0/0
          rule_desc: allow all on port 22 and 5432
    tags: ['never', 'ec2-create']
  - name: security group rule of React
    ec2_group:
      name: sec-grp-react
      description:  project207 sg for React
      vpc_id: vpc-0f8cca5fb42fe191f
      region: us-east-1
      rules:
        - proto: tcp
          ports:
            - 22
            - 80
            - 3000
          cidr_ip: 0.0.0.0/0
          rule_desc: allow all on port 22 and 5432
    tags: ['never', 'ec2-create', 'all']
  - name: security group rule of NodeJS
    ec2_group:
      name: sec-grp-nodejs
      description:  project207 sg for NodeJS
      vpc_id: vpc-0f8cca5fb42fe191f
      region: us-east-1
      rules:
        - proto: tcp
          ports:
            - 22
            - 5000
          cidr_ip: 0.0.0.0/0
          rule_desc: allow all on port 22 and 5432
    tags: ['never', 'ec2-create']

  # Block is a Group of Tasks combined together
  - name: Get Info Block
    block:
      - name: Get Running instance Info
        
        ec2_instance_info:
          region: us-east-1
        register: ec2info

      - name: Print info
        debug: var="ec2info.instances"

    # By specifying always on the tag,
    # I let this block to run all the time by module_default
    # this is for security to net create ec2 instances accidentally
    tags: ['always', 'getinfoonly']

  - name: Create EC2 Block
    block:
      - name: Launch ec2 instances
        tags: create_ec2
        ec2:
          region: us-east-1
          key_name: last_key_pair_211020
          group: sec-grp-postgresql
          instance_type: t2.micro
          image: ami-033b95fb8079dc481
          wait: yes
          wait_timeout: 500
          count: 1
          instance_tags:
            Name: ansible_postgresql
            stack: ansible_project
            environment: development
          monitoring: no
          vpc_subnet_id: subnet-095612f7c17c7d890
          assign_public_ip: yes
        register: ec2
        delegate_to: localhost

      - name : Add instance to host group
        add_host:
          hostname: "{{ item.public_ip }}"
          groupname: launched
        loop: "{{ ec2.instances }}"

      - name: Wait for SSH to come up
        local_action:
          module: wait_for
          host: "{{ item.public_ip }}"
          port: 22
          delay: 10
          timeout: 120
        loop: "{{ ec2.instances }}"
    # By specifying never on the tag of this block,
    # I let this block to run only when explicitely being called
    tags: ['never', 'ec2-create']

  - name: Create EC2 Block
    block:
      - name: Launch ec2 instances
        tags: create_ec2
        ec2:
          region: us-east-1
          key_name: last_key_pair_211020
          group: sec-grp-react
          instance_type: t2.micro
          image: ami-033b95fb8079dc481
          wait: yes
          wait_timeout: 500
          count: 1
          instance_tags:
            Name: ansible_react
            stack: ansible_project
            environment: development
          monitoring: no
          vpc_subnet_id: subnet-095612f7c17c7d890
          assign_public_ip: yes
        register: ec2
        delegate_to: localhost

      - name : Add instance to host group
        add_host:
          hostname: "{{ item.public_ip }}"
          groupname: launched
        loop: "{{ ec2.instances }}"

      - name: Wait for SSH to come up
        local_action:
          module: wait_for
          host: "{{ item.public_ip }}"
          port: 22
          delay: 10
          timeout: 120
        loop: "{{ ec2.instances }}"
    # By specifying never on the tag of this block,
    # I let this block to run only when explicitely being called
    tags: ['never', 'ec2-create']

  - name: Create EC2 Block
    block: 
      - name: Launch ec2 instances
        tags: create_ec2
        ec2:
          region: us-east-1
          key_name: last_key_pair_211020
          group: sec-grp-nodejs
          instance_type: t2.micro
          image: ami-033b95fb8079dc481
          wait: yes
          wait_timeout: 500
          count: 1
          instance_tags:
            Name: ansible_nodejs
            stack: ansible_project
            environment: development
          monitoring: no
          vpc_subnet_id: subnet-095612f7c17c7d890
          assign_public_ip: yes
        register: ec2
        delegate_to: localhost

      - name : Add instance to host group
        add_host:
          hostname: "{{ item.public_ip }}"
          groupname: launched
        loop: "{{ ec2.instances }}"

      - name: Wait for SSH to come up
        local_action:
          module: wait_for
          host: "{{ item.public_ip }}"
          port: 22
          delay: 10
          timeout: 120
        loop: "{{ ec2.instances }}"
    # By specifying never on the tag of this block,
    # I let this block to run only when explicitely being called
    tags: ['never', 'ec2-create']
```

```bash
ansible-playbook launch_servers.yaml --tags=ec2-create 
```

## Part 4 - Creating dynamic inventory

- Create `inventory_aws_ec2.yml` file under the ansible directory. 

```yaml
plugin: aws_ec2
regions:
  - "us-east-1"
filters:
  tag:stack: ansible_project
keyed_groups:
  - key: tags.Name
  - key: tags.environment
compose:
  ansible_host: public_ip_address
```

```bash
ansible-inventory -i inventory_aws_ec2.yml --graph
```

```
@all:
  |--@_ansible_control:
  |  |--ec2-3-80-96-146.compute-1.amazonaws.com
  |--@_ansible_nodejs:
  |  |--ec2-3-239-243-194.compute-1.amazonaws.com
  |--@_ansible_postgresql:
  |  |--ec2-3-236-160-236.compute-1.amazonaws.com
  |--@_ansible_react:
  |  |--ec2-3-236-197-117.compute-1.amazonaws.com
  |--@_development:
  |  |--ec2-3-236-160-236.compute-1.amazonaws.com
  |  |--ec2-3-236-197-117.compute-1.amazonaws.com
  |  |--ec2-3-239-243-194.compute-1.amazonaws.com
  |--@aws_ec2:
  |  |--ec2-3-236-160-236.compute-1.amazonaws.com
  |  |--ec2-3-236-197-117.compute-1.amazonaws.com
  |  |--ec2-3-239-243-194.compute-1.amazonaws.com
  |  |--ec2-3-80-96-146.compute-1.amazonaws.com
  |--@ungrouped:
```

- To make sure that all our hosts are reachable with dynamic inventory, we will run various ad-hoc commands that use the ping module.

```bash
$ ansible all -m ping --key-file "~/mykey.pem"
```

## Part 5 - Prepare the playbook files

- Create `ansible-Project` directory under home directory and change directory to this directory.

```bash
mkdir ansible-Project
cd ansible-Project
```

- Create `postgres`, `nodejs`, `react` directories.

```bash
mkdir postgres nodejs react
```

- Copy `~/student_files/todo-app-pern` directory to this directory.

- Change directory to `postgres` directory.

```bash
cd postgres
```

- Copy `init.sql` file from `student_files/todo-app-pern/database` to `postgres` directory.

- Create a Dockerfile

```Dockerfile
FROM postgres

COPY ./init.sql /docker-entrypoint-initdb.d/

EXPOSE 5432
```

- change directory `~/ansible` directory.

```bash
cd ~/ansible
```

- Create a yaml file as postgres playbook and name it `docker_postgre.yml`.

```yaml
- name: Install docker
  gather_facts: No
  any_errors_fatal: true
  hosts: _ansible_postgresql
  become: true
  tasks:
    - name: upgrade all packages
      yum: 
        name: '*'
        state: latest
    # we may need to uninstall any existing docker files from the centos repo first. 
    - name: Remove docker if installed from CentOS repo
      yum:
        name: "{{ item }}"
        state: removed
      with_items:
        - docker
        - docker-client
        - docker-client-latest
        - docker-common
        - docker-latest
        - docker-latest-logrotate
        - docker-logrotate
        - docker-engine
    - name: Install yum utils
      yum:
        name: "{{ item }}"
        state: latest
      with_items:
        - yum-utils
  # yum-utils is a collection of tools and programs for managing yum repositories, installing debug packages, source packages, extended information from repositories and administration.
    - name: Add Docker repo
      get_url:
        url: https://download.docker.com/linux/centos/docker-ce.repo
        dest: /etc/yum.repos.d/docker-ce.repo
    - name: Install Docker
      package:
        name: docker-ce
        state: latest
    - name: Install pip
      package: 
        name: python3-pip
        state: present
        update_cache: true
    - name: Install docker sdk
      pip:
        name: docker
    - name: Add user ec2-user to docker group
      user:
        name: ec2-user
        groups: docker
        append: yes
    - name: Start Docker service
      service:
        name: docker
        state: started
        enabled: yes
    - name: create build directory
      file:
        path: /home/ec2-user/postgresql
        state: directory
        owner: root
        group: root
        mode: '0755'
    - name: copy the sql script
      copy:
        src: /home/ec2-user/ansible-Project/postgres/init.sql
        dest: /home/ec2-user/postgresql
    - name: copy the Dockerfile
      copy:
        src: /home/ec2-user/ansible-Project/postgres/Dockerfile
        dest: /home/ec2-user/postgresql
    - name: remove serdar_postgre container and serdarcw/postgre if exists
      shell: "docker ps -q --filter 'name=serdar_postgre' && docker stop serdar_postgre && docker rm -fv serdar_postgre && docker image rm -f serdarcw/postgre || echo 'Not Found'"
    - name: build container image
      docker_image:
        name: serdarcw/postgre
        build:
          path: /home/ec2-user/postgresql
        source: build
        state: present
    - name: Launch postgresql docker container
      docker_container:
        name: serdar_postgre
        image: serdarcw/postgre
        state: started
        ports: 
        - "5432:5432"
        env:
          POSTGRES_PASSWORD: "Pp123456789"
        volumes:
          - /custom/mount:/var/lib/postgresql/data
      register: container_info
    - name: Print the container_info
      debug:
        msg: "{{ container_info }}"
```

- Execute it.

```
ansible-playbook docker_postgre.yml
```

- Change directory to `~/ansible-Project/nodejs` directory.

```bash
cd ~/ansible-Project/nodejs
```

- Create a Dockerfile.

```Dockerfile
FROM node:14

# Create app directory
WORKDIR /usr/src/app


COPY package*.json ./

RUN npm install
# If you are building your code for production
# RUN npm ci --only=production


# copy all files into the image
COPY . .

EXPOSE 5000

CMD ["node","app.js"]
```

- Change the `~/ansible-Project/todo-app-pern/server/.env` file as below.

```
SERVER_PORT=5000
DB_USER=postgres
DB_PASSWORD=Pp123456789
DB_NAME=clarustodo
DB_HOST=172.31.12.133 # (private ip of postgresql instance)
DB_PORT=5432
```

- change directory `~/ansible` directory.

```bash
cd ~/ansible
```

- Create a yaml file as nodejs playbook and name it `docker_nodejs.yml`.

```yaml
- name: Install docker
  gather_facts: No
  any_errors_fatal: true
  hosts: _ansible_nodejs
  become: true
  tasks:
    - name: upgrade all packages
      yum: 
        name: '*'
        state: latest
    # we may need to uninstall any existing docker files from the centos repo first.
    - name: Remove docker if installed from CentOS repo
      yum:
        name: "{{ item }}"
        state: removed
      with_items:
        - docker
        - docker-client
        - docker-client-latest
        - docker-common
        - docker-latest
        - docker-latest-logrotate
        - docker-logrotate
        - docker-engine
    - name: Install yum utils
      yum:
        name: "{{ item }}"
        state: latest
      with_items:
        - yum-utils
    - name: Add Docker repo
      get_url:
        url: https://download.docker.com/linux/centos/docker-ce.repo
        dest: /etc/yum.repos.d/docker-ce.repo
    - name: Install Docker
      package:
        name: docker-ce
        state: latest
    - name: Install pip
      package:
        name: python3-pip
        state: present
        update_cache: true
    - name: Install docker sdk
      pip:
        name: docker
    - name: Add user ec2-user to docker group
      user:
        name: ec2-user
        groups: docker
        append: yes
    - name: Start Docker service
      service:
        name: docker
        state: started
        enabled: yes
    - name: create build directory
      file:
        path: /home/ec2-user/nodejs
        state: directory
        owner: root
        group: root
        mode: '0755'
    # at this point do not forget change DB_HOST env variable for postgresql node
    - name: copy files to the nodejs node
      copy:
        src: /home/ec2-user/ansible-Project/todo-app-pern/server/
        dest: /home/ec2-user/nodejs
    - name: copy the Dockerfile
      copy:
        src: /home/ec2-user/ansible-Project/nodejs/Dockerfile
        dest: /home/ec2-user/nodejs
    - name: remove serdar_nodejs container if exists
      shell: "docker ps -q --filter 'name=serdar_nodejs' && docker stop serdar_nodejs && docker rm -fv serdar_nodejs && docker image rm serdarcw/nodejs || echo 'Not Found'"
    - name: build container image
      docker_image:
        name: serdarcw/nodejs
        build:
          path: /home/ec2-user/nodejs
        source: build
        state: present
    - name: Launch postgresql docker container
      docker_container:
        name: serdar_nodejs
        image: serdarcw/nodejs
        state: started
        ports:
        - "5000:5000"
      register: container_info
    - name: Print the container_info
      debug:
        msg: "{{ container_info }}"
```

- Execute it.

```
ansible-playbook docker_nodejs.yml
```

- Change directory to `~/ansible-Project/react` directory.

```bash
cd ~/ansible-Project/react
```

- Create a Dockerfile.

```Dockerfile
FROM node:14

# Create app directory
WORKDIR /app


COPY package*.json ./

RUN yarn install

# copy all files into the image
COPY . .

EXPOSE 3000

CMD ["yarn", "run", "start"]
```

- Change the `~/ansible-Project/todo-app-pern/client/.env` file as below.

```
REACT_APP_BASE_URL=http://<public ip of nodejs>:5000/
```

- change directory `~/ansible` directory.

```bash
cd ~/ansible
```

- Create a yaml file as react playbook and name it `docker_react.yml`.

```yaml
- name: Install docker
  gather_facts: No
  any_errors_fatal: true
  hosts: _ansible_react
  become: true
  tasks:
    - name: upgrade all packages
      yum: 
        name: '*'
        state: latest
    # we may need to uninstall any existing docker files from the centos repo first.
    - name: Remove docker if installed from CentOS repo
      yum:
        name: "{{ item }}"
        state: removed
      with_items:
        - docker
        - docker-client
        - docker-client-latest
        - docker-common
        - docker-latest
        - docker-latest-logrotate
        - docker-logrotate
        - docker-engine
    - name: Install yum utils
      yum:
        name: "{{ item }}"
        state: latest
      with_items:
        - yum-utils
    - name: Add Docker repo
      get_url:
        url: https://download.docker.com/linux/centos/docker-ce.repo
        dest: /etc/yum.repos.d/docker-ce.repo
    - name: Install Docker
      package:
        name: docker-ce
        state: latest
    - name: Install pip
      package:
        name: python3-pip
        state: present
        update_cache: true
    - name: Install docker sdk
      pip:
        name: docker
    - name: Add user ec2-user to docker group
      user:
        name: ec2-user
        groups: docker
        append: yes
    - name: Start Docker service
      service:
        name: docker
        state: started
        enabled: yes
    - name: create build directory
      file:
        path: /home/ec2-user/react
        state: directory
        owner: root
        group: root
        mode: '0755'
    # at this point do not forget change DB_HOST env variable for postgresql node
    - name: copy files to the nodejs node
      copy:
        src: /home/ec2-user/ansible-Project/todo-app-pern/client/
        dest: /home/ec2-user/react
    - name: copy the Dockerfile
      copy:
        src: /home/ec2-user/ansible-Project/react/Dockerfile
        dest: /home/ec2-user/react
    - name: remove serdar_react container and serdarcw/react image if exists
      shell: "docker ps -q --filter 'name=serdar_react' && docker stop serdar_react && docker rm -fv serdar_react && docker image rm -f serdarcw/react || echo 'Not Found'"
    - name: build container image
      docker_image:
        name: serdarcw/react
        build:
          path: /home/ec2-user/react
        source: build
        state: present
    - name: Launch react docker container
      docker_container:
        name: serdar_react
        image: serdarcw/react
        state: started
        ports:
        - "3000:3000"
      register: container_info
    - name: Print the container_info
      debug:
        msg: "{{ container_info }}"
```

- Execute it.

```
ansible-playbook docker_react.yml
```

## Part 6 - Prepare one playbook file for all instances.

- Create a `docker_project.yaml` file under `the ~/ansible` folder.

```yaml
- name: Docker install and configuration
  gather_facts: No
  any_errors_fatal: true
  hosts: _development
  become: true
  tasks:
    - name: upgrade all packages
      yum: 
        name: '*'
        state: latest
    # we may need to uninstall any existing docker files from the centos repo first.
    - name: Remove docker if installed from CentOS repo
      yum:
        name: "{{ item }}"
        state: removed
      with_items:
        - docker
        - docker-client
        - docker-client-latest
        - docker-common
        - docker-latest
        - docker-latest-logrotate
        - docker-logrotate
        - docker-engine
    - name: Install yum utils
      yum: 
        name: "{{ item }}"
        state: latest
      with_items:
        - yum-utils
    - name: Add Docker repo
      get_url:
        url: https://download.docker.com/linux/centos/docker-ce.repo
        dest: /etc/yum.repos.d/docker-ce.repo
    - name: Install Docker
      package:
        name: docker-ce
        state: latest
    - name: Install pip
      package:
        name: python3-pip
        state: present
        update_cache: true
    - name: Install docker sdk
      pip:
        name: docker
    - name: Add user ec2-user to docker group
      user:
        name: ec2-user
        groups: docker
        append: yes
    - name: Start Docker service
      service:
        name: docker
        state: started
        enabled: yes

- name: Postgre Database configuration
  hosts: _ansible_postgresql
  become: true
  gather_facts: No
  any_errors_fatal: true
  vars: 
    postgre_home: /home/ec2-user/ansible-Project/postgres
    postgre_container: /home/ec2-user/postgresql
    container_name: serdar_postgre
    image_name: serdarcw/postgre
  tasks:
    - name: create build directory
      file:
        path: "{{ postgre_container }}"
        state: directory
        owner: root
        group: root
        mode: '0755'
    - name: copy the sql script
      copy:
        src: /home/ec2-user/ansible-Project/postgres/init.sql
        dest: "{{ postgre_container }}"
    - name: copy the Dockerfile
      copy:
        src: /home/ec2-user/ansible-Project/postgres/Dockerfile
        dest: "{{ postgre_container }}" 
    - name: remove {{ container_name }} container and {{ image_name }} if exists
      shell: "docker ps -q --filter 'name={{ container_name }}' && docker stop {{ container_name }} && docker rm -fv {{ container_name }} && docker image rm -f {{ image_name }} || echo 'Not Found'"
    - name: build container image
      docker_image:
        name: "{{ image_name }}"
        build:
          path: "{{ postgre_container }}"
        source: build
        state: present
    - name: Launch postgresql docker container
      docker_container:
        name: "{{ container_name }}"
        image: "{{ image_name }}"
        state: started
        ports: 
        - "5432:5432"
        env:
          POSTGRES_PASSWORD: "Pp123456789"
        volumes:
          - /custom/mount:/var/lib/postgresql/data
      register: docker_info
- name: Nodejs Server configuration
  hosts: _ansible_nodejs
  become: true
  gather_facts: No
  any_errors_fatal: true
  vars: 
    nodejs_home: /home/ec2-user/ansible-Project/nodejs
    container_path: /home/ec2-user/nodejs
    container_name: serdar_nodejs
    image_name: serdarcw/nodejs
  tasks:
    - name: create build directory
      file:
        path: "{{ container_path }}"
        state: directory
        owner: root
        group: root
        mode: '0755'
    # at this point do not forget change DB_HOST env variable for postgresql node
    - name: copy files to the nodejs node
      copy:
        src: /home/ec2-user/ansible-Project/todo-app-pern/server/
        dest: "{{ container_path }}"
    - name: copy the Dockerfile
      copy:
        src: /home/ec2-user/ansible-Project/nodejs/Dockerfile
        dest: "{{ container_path }}"
    - name: remove {{ container_name }} container and {{ image_name }} if exists
      shell: "docker ps -q --filter 'name={{ container_name }}' && docker stop {{ container_name }} && docker rm -fv {{ container_name }} && docker image rm -f {{ image_name }} || echo 'Not Found'"
    - name: build container image
      docker_image:
        name: "{{ image_name }}"
        build:
          path: "{{ container_path }}"
        source: build
        state: present
    - name: Launch postgresql docker container
      docker_container:
        name: "{{ container_name }}"
        image: "{{ image_name }}"
        state: started
        ports:
        - "5000:5000"
- name: React UI Server configuration
  hosts: _ansible_react
  become: true
  gather_facts: No
  any_errors_fatal: true
  vars: 
    react_home: /home/ec2-user/ansible-Project/react
    container_path: /home/ec2-user/react
    container_name: serdar_react
    image_name: serdarcw/react
  tasks:
    - name: create build directory
      file:
        path: "{{ container_path }}"
        state: directory
        owner: root
        group: root
        mode: '0755'
    # at this point do not forget change DB_HOST env variable for postgresql node
    - name: copy files to the react node
      copy:
        src: /home/ec2-user/ansible-Project/todo-app-pern/client/
        dest: "{{ container_path }}"
    - name: copy the Dockerfile
      copy:
        src: /home/ec2-user/ansible-Project/react/Dockerfile
        dest: "{{ container_path }}"
    - name: remove {{ container_name }} container and {{ image_name }} image if exists
      shell: "docker ps -q --filter 'name={{ container_name }}' && docker stop {{ container_name }} && docker rm -fv {{ container_name }} && docker image rm -f {{ image_name }} || echo 'Not Found'"
    - name: build container image
      docker_image:
        name: "{{ image_name }}"
        build:
          path: "{{ container_path }}"
        source: build
        state: present
    - name: Launch react docker container
      docker_container:
        name: "{{ container_name }}"
        image: "{{ image_name }}"
        state: started
        ports:
        - "3000:3000"
```

- Execute it.

```bash
ansible-playbook docker_project.yaml
```


## Part 7 - Prepare playbook with roles solution.

- Cretae a role folder in /home/ec2-user/ansible. Then create role folders. 

```bash
mkdir roles && cd roles
ansible-galaxy init docker
ansible-galaxy init postgre
ansible-galaxy init nodejs
ansible-galaxy init react
```

- Add the `roles_path = /home/ec2-user/ansible/roles` to the ansible.cfg.

- Go to the /home/ec2-user/roles/docker/tasks/main.yml and copy the following.

```yml
    - name: update all pkgs
      yum:
        name: "*"
        state: latest

    - name: Remove docker if installed from CentOS repo
      yum:
        name: "{{ item }}"
        state: removed
      loop:
        - docker
        - docker-client
        - docker-client-latest
        - docker-common
        - docker-latest
        - docker-latest-logrotate
        - docker-logrotate
        - docker-engine

    - name: Install yum utils
      yum: 
        name: yum-utils
        state: latest

    - name: Add Docker repo
      get_url:
        url: https://download.docker.com/linux/centos/docker-ce.repo
        dest: /etc/yum.repos.d/docker-ce.repo

    - name: Install Docker
      package:
        name: docker-ce
        state: latest

    - name: Install pip
      package:
        name: python3-pip
        state: present

    - name: Install docker sdk
      pip:
        name: docker

    - name: Add user ec2-user to docker group
      user:
        name: ec2-user
        groups: docker
        append: yes

    - name: Start Docker service
      service:
        name: docker
        state: started
        enabled: yes
```

- Go to the /home/ec2-user/roles/postgre/tasks/main.yml and copy the followings.

```yml
    - name: create build directory
      file:
        path: "{{ postgre_container }}"
        state: directory
        owner: root
        group: root
        mode: '0755'

    - name: copy the sql script
      copy:
        src: init.sql   # write only file name
        dest: "{{ postgre_container }}"

    - name: copy the Dockerfile
      copy:
        src: Dockerfile   # write only file name
        dest: "{{ postgre_container }}"

    - name: remove {{ container_name }} container and {{ image_name }} if exists
      shell: "docker ps -q --filter 'name={{ container_name }}' && docker stop {{ container_name }} && docker rm -fv {{ container_name }} && docker image rm -f {{ image_name }} || echo 'Not Found'"

    - name: build container image
      docker_image:
        name: "{{ image_name }}"
        build:
          path: "{{ postgre_container }}"
        source: build
        state: present

    - name: Launch postgresql docker container
      docker_container:
        name: "{{ container_name }}"
        image: "{{ image_name }}"
        state: started
        ports: 
        - "5432:5432"
        env:
          POSTGRES_PASSWORD: "Pp123456789"
        volumes:
          - /db-data:/var/lib/postgresql/data
```

- Copy /home/ec2-user/ansible/ansible-project/postgres/init.sql and /home/ec2-user/ansible/ansible-project/postgres/Dockerfile to /home/ec2-user/ansible/roles/postgre/files.

- Copy these variables to /home/ec2-user/ansible/roles/postgre/vars/main.yml.

```yml
postgre_home: /home/ec2-user/ansible/ansible-Project/postgres
postgre_container: /home/ec2-user/postgresql
container_name: oliver_postgre
image_name: olivercw/postgre
```

- Go to the /home/ec2-user/ansible/roles/nodejs/tasks/main.yml and copy.

```yml
    - name: create build directory
      file:
        path: "{{ container_path }}"
        state: directory
        owner: root
        group: root
        mode: '0755'

    - name: copy files to the nodejs node
      copy:
        src: server/    # write only file name
        dest: "{{ container_path }}"

    - name: copy the Dockerfile
      copy:
        src: Dockerfile    # write only file name
        dest: "{{ container_path }}"

    - name: remove {{ container_name }} container and {{ image_name }} if exists
      shell: "docker ps -q --filter 'name={{ container_name }}' && docker stop {{ container_name }} && docker rm -fv {{ container_name }} && docker image rm -f {{ image_name }} || echo 'Not Found'"

    - name: build container image
      docker_image:
        name: "{{ image_name }}"
        build:
          path: "{{ container_path }}"
        source: build
        state: present

    - name: Launch postgresql docker container
      docker_container:
        name: "{{ container_name }}"
        image: "{{ image_name }}"
        state: started
        ports:
        - "5000:5000"
```

- Copy /home/ec2-user/ansible/ansible/project/server folder and /home/ec2-user/ansible/ansible-project/nodejs/Dockerfile to /home/ec2-user/ansible/roles/nodejs/files.

- Copy these variables to /home/ec2-user/ansible/roles/nodejs/vars/main.yml.

```yml
nodejs_home: /home/ec2-user/ansible/ansible-Project/nodejs
container_path: /home/ec2-user/nodejs
container_name: oliver_nodejs
image_name: olivercw/nodejs
```

- Go to the /home/ec2-user/ansible/roles/react/tasks/main.yml and copy.

```yml
    - name: create build directory
      file:
        path: "{{ container_path }}"
        state: directory
        owner: root
        group: root
        mode: '0755'

    - name: copy files to the react node
      copy:
        src: client/   # write only file name
        dest: "{{ container_path }}"

    - name: copy the Dockerfile
      copy:
        src: Dockerfile   # write only file name
        dest: "{{ container_path }}"

    - name: remove {{ container_name }} container and {{ image_name }} image if exists
      shell: "docker ps -q --filter 'name={{ container_name }}' && docker stop {{ container_name }} && docker rm -fv {{ container_name }} && docker image rm -f {{ image_name }} || echo 'Not Found'"

    - name: build container image
      docker_image:
        name: "{{ image_name }}"
        build:
          path: "{{ container_path }}"
        source: build
        state: present

    - name: Launch react docker container
      docker_container:
        name: "{{ container_name }}"
        image: "{{ image_name }}"
        state: started
        ports:
        - "3000:3000"
```

- Copy /home/ec2-user/ansible/ansible-project/client folder and /home/ec2-user/ansible/ansible-project/react/Dockerfile to /home/ec2-user/ansible/roles/react/files.

- Copy these variables to /home/ec2-user/ansible/roles/react/vars/main.yml.

```yml
react_home: /home/ec2-user/ansible/ansible-project/react
container_path: /home/ec2-user/react
container_name: oliver_react
image_name: olivercw/react
```

- Go to the /home/ec2-user/ansible/ and create a playbook.

```bash
cd /home/ec2-user/ansible/
vim play-role.yml
```

```yml
- name: Docker install and configuration
  hosts: _development
  become: true
  roles: 
    - docker
- name: Postgre Database configuration
  hosts: _ansible_postgresql
  become: true
  roles:
    - postgre
- name: Nodejs server configuration
  hosts: _ansible_nodejs
  become: true
  roles:
    - nodejs
- name: React UI Server configuration
  hosts: _ansible_react
  become: true
  roles:
    - react
```

- Run the playbook.

```bash
ansible-playbook play-role.yml
```