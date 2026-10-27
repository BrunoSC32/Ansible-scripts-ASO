# Guía de instalación del proyecto ASO

**Escenario:** dos máquinas virtuales openSUSE Leap 16 con entorno gráfico y conexión a Internet.

- **VM 1 — Controladora:** ejecuta Ansible.
- **VM 2 — Destino:** recibe los servicios.

Ejecuta cada comando en la terminal de la VM indicada.

## 1. Preparar la red de ambas máquinas

Configura las dos VM en una red que permita la comunicación entre ellas (por ejemplo, adaptador puente o una red NAT compartida).

En ambas VM:

```bash
ip -4 addr
```

Anota la IP de la VM 2. Desde la VM 1:

```bash
ping -c 4 IP_VM2
```

## 2. VM 2: habilitar el servidor SSH

```bash
sudo zypper refresh
sudo zypper install -y openssh-server python3
sudo systemctl enable --now sshd
sudo systemctl status sshd
```

Si `firewalld` está activo, permite SSH:

```bash
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --reload
```

## 3. VM 1: instalar herramientas

```bash
sudo zypper refresh
sudo zypper install -y git ansible openssh-clients
ansible --version
```

Instala la colección utilizada para administrar PostgreSQL:

```bash
ansible-galaxy collection install community.postgresql
```

## 4. VM 1: descargar el repositorio

```bash
cd ~
git clone https://github.com/BrunoSC32/Ansible-scripts-ASO.git
cd Ansible-scripts-ASO
```

No necesitas copiar los archivos YAML manualmente a la VM 2. Ansible realiza la configuración de forma remota mediante SSH.

## 5. VM 1: configurar el acceso SSH

```bash
ssh-keygen -t ed25519
ssh-copy-id USUARIO_VM2@IP_VM2
ssh USUARIO_VM2@IP_VM2
```

Sustituye `USUARIO_VM2` por el usuario de la VM destino e `IP_VM2` por su dirección IP. La primera conexión puede solicitar confirmar la huella SSH.

## 6. VM 1: preparar el inventario `hosts`

Desde la carpeta del repositorio:

```bash
nano hosts
```

Deja el archivo así, usando los datos de tu VM 2:

```ini
[servidores]
opensuse ansible_host=IP_VM2 ansible_user=USUARIO_VM2 ansible_python_interpreter=/usr/bin/python3
```

Guarda el archivo.

## 7. VM 1: comprobar conexión y sintaxis

```bash
ansible servidores -m ping
ansible-playbook site.yml --syntax-check
```

La primera orden debe devolver `pong`; la segunda debe validar la sintaxis del playbook.

## 8. VM 1: ejecutar instalación remota

```bash
ansible-playbook site.yml --ask-become-pass
```

Introduce la contraseña de `sudo` del usuario de la VM 2 cuando se solicite. En el resumen final comprueba que `failed=0` y `unreachable=0`.

## 9. VM 2: revisar servicios instalados

```bash
sudo systemctl status apache2
sudo systemctl status mysql
sudo systemctl status postgresql
sudo systemctl status smb
curl http://localhost/aso-check.php
```

Para Apache/PHP se espera `ASO_OK`. Desde la VM 1 también puedes abrir `http://IP_VM2/aso-check.php` si el firewall permite HTTP.

## 10. Probar Samba y PostgreSQL

En la VM 2:

```bash
sudo testparm -s
sudo pdbedit -L
sudo -u postgres psql -c "SELECT version();"
```

Desde otra computadora de la red, prueba Samba en `\\IP_VM2\compartido` (Windows) o `smb://IP_VM2/compartido` (gestor de archivos Linux). Para PostgreSQL, comprueba además una conexión remota con el usuario creado por el playbook.
