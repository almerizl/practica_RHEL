[Laboratorios del Modulo VI.txt](https://github.com/user-attachments/files/29934692/Laboratorios.del.Modulo.VI.txt)
================================================================================
REPORTE DE LABORATORIO: Laboratorios del Modulo VI
Estudiante: Cris Almeris Zanellato Lorenzo
Matrícula: 2025-2285
Materia: Sistemas Operativos 3
================================================================================

================================================================================
PRÁCTICA 1: CIFRADO DE DATOS Y RESGUARDO DE CONFIDENCIALIDAD (GPG2)
================================================================================
COMANDOS EJECUTADOS:

# Instalación del utilitario nativo criptográfico
   dnf install gnupg2 -y

# Creación del entorno de trabajo y archivo con datos sensibles
   mkdir ~/practica_cifrado && cd ~/practica_cifrado
   echo "esta es mi practica #1 de cifrado - cris almeris Zanellato lorenzo" > secreto.txt

# Cifrado simétrico del archivo (Solicita el ingreso de una clave/passphrase)
   gpg2 -c secreto.txt

COMPROBACIÓN:

- Al ejecutarse el cifrado se genera el archivo protegido 'secreto.txt.gpg'.
- Intento de lectura del archivo cifrado:
     cat secreto.txt.gpg

- Descifrado y recuperación de la información:
     rm -f secreto.txt
     gpg2 -d secreto.txt.gpg > secreto.txt
     cat secreto.txt

================================================================================
PRÁCTICA 2: SEGURIDAD PERIMETRAL Y CONTROL DE PUERTOS (IPTABLES / FIREWALL-CMD)
================================================================================
COMANDOS EJECUTADOS:

# Asegurar el estado operativo de los demonios perimetrales
   systemctl start sshd httpd vsftpd
   systemctl status sshd httpd vsftpd

# Inyección de políticas DROP en IPTABLES para aislar los tres puertos de red
   iptables -A INPUT -p tcp --dport 80 -j DROP
   iptables -A INPUT -p tcp --dport 21 -j DROP
   iptables -A INPUT -p tcp --dport 22 -j DROP
   iptables -L -n -v

# Remoción de las reglas aplicadas en IPTABLES para restaurar flujo libre
   iptables -D INPUT -p tcp --dport 80 -j DROP
   iptables -D INPUT -p tcp --dport 21 -j DROP
   iptables -D INPUT -p tcp --dport 22 -j DROP

# Habilitación y bloqueo usando Rich Rules en FIREWALL-CMD
   systemctl start firewalld
   firewall-cmd --permanent --add-rich-rule='rule family="ipv4" port port="80" protocol="tcp" reject'
   firewall-cmd --permanent --add-rich-rule='rule family="ipv4" port port="21" protocol="tcp" reject'
   firewall-cmd --permanent --add-rich-rule='rule family="ipv4" port port="22" protocol="tcp" reject'
   firewall-cmd --reload
   firewall-cmd --list-all

# Limpieza absoluta y deshabilitación de políticas enriquecidas
   firewall-cmd --permanent --remove-rich-rule='rule family="ipv4" port port="80" protocol="tcp" reject'
   firewall-cmd --permanent --remove-rich-rule='rule family="ipv4" port port="21" protocol="tcp" reject'
   firewall-cmd --permanent --remove-rich-rule='rule family="ipv4" port port="22" protocol="tcp" reject'
   firewall-cmd --reload

COMPROBACIÓN:

- Los servicios cambian su estado de forma inmediata a 'Active (running)'.
- Al aplicar 'iptables -L -n -v' se visualizan las reglas activas con contadores en 0.
- Mediante 'firewall-cmd --list-all' se constata la inyección de las Rich Rules de rechazo.

================================================================================
PRÁCTICA 3: AUDITORÍA DE RED Y ANÁLISIS DE TRÁFICO EN TIEMPO REAL (TCPDUMP)
================================================================================
COMANDOS EJECUTADOS:

# Creación de la estructura lógica de firmas Snort requerida
   mkdir ~/practica_snort && cd ~/practica_snort
   cat << 'EOF' > snort.rules
   alert icmp any any -> any any (sid:1;)
   alert tcp any any -> any 21 (sid:2;)
   alert tcp any any -> any 22 (sid:3;)
   alert tcp any any -> any 80 (sid:4;)
   EOF
   cat snort.rules

# Detener el cortafuegos local para permitir auditoría limpia en la interfaz
   systemctl stop firewalld

# Escucha activa de tráfico ICMP (Ping)
   tcpdump -i ens160 icmp

# Escucha activa de conexiones al puerto 21 (FTP)
   tcpdump -i ens160 port 21

# Escucha activa de conexiones al puerto 22 (SSH)
   tcpdump -i ens160 port 22

# Escucha activa de tráfico web al puerto 80 (HTTP)
   tcpdump -i ens160 port 80

COMPROBACIÓN:

- El comando 'cat snort.rules' valida la persistencia del archivo de firmas base.
- Inyección ICMP desde Host: 'ping 192.168.124.132' -> Captura tramas echo request/reply.
- Inyección FTP desde Host: 'curl 192.168.124.132:21' -> Captura tráfico TCP port 21.
- Inyección SSH desde Host: 'curl 192.168.124.132:22' -> Captura el saludo inicial TCP port 22.
- Inyección HTTP desde Host: 'curl http://192.168.124.132' -> Captura tramas HTTP port 80.

================================================================================
PRÁCTICA 4: IMPLEMENTACIÓN DE AUTENTICACIÓN MULTIFACTOR (2FA) PARA ACCESO SSH
================================================================================
COMANDOS EJECUTADOS:

# Restauración temporal de repositorios, limpieza e instalación del módulo PAM
   mv /etc/yum.repos.d/backup_repos/* /etc/yum.repos.d/ 2>/dev/null || mv ~/backup_repos/* /etc/yum.repos.d/
   dnf clean all
   dnf install google-authenticator -y

# Inicialización del asistente interactivo de generación de semillas TOTP
   google-authenticator

# Vinculación del factor de autenticación obligatorio en la pila PAM
   echo "auth required pam_google_authenticator.so" >> /etc/pam.d/sshd

# Endurecimiento del demonio SSHD mediante la inyección directa de políticas
   sed -i 's/KbdInteractiveAuthentication no/KbdInteractiveAuthentication yes/g' /etc/ssh/sshd_config
   sed -i 's/PermitRootLogin no/PermitRootLogin yes/g' /etc/ssh/sshd_config
   sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config

# Actualización obligatoria de clave administrativa y reinicio del servicio
   passwd root
   systemctl restart sshd
