# servidor-casa

Infraestructura compartida del servidor de casa (Rocky Linux):

- **Caddy**: proxy inverso. Recibe todo el tráfico web (puertos 80 y 443) y lo envía a cada proyecto según el nombre con el que se entra: `notas.home.arpa` va a Notas, `otro.home.arpa` a otro proyecto. Puede servir https con certificados propios, gratis.
- **AdGuard Home**: DNS de la red. Hace que cualquier `*.home.arpa` apunte a este servidor, así no hay que tocar el DNS al agregar proyectos. De paso, bloquea publicidad en toda la casa.

```
dispositivo ──► AdGuard Home (DNS): "notas.home.arpa es 192.168.1.50"
dispositivo ──► Caddy :80/:443 ──red "proxy"──► notas-web (proyecto Notas)
                                             └─► otro-web  (otro proyecto)
```

`.home.arpa` es el sufijo reservado oficialmente para redes domésticas: no choca con ningún dominio de internet.

En los ejemplos, el servidor tiene la IP `192.168.1.50`. La preparación del servidor (IP fija, SSH, usuarios, Docker y firewall) está en la guía `docs/rocky-linux.md` del repositorio Notas.

---

## 1. Instalar

```bash
cd /opt
git clone https://github.com/FrankVDev/servidor-casa.git infra
cd infra
docker compose up -d
docker compose ps        # caddy y adguard en "running"
```

Esto crea la red compartida `proxy` (subred fija `10.250.250.0/24`) que usan los proyectos. **Levanta esto antes que los proyectos.**

Si AdGuard no arranca por el puerto 53 ocupado, revisa qué lo usa con `sudo ss -lunp | grep ':53 '`.

## 2. Configurar AdGuard Home

1. Abre `http://192.168.1.50:3000` desde tu PC.
2. En el asistente, en **Interfaz web de administración**, elige **todas las interfaces** y el **puerto 3000** (no el 80: ese es de Caddy). En **Servidor DNS**, todas las interfaces y el puerto 53.
3. Crea el usuario y la contraseña del panel.
4. Ve a **Filtros → Reescrituras DNS → Añadir reescritura DNS**:
   - Dominio: `*.home.arpa`
   - Respuesta: `192.168.1.50`

   Con esta sola regla, cualquier nombre `algo.home.arpa` apunta al servidor; los proyectos nuevos solo se agregan en Caddy.
5. Comprueba desde tu PC:

   ```powershell
   nslookup notas.home.arpa 192.168.1.50
   ```

   Debe responder `192.168.1.50`.

Desde ahora, el panel también está en `http://dns.home.arpa`.

## 3. Que toda la casa use este DNS

En la configuración de tu router, busca la sección **DHCP** o **LAN** y pon `192.168.1.50` como **servidor DNS** (primario). Los dispositivos lo toman al reconectarse al wifi.

Dos cosas importantes:

- **No pongas un DNS público como secundario** (8.8.8.8, 1.1.1.1). Los dispositivos usan cualquiera de los dos al azar, y los nombres `.home.arpa` fallarían de forma intermitente.
- **Si el servidor se apaga, la casa se queda sin DNS**, es decir, sin internet en la práctica. Por eso AdGuard se reinicia solo (`restart: unless-stopped`). Si prefieres no depender del servidor, en lugar del router configura el DNS solo en los dispositivos que usarán los proyectos.

Si tu router no permite cambiar el DNS, AdGuard Home puede hacer de servidor DHCP (Ajustes → Ajustes de DHCP), apagando antes el DHCP del router.

**Android con DNS privado** (por ejemplo Cloudflare): ignora el DNS de la red, así que los nombres `.home.arpa` no resolverán. Pon el DNS privado en *Automático* o usa Tailscale (sección 6).

## 4. Agregar un proyecto

Cada proyecto vive en su carpeta (`/opt/proyectos/nombre`) con su propio `docker-compose.yml`.

**En el proyecto**, el servicio web no publica puertos y se une a la red `proxy` con un nombre propio:

```yaml
services:
  web:
    # sin "ports:"
    networks:
      default:
      proxy:
        aliases:
          - inventario-web

networks:
  proxy:
    external: true
```

**En `caddy/Caddyfile`**, un bloque nuevo:

```
http://inventario.home.arpa {
	reverse_proxy inventario-web:80
}
```

Recarga Caddy (sin cortar los demás sitios) y sube el cambio:

```bash
cd /opt/infra
docker compose exec -w /etc/caddy caddy caddy reload
git commit -am "Agregar inventario" && git push
```

El nombre ya funciona gracias a la reescritura `*.home.arpa`.

Si un proyecto necesita la IP real de cada visitante, puede confiar en la cabecera `X-Forwarded-For` **solo** para conexiones que vengan de `10.250.250.0/24`. Caddy descarta esa cabecera cuando la envía un visitante, así que no se puede falsear.

## 5. HTTPS (gratis, con certificados propios)

Los nombres `.home.arpa` no pueden tener certificados públicos, así que Caddy crea su **propia autoridad de certificados** y emite y renueva los certificados solo. Basta con instalar su certificado raíz **una vez por dispositivo**.

1. En `caddy/Caddyfile`, quita `http://` del nombre del sitio:

   ```
   notas.home.arpa {
   	reverse_proxy notas-web:80
   }
   ```

   Caddy servirá https y redirigirá http a https automáticamente.

2. Recarga Caddy y extrae el certificado raíz:

   ```bash
   docker compose exec -w /etc/caddy caddy caddy reload
   docker compose cp caddy:/data/caddy/pki/authorities/local/root.crt ./caddy-raiz.crt
   ```

   Cópialo a tu PC (`scp notas:/opt/infra/caddy-raiz.crt .`). Es público: no da acceso a nada.

3. Instálalo en cada dispositivo:

   | Dispositivo | Cómo |
   |---|---|
   | Windows | Doble clic → Instalar certificado → Equipo local → *Entidades de certificación raíz de confianza*. O en PowerShell como administrador: `certutil -addstore -f Root caddy-raiz.crt` |
   | Android | Ajustes → Seguridad → Más ajustes de seguridad → Cifrado y credenciales → Instalar certificado → Certificado de CA (la ruta varía por marca) |
   | iPhone | Envíatelo por AirDrop o correo y ábrelo → Ajustes → Perfil descargado → Instalar. Luego Ajustes → General → Información → Ajustes de confianza de certificados → actívalo |
   | Firefox | Usa su propio almacén: Ajustes → Privacidad y seguridad → Certificados → Ver certificados → Autoridades → Importar |

4. En el `.env` de cada proyecto con inicio de sesión (como Notas), pon `COOKIE_SECURE=true` y ejecuta `docker compose up -d`.

El certificado raíz vale 10 años y está en el volumen `infra_caddy_data`. **Respáldalo** (sección 7): si se pierde, Caddy crea uno nuevo y habría que instalarlo otra vez en todos los dispositivos.

Más adelante, con un dominio propio (unos 10 a 15 dólares al año), Caddy puede obtener certificados de Let's Encrypt, reconocidos por todos los dispositivos sin instalar nada.

## 6. Acceso desde fuera de casa: Tailscale

Con Tailscale instalado en el servidor y en tus dispositivos (ver la guía de Notas), el servidor puede hacer de **puerta de entrada a la red de casa** ("subnet router"). Así, desde fuera, la IP `192.168.1.50` es alcanzable por Tailscale y la misma reescritura `*.home.arpa → 192.168.1.50` sirve dentro y fuera de casa, sin cambiar nada para los dispositivos que no usan Tailscale.

1. En el servidor, permite reenviar tráfico y anuncia la red de casa:

   ```bash
   echo 'net.ipv4.ip_forward = 1' | sudo tee /etc/sysctl.d/99-tailscale.conf
   sudo sysctl -p /etc/sysctl.d/99-tailscale.conf
   sudo tailscale up --advertise-routes=192.168.1.0/24
   ```

2. En el panel de Tailscale, en la máquina del servidor: **Edit route settings** → aprueba `192.168.1.0/24`.
3. Para los nombres (también en Android con DNS privado): **DNS → Nameservers → Add nameserver → Custom**, escribe `192.168.1.50` y activa **Restrict to domain** con `home.arpa` (split DNS).

En Android e iPhone, la app de Tailscale acepta las rutas anunciadas automáticamente; en Windows, actívalo en el menú de Tailscale con **Use Tailscale subnets**.

Si estás en otra red que también usa `192.168.1.x` (la casa de un amigo, un hotel), puede haber confusión de rutas. Si te pasa a menudo, cambia la red de tu casa a un rango menos común, por ejemplo `192.168.47.x`.

## 7. Respaldo de la infraestructura

La configuración está en git. Lo que no está en git y conviene respaldar:

- **`infra_caddy_data`**: la autoridad de certificados de Caddy.
- **`infra_adguard_conf`**: la configuración de AdGuard (reescrituras, filtros, usuario).

```bash
cd /opt/infra && mkdir -p respaldos
for v in caddy_data adguard_conf; do
  docker run --rm -v infra_$v:/datos:ro -v "$PWD/respaldos":/destino:z alpine \
    tar czf /destino/$v-$(date +%Y-%m-%d).tgz -C /datos .
done
```

La carpeta `respaldos/` no se sube a git. Cópiala fuera del servidor.

## 8. Comandos útiles

| Tarea | Comando |
|---|---|
| Recargar Caddy tras editar el Caddyfile | `docker compose exec -w /etc/caddy caddy caddy reload` |
| Comprobar el Caddyfile antes de recargar | `docker compose exec -w /etc/caddy caddy caddy validate` |
| Ver errores de Caddy | `docker compose logs -f caddy` |
| Actualizar las imágenes | `docker compose pull && docker compose up -d` |

## Problemas comunes

**`algo.home.arpa` muestra "Este nombre no está configurado".** El DNS funciona, pero falta el bloque en el `Caddyfile` o recargar Caddy.

**Caddy responde 502.** No llega al proyecto: comprueba que el proyecto esté en marcha y unido a la red `proxy` con el mismo alias que usa el `Caddyfile` (`docker network inspect proxy`).

**El nombre no resuelve.** Comprueba con `nslookup notas.home.arpa 192.168.1.50`. Si responde bien, el dispositivo no está usando AdGuard como DNS (revisa el router, o el DNS privado en Android).

**El navegador advierte "la conexión no es privada".** Falta instalar el certificado raíz de Caddy en ese dispositivo (sección 5).
