# Cuaderno de bitácora

Registro cronológico del proyecto: qué se ha hecho, por qué, qué aporta en seguridad, qué problemas aparecieron y cuál es el siguiente paso. Las entradas más recientes van arriba.

---

## 2026-10-15 · Fase 0 · Planificación y arranque

### Qué se ha hecho
- Definido el alcance del laboratorio: pipeline DevSecOps completo (commit → CI con escaneos → imagen firmada → GitOps → políticas en clúster → detección en runtime) con coste 0 €.
- Elegido **Oracle Cloud Always Free** como plataforma de despliegue, descartando GCP.
- Iniciada la creación de la cuenta de OCI. En el asistente de instancia está seleccionada la shape `VM.Standard.A1.Flex` con la imagen Ubuntu 24.04 Minimal aarch64.
- Elegido **WSL** como entorno de trabajo local.
- Escrito el **cloud-init de bootstrap** (`cloud-init/user-data.yaml`), validado con `cloud-init schema`. Aún no se ha subido al repositorio ni probado en una VM real.
- Creado este repositorio.

### Pendiente de confirmar
- Ajustar la shape a **2 OCPU / 12 GB** y crear la instancia.
- Preparar WSL: clave SSH Ed25519 dedicada, `~/.ssh/config` con alias `oci` y `keychain`.
- Restringir el puerto 22 de la Security List a la IP propia (`/32`).

### Decisiones y motivos
- **Oracle en lugar de GCP**: GCP ofrece 300 $ durante 90 días, pero su VM Always Free (e2-micro, 1 GB de RAM) no da para k3s y el stack de seguridad. La A1 de Oracle es gratuita de forma indefinida y tiene RAM suficiente.
- **Límites de Oracle**: en junio de 2026 Oracle redujo la cuota Always Free de A1 de 4 OCPU / 24 GB a **2 OCPU / 12 GB** sin anunciarlo, y ha llegado a desactivar instancias que la superaban ([InfoQ](https://www.infoq.com/news/2026/07/oracle-cloud-free-tier-limits/), [documentación de Oracle](https://docs.oracle.com/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm)). Se dimensiona dentro del límite y la cuenta se mantiene como free (sin PAYG).
- **Una sola VM de 2/12** con k3s de un nodo, para dejar toda la RAM al stack. Wazuh queda fuera por consumo de memoria.
- **GitHub en lugar de GitLab**: Actions es ilimitado en repositorios públicos ([GitHub](https://docs.github.com/en/billing/concepts/product-billing/github-actions)), mientras que GitLab.com Free da 400 minutos al mes ([GitLab](https://docs.gitlab.com/ci/pipelines/compute_minutes/)).
- **WSL + OpenSSH en lugar de PuTTY**: mismo flujo que en Linux y sin formato `.ppk`. Regla: el trabajo vive en el sistema de archivos de Linux (`~/proyectos`), nunca en `/mnt/c`, porque los permisos de NTFS rompen SSH, Git y Ansible.
- **cloud-init mínimo**: solo el bootstrap imprescindible para arrancar seguro. El resto (k3s, CIS) irá en Ansible, versionado y repetible.

### Seguridad prevista en el cloud-init
- **SSH**: solo clave pública, sin root ni contraseñas, `AllowUsers ubuntu`, `MaxAuthTries 3` y reenvío solo local (para usar `kubectl` por túnel sin exponer el puerto 6443). Se valida con `sshd -t` antes de reiniciar el servicio.
- **fail2ban** en SSH con `bantime.increment`.
- **unattended-upgrades** con reinicio automático a las 04:30.
- **sysctl** endurecido y compatible con k3s.
- **auditd** vigilando `passwd`, `shadow`, `sudoers`, la configuración de SSH y `authorized_keys`.
- **iptables de OCI**: quita el `REJECT` de FORWARD (rompe la red entre pods) y abre 80, 443 y las redes de k3s. Guarda una copia en `rules.v4.orig` y es idempotente (probado con una copia de las reglas de OCI).

### Problemas y aprendizajes
- La consola de OCI muestra por defecto 1 OCPU / 6 GB en A1.Flex. Para ajustarla hay que desplegar la fila con la flecha y comprobar que se mantiene *Always Free-eligible*.
- Las imágenes de Oracle traen reglas de iptables propias: abrir un puerto en la Security List no basta ([documentación de OCI](https://docs.oracle.com/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm)).
- Oracle puede recuperar instancias Always Free si durante 7 días la CPU (p95), la red y la memoria se quedan por debajo del 20 %.

### Siguiente paso
- Crear la instancia con el cloud-init y verificarla (`cloud-init status --long`, `sshd -T`, `fail2ban-client status sshd`, `iptables -L`).
- Pasar la creación de la VM a **Terraform**, escaneado con Checkov en GitHub Actions.
