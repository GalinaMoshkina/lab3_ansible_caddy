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
f
## 3. Домен и настройка Caddyfile  
f
## Задание  
1. Переписать пример с созданием и удалением файла из шага 5 Части 1 с ad-hoc команд на плейбук формат,  
   а так же добавить четвертый шаг - перед удалением поменять содержимое файла на любое другое.
2. “Расширить” конфиг вебсервера Caddy любым функционалом по желанию: например, добавить проксирование, или какие-нибудь заголовки ( header ).     Вместо дефолтной страницы Caddy подставить свою, хотя бы index.html с Hello world внутри. Добавить это в качестве дополнительного шага в        tasks.

## Решение задания  
f
