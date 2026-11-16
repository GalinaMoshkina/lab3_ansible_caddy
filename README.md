# Отчет по лабораторной работе 3
Выполнили:  
* Журавлева Анна  
* Калинин Марк  
* Мошкина Галина  
# Ansible + Caddy  
### Задача  
На целевом сервере установить Ansible и вебсервер Caddy  
## 1. Установка и настройка Ansible  
Настройка Ansible  
Необходимо создать файл конфигурации ansible.cfg, а также inventory/hosts, чтобы Ansible понимал, какими серверами ему управлять.  
```
(base) gala@gala-NMH-WDX9:~/admcn$ ls
ansible.cfg  inventory
(base) gala@gala-NMH-WDX9:~/admcn$ cat ansible.cfg
[defaults]
host_key_checking = false
inventory = inventory/hosts
(base) gala@gala-NMH-WDX9:~/admcn$ cat inventory/hosts
[my_servers]
local_server ansible_host=localhost
(base) gala@gala-NMH-WDX9:~/admcn$ 
```  
Результат проверяем через команду `ansible my_servers -m ping -c local` - подключение Ansible локально, как мы и настроили, а не по SSH.  
<img width="1056" height="978" alt="image" src="https://github.com/user-attachments/assets/b4cdc6ee-0cde-4de7-8c73-3051430b2bf2" />  
Далее пробуем создать текстовый файл на локальном хосте  
<img width="1411" height="474" alt="image" src="https://github.com/user-attachments/assets/07ac3ed2-df94-4539-8ee1-2b4404f767a7" />  

## 2. Установка Caddy  
Тут мы создаем Ansible playbook - YAML-файл, где описан список задач, который Ansible выполняет по порядку.  
Вводим `mkdir roles && cd roles` для создания папки `roles`, в ней инициализируем исходное конфигурационное “дерево”: `ansible-galaxy init caddy_deploy`.  
```
(base) gala@gala-NMH-WDX9:~/admcn/roles$ tree
.
└── caddy_deploy
    ├── defaults
    │   └── main.yml
    ├── files
    ├── handlers
    │   └── main.yml
    ├── meta
    │   └── main.yml
    ├── README.md
    ├── tasks
    │   └── main.yml
    ├── templates
    ├── tests
    │   ├── inventory
    │   └── test.yml
    └── vars
        └── main.yml

10 directories, 8 files
```
Согласно методичке наполняем файл roles/caddy_deploy/tasks/main.yml (здесь описываются шаги, которые будут выполняться в плейбуке):  
```
(base) gala@gala-NMH-WDX9:~/admcn$ cat roles/caddy_deploy/tasks/main.yml
---
# tasks file for caddy_deploy 
- name: Install prerequisites
   apt:
     pkg:
     - debian-keyring
     - debian-archive-keyring
     - apt-transport-https
     - curl
- name: Add key for Caddy repo
   apt_key:
     url: https://dl.cloudsmith.io/public/caddy/stable/gpg.key
     state: present
     keyring: /usr/share/keyrings/caddy-stable-archive-keyring.gpg
- name: add Caddy repo
   apt_repository:
     repo: "deb [signed-by=/usr/share/keyrings/caddy-stable-archive-keyring.gpg] https://dl.cloudsmith.io/public/caddy/stable/deb/debian any-version main"
     state: present
     filename: caddy-stable
- name: add Caddy src repo
   apt_repository:
     repo: "deb-src [signed-by=/usr/share/keyrings/caddy-stable-archive-keyring.gpg] https://dl.cloudsmith.io/public/caddy/stable/deb/debian any-version main"
     state: present
     filename: caddy-stable
- name: Install Caddy webserver
   apt:
     name: caddy
     update_cache: yes
     state: present
```
В рабочей директории создаем собственно файл конфигурации самого плейбука `caddy_deploy.yml`, где указываем нужные нам хосты и роли:  
```
(base) gala@gala-NMH-WDX9:~/admcn$ cat caddy_deploy.yml
---
- name: Install and configure Caddy webserver # Любое описание
  hosts: my_servers # хосты из файла inventory/hosts, где будем выполнять наш плейбук
  connection: local # аналог -c local, но для плейбуков
  roles:
    - caddy_deploy # собственно, роль для выполнения
```
Запускаем плейбук с помощью команды `ansible-playbook caddy_deploy.yml`, на этом этапе воникла ошибка с командой apt, вызванная отсутствием GPG-ключа для репозитория PostgreSQL. Для ее устранения ключ был добавлен вручную.  
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/35211897-3c0c-47f8-9354-b455a3369279" />  
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/3b214632-5f2b-44bc-bf58-893e58fad4f3" />  
Новая ошибка  
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/46153e4a-6ebc-4615-b718-a07fe00bd344" />  
apt не может проверить подпись репозитория Caddy, потому что скачанный GPG-ключ не был правильно распознан системой, либо вовсе был просрочен.    
Было принято решение поменять в файле `main.yml` задачи "add Caddy repo" и "add Caddy src repo", а именно добавить `[trusted=yes]` в `repo`:  
```
- name: Add Caddy repository (allowing expired signature for this lab)
  apt_repository:
    repo: "deb [trusted=yes] https://dl.cloudsmith.io/public/caddy/stable/deb/debian any-version main"
    state: present
    filename: caddy-stable
    update_cache: yes
```  
Тогда, убирается сам блок "Add key for Caddy repo":  
```
- name: Add key for Caddy repo
   apt_key:
     url: https://dl.cloudsmith.io/public/caddy/stable/gpg.key
     state: present
     keyring: /usr/share/keyrings/caddy-stable-archive-keyring.gpg
```
## 3. Домен и настройка Caddyfile  
Как оказалось, исправлений из прошлой части не хватило для того, чтобы поправить ситуацию с запуском плейбука.  
Добавляем `roles/caddy_deploy/vars/main.yml` и `roles/caddy_deploy/templates/Caddyfile.j2` и тоже их меняем, а именно вместо localhost прописываем http://localhost (была ошибка с domain_name: "localhost").  
```
(base) gala@gala-NMH-WDX9:~/admcn$ cat roles/caddy_deploy/vars/main.yml
---
# vars file for caddy_deploy
domain_name: "http://localhost"
log: # Можно поиграться со значениями
  file: /var/log/caddy/caddy_access.log
  level: "INFO"
(base) gala@gala-NMH-WDX9:~/admcn$ cat roles/caddy_deploy/templates/Caddyfile.j2
{{ domain_name }} {
    root * /usr/share/caddy
    file_server

    log {
        output file {{ log.file }}
        format json
        level {{ log.level }}
    }
}
```
Также в методичке указан путь /var/log/caddy_access.log. Пользователь caddy, от имени которого работает сервис, не имеет прав на запись в /var/log/. Возникает ошибка permission denied. Был изменен на /var/log/caddy/caddy_access.log и добавлена задача создания этой директории с нужными правами.  
Теперь `roles/caddy_deploy/tasks/main.yml`. В нем добавляем шаги, отвечающие за создание конфигурационного файла из шаблона и последующую перезагрузку сервиса. Только, в месте src: templates/Caddyfile.j2 убираем templates, оставляем src: Caddyfile.j2 (тоже возникала вместе с этим ошибка).  
```
(base) gala@gala-NMH-WDX9:~/admcn$ cat roles/caddy_deploy/tasks/main.yml
---
# tasks file for caddy_deploy 
- name: Install prerequisites
  apt:
    pkg:
    - debian-keyring
    - debian-archive-keyring
    - apt-transport-https
    - curl
- name: add Caddy repo
  apt_repository:
    repo: "deb [trusted=yes] https://dl.cloudsmith.io/public/caddy/stable/deb/debian any-version main"
    state: present
    filename: caddy-stable
- name: add Caddy src repo
  apt_repository:
    repo: "deb-src [trusted=yes] https://dl.cloudsmith.io/public/caddy/stable/deb/debian any-version main"
    state: present
    filename: caddy-stable
- name: Install Caddy webserver
  apt:
    name: caddy
    update_cache: yes
    state: present
- name: Create config file
  template:
    src: Caddyfile.j2 # Откуда берем
    dest: /etc/caddy/Caddyfile # Куда кладем

- name: Reload with new config
  service:
    name: caddy
    state: reloaded
- name: Ensure caddy log directory exists
  file:
    path: /var/log/caddy
    state: directory
    owner: caddy
    group: caddy
    mode: '0755'
```
<img width="1852" height="825" alt="image" src="https://github.com/user-attachments/assets/71f097aa-470c-4e25-a87d-7ffab2b78e81" />  
<img width="1841" height="984" alt="image" src="https://github.com/user-attachments/assets/2c0b4da1-7fc9-4550-8142-2ce2a5725055" />  
<img width="1497" height="356" alt="image" src="https://github.com/user-attachments/assets/e468fc10-2e5e-479f-ba88-5a44ebc89848" />
Ура!!! Плейбук завершился с failed=0, а служба Caddy перешла в статус active (running)!  


## Задание  
1. Переписать пример с созданием и удалением файла из шага 5 Части 1 с ad-hoc команд на плейбук формат,  
   а так же добавить четвертый шаг - перед удалением поменять содержимое файла на любое другое.
2. “Расширить” конфиг вебсервера Caddy любым функционалом по желанию: например, добавить проксирование, или какие-нибудь заголовки ( header ).     Вместо дефолтной страницы Caddy подставить свою, хотя бы index.html с Hello world внутри. Добавить это в качестве дополнительного шага в        tasks.

## Решение задания  
f
