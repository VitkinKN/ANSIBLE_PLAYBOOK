# HOMEWORK_ANSIBLE_PLAYBOOK_8_2_(VITKIN_K_N)



### 1. Подготовка к выполнению.
#### Создаём свой собственный публичный репозиторий на github

![Playbook repo for task ansible - playbook](https://github.com/VitkinKN/ANSIBLE_PLAYBOOK/tree/master/playbook)

#### Подготавливаем хосты в соотвтествии с группами из предподготовленного playbook.
```
konstantin@konstantin-forever:~$ docker ps
CONTAINER ID   IMAGE           COMMAND   CREATED        STATUS       PORTS     NAMES
702b99a84a80   ubuntu:latest   "bash"    19 hours ago   Up 2 hours             elastichost
443efc9e32b6   ubuntu:latest   "bash"    19 hours ago   Up 2 hours             kibanahost
```
#### Скачаем дистрибутив java и положим его в директорию playbook/files/
- *Также скачаем elasticsearch-7.10.1-linux-x86_64.tar.gz и kibana-7.10.1-linux-x86_64.tar.gz и поместим их в директорию playbook/files/*
___
### 2. Выполнение.
#### *Приготовим свой собственный inventory файл prod.yml*
#### *Допишим playbook: сделаем ещё один play, который устанавливает и настраивает kibana.*
```yaml
---
- name: Install Java
  hosts: all
  tasks:
    - name: Set facts for Java 11 vars
      set_fact:
        java_home: "/opt/jdk/{{ java_jdk_version }}"
      tags: java
    - name: Upload .tar.gz file containing binaries from local storage
      copy:
        src: "{{ java_oracle_jdk_package }}"
        dest: "/tmp/jdk-{{ java_jdk_version }}.tar.gz"
        mode: 0755
      register: download_java_binaries
      until: download_java_binaries is succeeded
      tags: java
    - name: Ensure installation dir exists
      become: true
      file:
        state: directory
        path: "{{ java_home }}"
      tags: java
    - name: Extract java in the installation directory
      become: true
      unarchive:
        copy: false
        src: "/tmp/jdk-{{ java_jdk_version }}.tar.gz"
        dest: "{{ java_home }}"
        extra_opts: [--strip-components=1]
        creates: "{{ java_home }}/bin/java"
      tags:
        - java
    - name: Export environment variables
      become: true
      template:
        src: jdk.sh.j2
        dest: /etc/profile.d/jdk.sh
      tags: java
- name: Install Elasticsearch
  hosts: elastichost
  tasks:
    - name: Upload tar.gz Elasticsearch from local storage
      copy:
        src: "{{ elastic_doun }}"
        dest: "/tmp/elasticsearch-{{ elastic_version }}.tar.gz"
        mode: 0755
      register: Upload_elastic
      until: Upload_elastic is succeeded
      tags: elastic
    - name: Create directrory for Elasticsearch
      file:
        state: directory
        path: "{{ elastic_home }}"
      tags: elastic
    - name: Extract Elasticsearch in the installation directory
      become: true
      unarchive:
        copy: false
        src: "/tmp/elasticsearch-{{ elastic_version }}.tar.gz/elasticsearch-7.10.1-linux-x86_64.tar.gz"
        dest: "{{ elastic_home }}"
        extra_opts: [--strip-components=1]
        creates: "{{ elastic_home }}/bin/elasticsearch"
      tags:
        - elastic
    - name: Set environment Elastic
      become: true
      template:
        src: templates/elk.sh.j2
        dest: /etc/profile.d/elk.sh
      tags: elastic
- name: Install Kibana
  hosts: kibanahost
  tasks:
    - name: Upload tar.gz kibana from local storage
      copy:
        src: "{{ kibana_doun }}"
        dest: "/tmp/kibana-{{ kibana_version }}-linux-x86_64.tar.gz"
        mode: 0755
      register: Upload_kibana
      until: Upload_kibana is succeeded
      tags: kibana
    - name: Create directrory for Kibana ({{ kibana_home }})
      file:
        path: "{{ kibana_home }}"
        state: directory
      tags: kibana
    - name: Extract Kibana in the installation directory
      become: true
      unarchive:
        copy: false
        src: "/tmp/kibana-{{ kibana_version }}-linux-x86_64.tar.gz/kibana-7.10.1-linux-x86_64.tar.gz"
        dest: "{{ kibana_home }}"
        extra_opts: [--strip-components=1]
        creates: "{{ kibana_home }}/bin/kibana"
      tags:
        - kibana
    - name: Set environment Kibana
      become: true
      template:
        src: templates/kib.sh.j2
        dest: /etc/profile.d/kib.sh
      tags: kibana
```
___
#### *Запустим ansible-lint site.yml и исправим ошибки, если они есть.*
```
konstantin@konstantin-forever:~/DEVOPS_COURSE/WORK_ANSIBLE_PLAYBOOK/playbook$ ansible-lint site.yml
WARNING  Overriding detected file kind 'yaml' with 'playbook' for given positional argument: site.yml
WARNING  Listing 8 violation(s) that are fatal
risky-file-permissions: File permissions unset or incorrect
site.yml:17 Task/Handler: Ensure installation dir exists
risky-file-permissions: File permissions unset or incorrect
site.yml:33 Task/Handler: Export environment variables
var-naming: Task registers a variable that violates variable naming standards
site.yml:42 Task/Handler: Upload tar.gz Elasticsearch from local storage
risky-file-permissions: File permissions unset or incorrect
site.yml:50 Task/Handler: Create directrory for Elasticsearch
risky-file-permissions: File permissions unset or incorrect
site.yml:65 Task/Handler: Set environment Elastic
var-naming: Task registers a variable that violates variable naming standards
site.yml:74 Task/Handler: Upload tar.gz kibana from local storage
risky-file-permissions: File permissions unset or incorrect
site.yml:82 Task/Handler: Create directrory for Kibana ({{ kibana_home }})
risky-file-permissions: File permissions unset or incorrect
site.yml:97 Task/Handler: Set environment Kibana
You can skip specific rules or tags by adding them to your configuration file:
# .ansible-lint
warn_list:  # or 'skip_list' to silence them completely
  - experimental  # all rules tagged as experimental
Finished with 0 failure(s), 8 warning(s) on 1 files.
```
- *Есть предупреждения, ошибок нет*
#### *Запустим playbook на этом окружении с флагом --check*
```
konstantin@konstantin-forever:~/DEVOPS_COURSE/WORK_ANSIBLE_PLAYBOOK/playbook$ ansible-playbook -i inventory/prod.yml site.yml --check
PLAY [Install Java] ************************************************************
TASK [Gathering Facts] *********************************************************
ok: [kibanahost]
ok: [elastichost]
TASK [Set facts for Java 11 vars] **********************************************
ok: [elastichost]
ok: [kibanahost]
TASK [Upload .tar.gz file containing binaries from local storage] **************
ok: [elastichost]
ok: [kibanahost]
TASK [Ensure installation dir exists] ******************************************
ok: [elastichost]
ok: [kibanahost]
TASK [Extract java in the installation directory] ******************************
skipping: [elastichost]
skipping: [kibanahost]
TASK [Export environment variables] ********************************************
ok: [elastichost]
ok: [kibanahost]
PLAY [Install Elasticsearch] ***************************************************
TASK [Gathering Facts] *********************************************************
ok: [elastichost]
TASK [Upload tar.gz Elasticsearch from local storage] **************************
ok: [elastichost]
TASK [Create directrory for Elasticsearch] *************************************
ok: [elastichost]
TASK [Extract Elasticsearch in the installation directory] *********************
skipping: [elastichost]
TASK [Set environment Elastic] *************************************************
ok: [elastichost]
PLAY [Install Kibana] **********************************************************
TASK [Gathering Facts] *********************************************************
ok: [kibanahost]
TASK [Upload tar.gz kibana from local storage] *********************************
ok: [kibanahost]
TASK [Create directrory for Kibana (/opt/kibana/7.10.1)] ***********************
ok: [kibanahost]
TASK [Extract Kibana in the installation directory] ****************************
skipping: [kibanahost]
TASK [Set environment Kibana] **************************************************
ok: [kibanahost]
PLAY RECAP *********************************************************************
elastichost                : ok=9    changed=0    unreachable=0    failed=0    skipped=2    rescued=0    ignored=0   
kibanahost                 : ok=9    changed=0    unreachable=0    failed=0    skipped=2    rescued=0    ignored=0 
```
#### *Запустим playbook на prod.yml окружении с флагом --diff. Убедимся, что изменения на системе произведены*
```
konstantin@konstantin-forever:~/DEVOPS_COURSE/WORK_ANSIBLE_PLAYBOOK/playbook$ ansible-playbook -i inventory/prod.yml site.yml --diff
PLAY [Install Java] ************************************************************
TASK [Gathering Facts] *********************************************************
ok: [kibanahost]
ok: [elastichost]
TASK [Set facts for Java 11 vars] **********************************************
ok: [elastichost]
ok: [kibanahost]
TASK [Upload .tar.gz file containing binaries from local storage] **************
ok: [elastichost]
ok: [kibanahost]
TASK [Ensure installation dir exists] ******************************************
ok: [elastichost]
ok: [kibanahost]
TASK [Extract java in the installation directory] ******************************
skipping: [elastichost]
skipping: [kibanahost]
TASK [Export environment variables] ********************************************
ok: [elastichost]
ok: [kibanahost]
PLAY [Install Elasticsearch] ***************************************************
TASK [Gathering Facts] *********************************************************
ok: [elastichost]
TASK [Upload tar.gz Elasticsearch from local storage] **************************
ok: [elastichost]
TASK [Create directrory for Elasticsearch] *************************************
ok: [elastichost]
TASK [Extract Elasticsearch in the installation directory] *********************
skipping: [elastichost]
TASK [Set environment Elastic] *************************************************
ok: [elastichost]
PLAY [Install Kibana] **********************************************************
TASK [Gathering Facts] *********************************************************
ok: [kibanahost]
TASK [Upload tar.gz kibana from local storage] *********************************
ok: [kibanahost]
TASK [Create directrory for Kibana (/opt/kibana/7.10.1)] ***********************
ok: [kibanahost]
TASK [Extract Kibana in the installation directory] ****************************
skipping: [kibanahost]
TASK [Set environment Kibana] **************************************************
ok: [kibanahost]
PLAY RECAP *********************************************************************
elastichost                : ok=9    changed=0    unreachable=0    failed=0    skipped=2    rescued=0    ignored=0   
kibanahost                 : ok=9    changed=0    unreachable=0    failed=0    skipped=2    rescued=0    ignored=0   
```
#### *Повторно запустите playbook с флагом --diff и убедимся, что playbook идемпотентен*
```
...
TASK [Extract Kibana in the installation directory] ************************************************************************************
skipping: [kibanahost]
TASK [Set environment Kibana] **********************************************************************************************************
ok: [kibanahost]
PLAY RECAP *****************************************************************************************************************************
elastichost                : ok=9    changed=0    unreachable=0    failed=0    skipped=2    rescued=0    ignored=0   
kibanahost                 : ok=9    changed=0    unreachable=0    failed=0    skipped=2    rescued=0    ignored=0   
```
___
