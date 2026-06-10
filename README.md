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

# 3. Passphrase del vault (una sola vez; pídela al responsable del proyecto)
echo 'la-passphrase-del-vault' > ~/.ansible_vault_pass
chmod 600 ~/.ansible_vault_pass
```

> Sin `~/.ansible_vault_pass` **ningún playbook arranca** (el inventario carga
> `vault.yml`, que está cifrado). Alternativa: comenta `vault_password_file`
> en `ansible.cfg` y ejecuta con `--ask-vault-pass`.

## Requisitos del servidor destino

- Ubuntu (Server o Desktop, 64-bit; probado en 24.04 y 26.04).
- Acceso SSH como `root` (la contraseña se pide al arrancar el playbook).
- La IP se configura en `inventory/hosts.yml` → `ansible_host`.

## Uso

```bash
# Probar conectividad
ansible-playbook playbooks/ping.yml

# SOLO el núcleo ChirpStack + nginx (sin dashboard, sin firewall)
ansible-playbook playbooks/chirpstack.yml

# Stack COMPLETO (incluye registro de gateways, dashboard, firewall)
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
2. **Deploy key del dashboard**: el rol `dashboard` la genera y te la muestra
   para añadirla en GitHub (Settings → Deploy keys del repo del dashboard).

## Secretos

Los secretos viven cifrados en `inventory/group_vars/chirpstack_servers/vault.yml`:

```bash
ansible-vault edit inventory/group_vars/chirpstack_servers/vault.yml
```

Claves soportadas (las dos últimas son opcionales: si no existen se usa un
valor temporal INSEGURO, válido solo para pruebas):

| Clave | Usada por |
|---|---|
| `vault_dashboard_pg_password` | BD del dashboard |
| `vault_dashboard_gateway_id` | dashboard (.env) |
| `vault_dashboard_chirpstack_jwt` | dashboard → API ChirpStack |
| `vault_postgresql_db_password` | BD de ChirpStack (genera con `openssl rand -base64 24`) |
| `vault_chirpstack_api_secret` | firma de JWT/API keys de ChirpStack (`openssl rand -base64 32`) |

⚠ Si cambias `vault_chirpstack_api_secret` con ChirpStack ya en marcha, se
invalidan las sesiones y **las API keys existentes** (habría que regenerar
`/etc/chirpstack/.ansible_api_key`). Si cambias la password de Postgres, el
mismo run actualiza usuario y `chirpstack.toml` a la vez — sin acción manual.

## Errores frecuentes

- **`REMOTE HOST IDENTIFICATION HAS CHANGED`** tras reinstalar el VPS:
  `ssh-keygen -R <IP>` en la máquina de control y reintenta.
- **`Failed to update apt cache ... Mirror sync in progress`**: el mirror del
  proveedor está sincronizando; espera unos minutos y reintenta.
- **El navegador del móvil "rechaza la conexión"**: está forzando HTTPS;
  escribe `http://` explícito (no hay TLS configurado aún).
