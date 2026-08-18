# Despliegue de infraestructura web con Nginx en AWS

Laboratorio práctico de infraestructura: despliegue de dos sitios web estáticos independientes sobre una misma instancia EC2, con enrutamiento DNS, Nginx como servidor web, y certificados SSL/TLS automatizados con Certbot (Let's Encrypt).

> **Nota:** este laboratorio se realizó sobre una cuenta sandbox de AWS de duración temporal (4 horas). Los servidores ya no están activos, por lo que este repositorio documenta el proceso mediante configuraciones reales y capturas de pantalla del despliegue.

## Arquitectura

```
Usuario → DNS (Route 53) → Nginx (EC2 / Ubuntu)
                              ├── Sitio predeterminado (default)
                              ├── Sitio estático 1 (web.*)
                              └── Sitio estático 2 (app.*)

Let's Encrypt → Certbot → Certificados SSL/TLS → Nginx
```

## Qué se hizo

1. **Provisioning:** instancia EC2 (Ubuntu Server) con acceso SSH restringido a IP propia, puertos 80/443 habilitados y llave privada descargada.
2. **Instalación de Nginx** y verificación del despliegue por defecto.
3. **Gestión DNS:** creación de registros tipo A en Route 53 para dos subdominios distintos apuntando a la instancia.
4. **Hosting múltiple:** una carpeta y un bloque `server` independiente por sitio en `sites-available/`, activados mediante enlaces simbólicos en `sites-enabled/`.
5. **Validación y recarga** de configuración sin downtime (`nginx -t` + `systemctl reload`).
6. **SSL/TLS con Certbot:** emisión automática de certificados para ambos dominios, migrando el tráfico de HTTP (80) a HTTPS (443).
7. **Renovación automática:** verificación con `--dry-run` y confirmación del timer de systemd que gestiona la renovación cada 90 días.
8. **Acceso remoto:** configuración de `~/.ssh/config` para conectarse con `ssh servidor-nginx` sin parámetros adicionales.

## Configuración de Nginx

Cada sitio tiene su propio bloque de servidor en [`nginx/sites-available/`](nginx/sites-available/):

- [`web.741960641592.realhandsonlabs.net.conf`](nginx/sites-available/web.741960641592.realhandsonlabs.net.conf) — primer sitio estático
- [`app.741960641592.realhandsonlabs.net.conf`](nginx/sites-available/app.741960641592.realhandsonlabs.net.conf) — segundo sitio estático

Activación mediante enlace simbólico:

```bash
sudo ln -s /etc/nginx/sites-available/web.741960641592.realhandsonlabs.net.conf /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

## SSL/TLS con Certbot

```bash
# Preparar el comando certbot
sudo ln -s /snap/bin/certbot /usr/local/bin/certbot

# Emitir certificados para los sitios detectados automáticamente
sudo certbot --nginx

# Probar la renovación automática (sin aplicar cambios reales)
sudo certbot renew --dry-run
```

Certbot modifica automáticamente los bloques `server` para escuchar en el puerto 443, y valida la propiedad del dominio mediante un desafío HTTP (`HTTP-01`) contra el DNS configurado en Route 53. La renovación queda gestionada por un timer de systemd que se ejecuta dos veces al día.

## Acceso SSH simplificado

Ver [`ssh/config`](ssh/config). Con esta configuración, conectarse al servidor se reduce a:

```bash
ssh servidor-nginx
```

## Evidencia (capturas del despliegue)

| Paso | Captura |
|---|---|
| Instancia EC2 en ejecución | ![EC2](/screenshots/01-ec2-instancia-running.png) |
| Nginx instalado y funcionando | ![Nginx](/screenshots/02-nginx-instalado.png) |
| Creando registros DNS en Route 53 | ![Route53](/screenshots/03-route53-dns-records.png) |
| Registros DNS en Route 53 | ![Route53](/screenshots/03.1-registros-creados.png) |
| Enlaces simbólicos en `sites-enabled` | ![Symlinks](\screenshots/04-enlaces-simbolicos-sites-enabled.png) |
| Sitio funcionando por HTTP | ![HTTP](/screenshots/05-sitio-http-funcionando.png) |
| Sitio funcionando por HTTPS | ![HTTPS](/screenshots/06-sitio-https-funcionando.png) |
| Sitio  HTTPS desde Telefono  | ![HTTPS](/screenshots/06.1-prueba-de-seguridad.jpg) |
| Renovación automática probada (`--dry-run`) | ![Dry run](/screenshots/07-certbot-dry-run-exitoso.png) |
| Próxima renovación programada | ![Renovación](/screenshots/08-certbot-renovacion-programada.png) |

## Tecnologías utilizadas

`AWS EC2` · `AWS Route 53` · `Nginx` · `Certbot / Let's Encrypt` · `SSH` · `Linux · Ubuntu` · `Systemd Timers` · `DNS (A Record)` · `SSL/TLS`

## Cómo funciona Let's Encrypt + Certbot

- **Let's Encrypt** es la autoridad certificadora (CA): emite los certificados SSL/TLS de forma gratuita.
- **Certbot** es el cliente que se comunica con Let's Encrypt: solicita el certificado, valida la propiedad del dominio, lo instala en Nginx y gestiona su renovación automática.
- El **timer de systemd** (`snap.certbot.renew.timer`) ejecuta periódicamente el proceso de renovación, evitando que el certificado expire sin intervención manual.
