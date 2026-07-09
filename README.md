# BRM Umbrel Store

Tienda comunitaria de apps para [Umbrel OS](https://umbrel.com), orientada a
domótica local-first con **Home Assistant Container** (sin Supervisor) sobre
Raspberry Pi. Incluye también el código fuente y la build de la app Onda.

## Instalación de la tienda

En Umbrel: **App Store → ⋮ → Community App Stores** → añadir la URL:

    https://github.com/BorjaRodMar/brm-umbrel-store

Las apps aparecerán en la tienda "BRM Umbrel Store".

## Apps

| App | Descripción | Puerto |
|---|---|---|
| **Matter Hub** | [home-assistant-matter-hub](https://github.com/riddix/home-assistant-matter-hub): bridge que expone entidades de Home Assistant a controladores Matter (Alexa, Google Home…) | 8482 |
| **Govee2MQTT** | [wez/govee2mqtt](https://github.com/wez/govee2mqtt): puente de dispositivos Govee a MQTT con Home Assistant discovery | 8056 |
| **Ring-MQTT** | [tsightler/ring-mqtt](https://github.com/tsightler/ring-mqtt): puente de dispositivos Ring (alarma, sensores, timbres, cámaras) a MQTT con Home Assistant discovery y pasarela RTSP para streaming | 55123 |
| **Onda** | Extrae el audio de vídeos de YouTube y tracks de SoundCloud a MP3 (yt-dlp + FFmpeg). Solo para contenido propio o con licencia libre | 8480 |

---

### Matter Hub

Requiere un **token de acceso de larga duración** de Home Assistant, que se
configura **en local tras instalar** (los secretos no viven en este repo y
umbreld no sustituye variables de entorno al instalar):

    nano ~/umbrel/app-data/brm-matter-hub/docker-compose.yml

Sustituir `${HAMH_TOKEN}` por el token (HA → perfil → Seguridad → Tokens de
acceso de larga duración) y ajustar `HAMH_HOME_ASSISTANT_URL` a la IP de tu
Home Assistant. Reiniciar:

    sudo umbreld client apps.restart.mutate --appId brm-matter-hub

Notas:

- Usa `network_mode: host` y anuncia mDNS por la interfaz definida en
  `HAMH_MDNS_NETWORK_INTERFACE` (por defecto `end0`, la Ethernet de la
  Raspberry Pi 5). Ajústala a tu hardware.
- No puede convivir con otra pila Matter en el mismo host (p. ej. el Matter
  Server oficial de HA): ambas pelean por el puerto mDNS 5353.
- La identidad Matter (fabric) vive en `app-data/brm-matter-hub/data/`.
  Haz backup de ese directorio; restaurándolo tras una reinstalación se
  conserva el emparejamiento sin re-vincular con Alexa.

### Govee2MQTT

Requiere un broker MQTT accesible desde el host (p. ej. la app **Mosquitto**
de Umbrel con el puerto 1883 publicado) y una **API key de Govee**
(app Govee Home → perfil → ⚙️ → "Apply for API key").

La app se instala **sin** API key a propósito: con una key inválida el
servicio entra en crash-loop y Umbrel aborta la instalación. Tras instalar,
añadirla en local:

    nano ~/umbrel/app-data/brm-govee2mqtt/docker-compose.yml

En `environment:` del servicio `server`, añadir (ajustando también
`GOVEE_MQTT_HOST` a la IP de tu broker):

    GOVEE_API_KEY: "tu-api-key"

Y reiniciar:

    sudo umbreld client apps.restart.mutate --appId brm-govee2mqtt

Notas:

- Sin `GOVEE_EMAIL`/`GOVEE_PASSWORD`: el login a la API no documentada de
  Govee (AWS IoT, push en tiempo real) falla con error 454
  ([issue #76](https://github.com/wez/govee2mqtt/issues/76)). La app
  funciona con Platform API (sondeo periódico) + LAN API, suficiente para
  la mayoría de dispositivos.
- En Home Assistant, con la integración **MQTT** apuntando al broker, los
  dispositivos aparecen solos vía MQTT discovery.

### Ring-MQTT

Requiere un broker MQTT accesible desde el host (app **Mosquitto** de Umbrel,
puerto 1883) y una cuenta Ring con **2FA** activado.

**Tras instalar hay que crear el `config.json` a mano.** ring-mqtt v5 NO lo
autogenera desde variables de entorno en una instalación limpia: si falta
`/data/config.json` el contenedor arranca, avisa y se apaga. Averigua la ruta
del volumen y crea el fichero apuntando a tu broker:

    DATADIR=$(sudo docker inspect brm-ring-mqtt_server_1 --format '{{ range .Mounts }}{{ if eq .Destination "/data" }}{{ .Source }}{{ end }}{{ end }}')
    printf '{
  "mqtt_url": "mqtt://192.168.7.79:1883"
}
' | sudo tee "$DATADIR/config.json"
    sudo umbreld client apps.restart.mutate --appId brm-ring-mqtt

Ajusta la IP a la de tu broker. Después, el token de Ring (2FA) se genera desde
la web UI de la app (puerto 55123) y se guarda en `/data/ring-state.json`. Abre:

    http://<ip-o-host>:55123

Introduce usuario/contraseña de Ring y el código 2FA. El token queda guardado y
se renueva solo.

Notas:

- Usa `network_mode: host` para el streaming de cámaras: bincula directamente
  la web UI (55123), RTSP (8554) y go2rtc/WebRTC (8555) en la IP del host, de
  modo que Home Assistant pueda consumir los streams.
- El `config.json` y el token viven en `app-data/brm-ring-mqtt/data/`. Haz
  backup de ese directorio; sobrevive a reinicios y actualizaciones, pero **se
  borra al desinstalar** la app (habría que recrear el config y repetir el 2FA).
- En Home Assistant, con la integración **MQTT** activa, los dispositivos Ring
  aparecen solos vía MQTT discovery.

### Onda

Sin configuración. El código fuente vive en [`app-src/`](app-src/) y la
imagen multi-arch (`linux/arm64`, `linux/amd64`) se compila y publica en
GHCR automáticamente con [GitHub Actions](.github/workflows/build.yml)
cuando cambia el código.

---

## Mantenimiento

- **Actualizar secretos tras reinstalar/actualizar**: las ediciones locales
  de `app-data/*/docker-compose.yml` (token, API key) se pierden si la app
  se reinstala desde la tienda. Hay que volver a aplicarlas.
- **CLI de apps** en Umbrel OS moderno:
  `sudo umbreld client apps.<install|uninstall|restart|stop|start>.mutate --appId <id>`
- Solo `/home/umbrel/` persiste entre reinicios en Umbrel OS; por eso estas
  apps se empaquetan como apps nativas en vez de contenedores manuales.

## Estructura del repo

    umbrel-app-store.yml     # id y nombre de la tienda
    brm-matter-hub/          # manifiesto + compose de Matter Hub
    brm-govee2mqtt/          # manifiesto + compose de Govee2MQTT
    brm-ring-mqtt/           # manifiesto + compose de Ring-MQTT
    brm-yt2mp3/              # manifiesto + compose de Onda
    app-src/                 # código fuente de Onda
    .github/workflows/       # build multi-arch de la imagen de Onda
