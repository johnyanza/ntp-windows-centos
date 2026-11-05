# ntp-windows-centos
• Guía para sincronizar hora de Windows con CentOS.
# Sincronización de Hora: Cliente Windows a Servidor NTP CentOS

Este documento detalla el procedimiento para configurar un servidor CentOS como fuente de tiempo principal (NTP) y forzar a un cliente Windows con la hora desactualizada a sincronizarse con él.

## 1. Configuración del Servidor CentOS (NTP)

Dependiendo de la versión de CentOS, el servicio de tiempo varía entre `chrony` (CentOS 8+) o `ntp` (CentOS 7).

### Paso A: Permitir la red local en la configuración
Editar el archivo de configuración correspondiente para permitir peticiones desde el rango de IPs de la red local (ejemplo: `10.21.165.0/24`).

* **Para Chrony (`/etc/chrony.conf`):**
  ```bash
  allow 10.21.165.0/24
  ```
* **Para NTPd (`/etc/ntp.conf`):**
  ```bash
  restrict 10.21.165.0 mask 255.255.255.0 nomodify notrap
  ```

### Paso B: Habilitar el servicio y abrir el Firewall
Asegurar que el puerto **123 UDP** esté abierto para el tráfico NTP.

```bash
# Iniciar y habilitar el servicio (Ejemplo con Chrony)
sudo systemctl enable --now chronyd
sudo systemctl restart chronyd

# Abrir el puerto en el firewall
sudo firewall-cmd --add-service=ntp --permanent
sudo firewall-cmd --reload
```

---

## 2. Configuración del Cliente Windows

Los siguientes comandos se deben ejecutar en una terminal de **Símbolo del sistema (CMD)** con privilegios de **Administrador**.

### Paso A: Configurar el Servidor de Origen
Establecer la IP del servidor CentOS (en este ejemplo, `10.21.165.226`) como el par manual NTP.

```cmd
w32tm /config /manualpeerlist:"10.21.165.226,0x8" /syncfromflags:manual /update
```
*(Nota: El flag `0x8` asegura que Windows envíe solicitudes en un intervalo de tiempo regular basado en la configuración del cliente).*

### Paso B: Forzar la Sincronización Inicial
```cmd
w32tm /resync /rediscover
```

### Paso C: Asegurar el inicio automático del servicio
Para evitar que la configuración se pierda tras un reinicio del sistema:
```cmd
sc config w32time start= auto
```

---

## 3. Verificación del Estado

Para comprobar que Windows está absorbiendo correctamente la hora del servidor CentOS, se utiliza el siguiente comando:

```cmd
w32tm /query /status
```

### Indicadores de éxito a verificar en el output:
* **Origen:** Debe mostrar la IP del servidor CentOS (`10.21.165.226,0x8`).
* **Capa (Stratum):** Debe mostrar un valor válido (ej. `4 (referencia secundaria)`), lo que indica que el tiempo es jerárquicamente confiable.
* **Última sincronización de hora correcta:** Debe reflejar una fecha y hora recientes.

---

## 4. Prueba de Estrés / Validación Manual

Para certificar que el mecanismo de corrección automática funciona:
1. Cambiar la hora de Windows a una hora incorrecta de forma deliberada: `time 10:00:00`
2. Forzar la sincronización inmediata: `w32tm /resync /force`
3. El reloj del sistema debería corregirse instantáneamente regresando a la hora exacta del servidor CentOS.
