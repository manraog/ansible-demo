# Ansible para millennials
# _Sesión 1_
## Administración de configuración
Los administradores de sistemas se encargan de instalar servidores y configurar esos servidores, estas son tareas muy repetitivas. 
Pueden automatizar estas tareas mediante escripts pero es algo complicado en si la insfraestructura es muy grande o cambiante.

La administración de configuración trata de resolver este problema. Es la practica de manejar los cambios de forma sitematica para que se mantenga la integridad del sistema en el tiempo. La administración de configuración se asegura que la descripción y el estado del sistema sea conocida y así no confiar en el conocimiento implicito del equipo de desarrollo.

## Ejemplos de problemas a resolver
- Automatizar:
	- Instalación, actualización y borrado de software
	- Inicio y detención de servicios
	- Aplicar políticas de seguridad
	- Configuración del SO
	- Despliegue de aplicaciones
- Escalabilidad:
	- Instala X versión de Y software en N sistemas
	- Asegurar que N máquinas tengan X configuración

## Caracteristicas de Ansible
- No requiere agente, utiliza SSH
- Simple de usar
- Facil de leer
- Facil de modificar
- Ejecución controlada

## Palabras clave
- **Control node:** Maquina Linux con Ansible instalado en la que se pueden ejecutar playbooks.
- **Managed node o Host:** Servidor a controlar con Ansible
- **Inventory:** Lista de hosts a controlar con Ansible, pueden ser agrupados para controlar grupos de hosts.
- **Module:** Unidad de código a ejecutar por Ansible, cada modulo tienen un propósito especifico (administrar usuarios, instalar paquetes, etc...)
- **Task:** Unidad de acción en Ansible (llamada a un modulo)
- **Playbook:** Lista de tareas a ejecutar en orden, soporta variables y están escritos en YAML.

## Requerimientos del *control node*:
- Distribución Linux
- Python 2 (2.7) o Python 3 (3.5 o mayor)
- sshpass (Opcional - Login con password)

## Requerimientos del *Host*:
- sshd
- sft o scp
- Python 2 (2.7) o Python 3 (3.5 o mayor)

## Hola mundo en Ansible

### Requisitos
		cd demo1
		docker-compose up -d

### Inventario
		# Hosts
		[test]
		docker1 ansible_port=2222
		docker2 ansible_port=2223

		# Variables de grupo
		[test:vars]
		ansible_user=ubuntu
		ansible_host=localhost
		ansible_ssh_extra_args='-o StrictHostKeyChecking=no'

### Comprobar estado de Docker2
	ansible -i inventory.ini --ask-pass -m ping docker2

### Comprobar estado de Docker2 mostrando logs
	ansible -vvv -i inventory.ini --ask-pass -m ping docker2

### Comprobar estado de todos los servidores
	ansible -i inventory.ini --ask-pass -m ping all

### Generar llaves SSH
	ssh-keygen
	ls -l /home/$USER/.ssh

### Copiar llave publica
	ssh-copy-id ubuntu@localhost -p 2222
---
	ansible -i inventory.ini --ask-pass -m authorized_key -a "user=ubuntu state=present key={{ lookup('file', '/home/ansible/.ssh/id_rsa.pub') }}" test
---
	ssh ubuntu@localhost -p 2222
	ansible -i inventory.ini -m ping all

### Comprobar el usuario
	ansible -i inventory.ini -m command -a "whoami" all

### Coprobar sudoers
	ansible -i inventory.ini -b --ask-become-pass -m command -a "whoami" all

### Detener y eliminar contenedores
	docker-compose down


## Referencias:
- https://docs.ansible.com/ansible/latest/index.html
- https://www.jeffgeerling.com/blog/brief-history-ssh-and-remote-access
- https://sysadmincasts.com/episodes/43-19-minutes-with-ansible-part-1-4
- https://www.hostinger.com/tutorials/ssh-tutorial-how-does-ssh-work
- https://www.ssl2buy.com/wiki/symmetric-vs-asymmetric-encryption-what-are-differences


# _Sesión 2_

## YAML

### Ejemplos

```yaml
- Cat
- Dog
- Goldfish
```

```yaml
martin:
    name: Martin D'vloper
    job: Developer
    skill: Elite
```

```yaml
---
# Employee records
-  martin:
    name: Martin D'vloper
    job: Developer
    skills:
      - python
      - perl
      - pascal
-  tabitha:
    name: Tabitha Bitumen
    job: Developer
    skills:
      - lisp
      - fortran
      - erlang
```

### YAML a JSON
https://www.json2yaml.com/

## Playboks
- Aplicar configuraciones
- Desplegar aplicaciones
- Utilizamos módulos para definir tareas
- Similar a una receta de cocina o lista de tareas
- Fácil de leer
- Fácil de modificar

## Inventario

```ini
# Hosts subgroups
[ubuntu_desktops]
ubuntu1 ansible_host=localhost ansible_user=ubuntu ansible_port=2224

[centos_servers]
centos1 ansible_host=localhost ansible_user=root ansible_port=2225
centos_nginx1 ansible_host=localhost ansible_user=root ansible_port=2226

# Host groups
[demo2:children]
ubuntu_desktops
centos_servers

# Variables de grupo
[demo2:vars]
ansible_ssh_extra_args='-o StrictHostKeyChecking=no'
```

## Copiar llave pública
	ansible -i inventory.ini --ask-pass -m authorized_key -a "user=ubuntu state=present key={{ lookup('file', '/home/ansible/.ssh/id_rsa.pub') }}" ubuntu_desktops

	ansible -i inventory.ini --ask-pass -m authorized_key -a "user=root state=present key={{ lookup('file', '/home/ansible/.ssh/id_rsa.pub') }}" centos_servers

## Playbook

```bash
vim playbook1.yml
```

```yaml
---
- hosts: centos1
  tasks:
    - name: 'Instala Apache'
      yum: 
        name: 'httpd' 
        state: 'latest'

- hosts: ubuntu1
  become: true
  vars:
    packages: [ 'vim', 'git', 'curl' ]
  tasks:
    - name: 'Ejecuta el equivalente de "apt-get update"'
      apt:
        update_cache: yes
    - name: 'Instala los paquetes'
      apt: 
        name: '{{ packages }}' 
        state: 'latest'
```

## Check mode o dry-run
	ansible-playbook --syntax-check test-playook1.yml
	
	ansible-playbook -i inventory.ini --ask-become-pass --check playook1.yml

## Ejecutar Playbook
	ansible-playbook -v -i inventory.ini --ask-become-pass playook1.yml

```bash
ansible -i inventory.ini -m setup ubuntu-host
```

## Verificar instalación de paqutes
```bash
sudo docker exec -it centos yum list installed httpd
sudo docker exec -it ubuntu curl --version
sudo docker exec -it ubuntu git --version
sudo docker exec -it ubuntu vim --version
```

## Idempotencia

Si ejecutamos nuevamente el playbook obtendremos un resultado de ejecución distinto:
```bash
ansible-playbook -v -i inventory.ini --ask-become-pass playook1.yml
```

## Handlers

### Playbook

```bash
vim playbook2.yml
```

```yaml
---
- hosts: nginx-servers 
  become: true
  
  tasks:
    - name: 'Add epel-release repo'
      yum:
        name: 'epel-release'
        state: present
    - name: 'Install nginx'
      yum:
        name: 'nginx'
        state: present
    - name: 'Insert Index Page'
      template:
        src: 'index.html.j2'
        dest: '/usr/share/nginx/html/index.html'
    - name: 'Setup nginx conf'
      template:
        src: 'nginx.conf.j2'
        dest: '/etc/nginx/nginx.conf'
      notify: 'restart nginx'
    - name: 'Start NGiNX'
      service:
        name: 'nginx'
        state: started

  handlers:
    - name: 'restart nginx'
      service:
        name: 'nginx'
        state: restarted
```

### Template configuración NGiNX

```bash
vim nginx.conf.j2
```


```conf
# For more information on configuration, see:
#   * Official English Documentation: http://nginx.org/en/docs/
#   * Official Russian Documentation: http://nginx.org/ru/docs/
#
#
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

# Load dynamic modules. See /usr/share/nginx/README.dynamic.
include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 1024;
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 2048;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    # Load modular configuration files from the /etc/nginx/conf.d directory.
    # See http://nginx.org/en/docs/ngx_core_module.html#include
    # for more information.
    include /etc/nginx/conf.d/*.conf;

    server {
        listen       80 default_server;
        listen       [::]:80 default_server;
        server_name  _;
        root         /usr/share/nginx/html;

        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;

        location / {
        }

        error_page 404 /404.html;
            location = /40x.html {
        }

        error_page 500 502 503 504 /50x.html;
            location = /50x.html {
        }
    }

# Settings for a TLS enabled server.
#
#    server {
#        listen       443 ssl http2 default_server;
#        listen       [::]:443 ssl http2 default_server;
#        server_name  _;
#        root         /usr/share/nginx/html;
#
#        ssl_certificate "/etc/pki/nginx/server.crt";
#        ssl_certificate_key "/etc/pki/nginx/private/server.key";
#        ssl_session_cache shared:SSL:1m;
#        ssl_session_timeout  10m;
#        ssl_ciphers HIGH:!aNULL:!MD5;
#        ssl_prefer_server_ciphers on;
#
#        # Load configuration files for the default server block.
#        include /etc/nginx/default.d/*.conf;
#
#        location / {
#        }
#
#        error_page 404 /404.html;
#            location = /40x.html {
#        }
#
#        error_page 500 502 503 504 /50x.html;
#            location = /50x.html {
#        }
#    }

}
```

### Template Index

```bash
vim index.html.j2
```

```html
<!doctype html>
<html>
  <head>
    <title>NGINX S&PS</title>
  </head>
  <body>
        <h1 style="text-align: center;"><strong><span style="color: #ff0000;">S&amp;P</span> Solutions</strong></h1>  
        {# Usamos uno de los 'facts' recopilados por Ansible para mostrar la fecha del host #}
        <h1 style="text-align: center;"><strong>{{ ansible_date_time.date }}</strong></h1>  
        <!--<img src="https://img.devrant.com/devrant/rant/r_565529_WRQcK.jpg" alt="Doge HTML" align="middle"> -->        
  </body>
</html>
```

### Ejecutar Playbook
	ansible-playbook -vvv -i inventory.ini --ask-become-pass playook2.yml

	ansible-playbook -vvv -i inventory.ini --ask-become-pass playook2.yml

### Cambio de Configuración

```bash
vim ansiblePlaybooks/nginx.conf.j2
```

Y añadimos unos comentarios al inicio de nuestro template para simular cambios en nuestra configuracion:
```conf
# Configuración super loca
# Más configuración
# Fin de los cambios
```
