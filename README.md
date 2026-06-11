# ChirpStack — despliegue con Ansible

Stack LoRaWAN completo (ChirpStack v4 + PostgreSQL + Redis + Mosquitto +
gateway-bridge + nginx) desplegado en un VPS Ubuntu con Ansible.

## Requisitos en la máquina de control (Mac/Linux)

Ansible **no corre como nodo de control en Windows**; usa una Mac, Linux o WSL.

```bash
# 1. Ansible + sshpass (sshpass es necesario porque ansible.cfg usa ask_pass)
brew install ansible sshpass          # macOS
# sudo apt install ansible sshpass    # Ubuntu/Debian

# 2. Colecciones que usan los roles (una sola vez)
ansible-galaxy collection install -r requirements.yml
```

> No hace falta passphrase de vault: actualmente no hay ficheros cifrados.
> Las contraseñas usan valores temporales inseguros (ver **Secretos** abajo).

## Requisitos del servidor destino

- Ubuntu (Server o Desktop, 64-bit; probado en 24.04 y 26.04).
- Acceso SSH como `root` (la contraseña se pide al arrancar el playbook).
- La IP se configura en `inventory/hosts.yml` → `ansible_host`.

## Uso

```bash
# Probar conectividad
ansible-playbook playbooks/ping.yml

# SOLO el núcleo ChirpStack + nginx (sin REST API, sin gateways, sin firewall)
ansible-playbook playbooks/chirpstack.yml

# Stack ChirpStack COMPLETO (REST API + registro de gateways + nginx + firewall)
ansible-playbook playbooks/site.yml

# Solo una parte, por tags
ansible-playbook playbooks/site.yml --tags gateways
```

Tras el despliegue: UI de ChirpStack en `http://<IP-del-VPS>/` (ojo: con
`http://` explícito — no hay HTTPS todavía). Login inicial `admin/admin`:
**cámbialo inmediatamente**.

## Pasos manuales (una sola vez por servidor)

1. **API key de admin** (la necesita el rol `chirpstack_gateway`): créala en la
   UI → API Keys y pégala en `/etc/chirpstack/.ansible_api_key` (mode 0600).
   El rol `chirpstack_api_key` falla con instrucciones detalladas si falta.

## Secretos

⚠ **Por defecto NO hay vault**: la contraseña de Postgres y el `api_secret` de
ChirpStack usan valores temporales **INSEGUROS**, suficientes para una VPS de
pruebas pero NO para producción.

Para endurecerlos, crea un vault con estas dos claves:

```bash
# 1. Crear el fichero cifrado (te pide una passphrase nueva)
ansible-vault create inventory/group_vars/chirpstack_servers/vault.yml
```

Contenido del vault:

```yaml
vault_postgresql_db_password: "<openssl rand -base64 24>"
vault_chirpstack_api_secret:  "<openssl rand -base64 32>"
```

```bash
# 2. Guardar la passphrase y reactivar su uso automático
echo 'tu-passphrase' > ~/.ansible_vault_pass && chmod 600 ~/.ansible_vault_pass
# 3. Descomentar `vault_password_file` en ansible.cfg
```

`vars.yml` y el rol postgresql ya referencian esas claves con fallback, así que
en cuanto existan se usan solas (sin tocar nada más).

⚠ Cambiar `vault_chirpstack_api_secret` con ChirpStack ya en marcha invalida las
sesiones y **las API keys existentes** (habría que regenerar
`/etc/chirpstack/.ansible_api_key`). Hazlo **antes** de crear la API key de admin.
Cambiar la password de Postgres sí es seguro: el mismo run actualiza el usuario
y `chirpstack.toml` a la vez.

## Errores frecuentes

- **`REMOTE HOST IDENTIFICATION HAS CHANGED`** tras reinstalar el VPS:
  `ssh-keygen -R <IP>` en la máquina de control y reintenta.
- **`Failed to update apt cache ... Mirror sync in progress`**: el mirror del
  proveedor está sincronizando; espera unos minutos y reintenta.
- **El navegador del móvil "rechaza la conexión"**: está forzando HTTPS;
  escribe `http://` explícito (no hay TLS configurado aún).
