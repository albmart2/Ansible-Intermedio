# Curso Intermedio de Ansible

Si el nivel básico consiste en instalar paquetes y hacer pings, el nivel intermedio empieza cuando te das cuenta de que copiar el mismo bloque YAML 27 veces no es automatización, es sufrimiento asistido por ordenador.

## Índice
1. [Inventarios avanzados](#1-inventarios-avanzados)
2. [Variables y precedencia](#2-variables-y-precedencia)
3. [Archivos de variables](#3-archivos-de-variables)
4. [Templates con Jinja2](#4-templates-con-jinja2)
5. [Roles](#5-roles)
6. [Tags](#6-tags)
7. [Register](#7-register)
8. [Blocks, Rescue y Always](#8-blocks-rescue-y-always)
9. [Delegation](#9-delegation)
10. [Async y Poll](#10-async-y-poll)
11. [Ansible Vault](#11-ansible-vault)
12. [Includes e Imports](#12-includes-e-imports)
13. [Gestión de errores](#13-gestión-de-errores)
14. [Facts personalizados](#14-facts-personalizados)
15. [Buenas prácticas](#15-buenas-prácticas)
16. [Ejemplo completo](#16-ejemplo-completo)
17. [Resumen para examen](#17-resumen-para-examen)

## 1. Inventarios avanzados

### Inventarios agrupados

```ini
[web]
web01
web02

[db]
db01

[produccion:children]
web
db
```

Ahora puedes ejecutar:

```bash
ansible produccion -m ping
```

### Variables por host

```ini
web01 ansible_host=192.168.1.10
web02 ansible_host=192.168.1.11
```

### Variables por grupo

```ini
[web]
web01
web02

[web:vars]
http_port=80
```

## 2. Variables y precedencia

Ansible puede obtener variables desde:

- Playbook
- Inventario
- Group vars
- Host vars
- Línea de comandos

Ejemplo:

```bash
ansible-playbook deploy.yml -e "version=2.0"
```

Dentro del playbook:

```yaml
version: 1.0
```

Ganará:

```yaml
2.0
```

Porque la línea de comandos tiene más prioridad.

## 3. Archivos de variables

### vars_files

```yaml
---
- hosts: web

  vars_files:
    - variables.yml
```

variables.yml

```yaml
usuario: alba
puerto: 8080
```

Uso:

```yaml
{{ usuario }}
```

## 4. Templates con Jinja2

Uno de los conceptos más importantes.

### Archivo plantilla

```jinja2
Servidor: {{ inventory_hostname }}

Puerto: {{ puerto }}
```

Guardar:

```bash
config.j2
```

### Playbook

```yaml
- name: Generar configuración
  template:
    src: config.j2
    dest: /tmp/config.txt
```

Resultado:

```yaml
Servidor: web01
Puerto: 8080
```

Magia negra corporativa aceptada por la industria.

## 5. Roles

Cuando el proyecto crece, aparecen los roles.

Crear:

```bash
ansible-galaxy init nginx
```

Estructura:

```bash
nginx/

├── defaults
├── files
├── handlers
├── meta
├── tasks
├── templates
├── tests
└── vars
```

### tasks/main.yml

```yaml
---
- name: Instalar nginx
  apt:
    name: nginx
    state: present
```

### Uso del rol

```yaml
---
- hosts: web

  roles:
    - nginx
```

## 6. Tags

Permiten ejecutar tareas concretas.

```yaml
tasks:

- name: Instalar nginx
  apt:
    name: nginx
    state: present
  tags:
    - install

- name: Reiniciar nginx
  service:
    name: nginx
    state: restarted
  tags:
    - restart
```

Ejecutar:

```bash
ansible-playbook web.yml --tags install
```

## 7. Register

Guarda resultados de una tarea.

```yaml
- name: Obtener hostname
  command: hostname
  register: resultado
```

Ver contenido:

```yaml
- debug:
    var: resultado.stdout
```

Resultado:

```yaml
web01
```
