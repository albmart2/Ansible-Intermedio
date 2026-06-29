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

## 8. Blocks, Rescue y Always

Equivalente a try-catch-finally.

```yaml
tasks:

- block:

    - name: Instalar paquete
      apt:
        name: nginx
        state: present

  rescue:

    - debug:
        msg: "Algo ha fallado"

  always:

    - debug:
        msg: "Esto siempre se ejecuta"
```

## 9. Delegation

Ejecutar una tarea en otra máquina.

```yaml
- name: Crear backup
  command: tar czf backup.tar.gz /datos
  delegate_to: backup-server
```

Aunque el playbook se lance sobre web01.

## 10. Async y Poll

Para tareas largas.

```yaml
- name: Actualización larga
  shell: /scripts/update.sh
  async: 300
  poll: 0
```

No espera resultado.

Consultar:

```yaml
- name: Revisar estado
  async_status:
    jid: "{{ job_id }}"
```

## 11. Ansible Vault

Guardar contraseñas cifradas.

### Crear:

```bash
ansible-vault create secretos.yml
```

### Editar:

```bash
ansible-vault edit secretos.yml
```

### Ver:

```bash
ansible-vault view secretos.yml
```

### Ejecutar:

```bash
ansible-playbook playbook.yml --ask-vault-pass
```

## 12. Includes e Imports

### Import

Carga al inicio.

```yaml
- import_tasks: instalar.yml
```

### Include

Carga dinámicamente.

```yaml
- include_tasks: instalar.yml
```

Regla rápida:

- import = estático
- include = dinámico

## 13. Gestión de errores

### Ignorar errores:

```yaml
ignore_errors: yes
```

### Fallar manualmente:

```yaml
- fail:
    msg: "Versión incorrecta"
```

### Controlar cambios:

```yaml
changed_when: false
```

### Controlar fallos:

```yaml
failed_when:
  resultado.rc != 0
```

## 14. Facts personalizados

### Crear:

```bash
/etc/ansible/facts.d/app.fact
```

### Contenido:

```json
{
    "version": "2.0"
}
```

### Acceso:

```yaml
{{ ansible_local.app.version }}
```

## 15. Buenas prácticas

✅ Usar roles.

✅ Usar variables.

✅ Usar Vault para secretos.

✅ Nombrar todas las tareas.

✅ Evitar comandos shell innecesarios.

✅ Crear entornos separados:

```
dev
pre
pro
```

✅ Mantener playbooks pequeños.

## 16. Ejemplo completo

```yaml
---
- name: Despliegue aplicación
  hosts: web
  become: yes

  vars:
    app_dir: /opt/app

  tasks:

    - name: Instalar nginx
      apt:
        name: nginx
        state: present

    - name: Crear directorio
      file:
        path: "{{ app_dir }}"
        state: directory

    - name: Copiar configuración
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify:
        - Reiniciar nginx

  handlers:

    - name: Reiniciar nginx
      service:
        name: nginx
        state: restarted
```

Si dominas **roles, templates, variables, register, handlers y vault**, ya estás bastante cerca de lo que se usa de verdad en muchos entornos. El siguiente salto sería un nivel "DevOps serio": Galaxy, Collections, Ansible AWX, Tower, CI/CD con Ansible, Docker, Kubernetes y automatización multi-entorno. Ahí es donde los YAML empiezan a reproducirse solos por las noches y nadie recuerda quién escribió el playbook original.
