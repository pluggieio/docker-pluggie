# Pluggie Docker Edition


## Installation

Detailed step-by-step instructions are shown on your account page at
[https://my.pluggie.net](https://my.pluggie.net) under "Installation on Docker".
The short version:

1. Create a directory for Pluggie container data files (it is mounted on
   `/data` and holds the access key, certificates and client certificates;
   include it in your backups), for example:
   ```
   mkdir -p /YOUR_HOMEDIR/.pluggie
   ```
2. Save the Docker run script shown on the "Installation on Docker" panel of
   your account page as `docker-pluggie.sh`.
3. Edit the script and set your own values for:
   - `PLUGGIE_DIR` — directory for Pluggie data (required)
   - `PLUGGIE_HOST` — IP address to listen on (optional, default `127.0.0.1`)
   - `PLUGGIE_PORT` — port to listen on (optional, default `8678`)
4. Make the script executable and run it:
   ```
   chmod +x docker-pluggie.sh
   ./docker-pluggie.sh
   ```

### Docker Compose

```yaml
services:
  pluggie:
    image: ghcr.io/pluggieio/docker-pluggie:latest
    container_name: docker-pluggie
    restart: unless-stopped
    cap_drop: [ALL]
    cap_add: [NET_ADMIN, DAC_OVERRIDE, CHOWN, SETUID, SETGID]
    ports:
      - "127.0.0.1:8678:8099"
    environment:
      PLUGGIE_HOST: "127.0.0.1"
      PLUGGIE_PORT: "8678"
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - ./pluggie:/data
```

Update with `docker compose pull && docker compose up -d`; the data stays in
`./pluggie`.

**Keep `/data` mounted from a host directory (bind mount).** Without it the
data lives inside the container and is lost whenever the container is
recreated (update, `docker rm`, `docker compose up` after a pull); the
container logs a warning, the admin UI shows it and client certificates cannot
be turned on. With a named volume instead of a directory, `docker compose
down -v` and `docker volume prune -a` delete all data.


### One-Time Setup

1. Open the Pluggie admin interface in your browser:
   ```
   http://PLUGGIE_HOST:PLUGGIE_PORT
   ```
2. Set the Access Key (from [https://my.pluggie.net](https://my.pluggie.net))
   and other settings, then save
3. Check the container status in log output:
   ```
   docker logs -f docker-pluggie
   ```
4. Test the connection by opening your browser and navigating to your chosen
   generated domain name:
   ```
   https://generated-domain-name.pluggie.net
   ```
   or your custom domain if configured in the admin interface
   ([https://my.pluggie.net](https://my.pluggie.net))


## Using with Home Assistant

1. Set "Proxied Host" to your Home Assistant URL as seen from the container,
   for example `http://192.168.1.10:8123`
2. Let Home Assistant trust the Pluggie container as a reverse proxy:
   Settings -> System -> Network -> HTTP server, turn on "Trust X-Forwarded-For"
   and add the container IP address as Home Assistant sees it to "Trusted proxies"
   (the address is shown in the Home Assistant log message about an untrusted proxy).
   Older versions: `http:` block in `configuration.yaml`.

Step-by-step guide with troubleshooting: https://pluggie.io/blog/home-assistant-trusted-proxies#docker


## Configuration

### Required Configuration

```
Access Key: YOUR_ACCESS_KEY          # Access key from https://my.pluggie.net
```


### Advanced Configuration

```
Log Level: info                        # Optional: debug, info, warning, error; Default: info
Proxied Host: ""                       # Optional: Full URL of proxied host; Default: http://localhost:8080

# Basic Auth
Basic Authentication Username: ""      # Optional: Username for Basic Auth protection
Basic Authentication Password: ""      # Optional: Password for Basic Auth protection

# Let's Encrypt Configuration
ACME Root CA Certificate: ""           # Optional: Custom root CA certificate
ACME Server: ""                        # Optional: Custom ACME server URL
Key Type: ecdsa                        # Optional: Certificate key type (ecdsa, rsa)
Elliptic Curve: secp256r1              # Optional: Curve for ECDSA keys (secp256r1, secp384r1); Default: secp256r1
```

### Configuration Options Explained

**Basic**
- `Access Key`: Your unique access key that identifies and authorizes this Pluggie client.

**Logging**
- `Log Level`: Controls the detail level of logging. Use 'debug' for troubleshooting, 'info' for normal operation.

**Network**
- `Proxied Host`: URL of the device to connect to (e.g. https://myrouter.internal:8080). Note that `localhost` resolves inside the container; to reach a service on the Docker host use the host's LAN IP.

**Security**
- `Basic Authentication Username`: Optional username for Basic Auth protection. If set along with password, will enable Basic Auth.
- `Basic Authentication Password`: Optional password for Basic Auth protection. Must be set together with username.


**SSL/TLS Certificate**
- `ACME Root CA Certificate`: Custom certificate authority. Only needed if you use your own CA.
- `ACME Server`: Alternative ACME server URL. Use this if you want to use a different certificate provider.
- `Key Type`: Choose between ECDSA (faster, modern) or RSA (wider compatibility) certificates.
- `Elliptic Curve`: Security level for ECDSA certificates. secp256r1 is recommended for most users.


## Client certificates (mTLS)

Optional extra protection for your Pluggie address: only devices with a client certificate issued by this container can open it. Settings are in the admin UI under **Show Advanced Settings → Client Certificates (mTLS)** and take effect immediately (no Save needed).

**Modes**
- `Off`: client certificates are not requested (default).
- `Test`: certificates are requested, accepted and recorded ("Last seen"), nothing is blocked. Use it to check that your devices work.
- `Enforce`: requests without a valid certificate are rejected with an explanation page. Available only after at least one device has connected with its certificate in Test mode.

**Adding a device**
1. Enter a device name and click **Add device**. A `.p12` certificate file is downloaded and its password is shown once; it is not stored anywhere.
2. Install the file on the device:
   - Android (requires a screen lock — PIN, pattern or password — before it lets you install a certificate):
     1. Download the `.p12` file to the phone.
     2. Open Settings, search for "certificate", tap "Install a certificate" → "VPN & app user certificate", pick the file and enter the password shown when the certificate was issued (fallback: Security & privacy → More security settings → Encryption & credentials).
     3. Choose the certificate when Chrome or the Home Assistant app asks for it.
   - Home Assistant companion app on iOS (2026.9.1 or newer): the app asks for the certificate when the server requests it.
   - Desktop: import the file into the system keychain or your browser.
3. Open your Pluggie address on the device and check that "Last seen" is filled in, then switch to Enforce.

One certificate per device; **Revoke** blocks that device only. Certificates are valid for 5 years.

**Changed your Pluggie address?** No restart needed: usually within a minute the container reconnects, gets a certificate for the new address and keeps the mode and device certificates. Device certificates stay valid, nothing has to be reissued. Update the address in your apps and bookmarks; the browser or app may ask once more which certificate to use for the new address. The certificate authority keeps the previous address in its name; that is expected.

**Locked out?** The admin UI and your local network are never affected. Open the admin UI from your local network and switch the mode to Off.

**Locked out while away?** You can also turn client certificates off from your Pluggie account at [my.pluggie.net](https://my.pluggie.net); this requires two-factor authentication on the account. The Pluggie endpoint announces such a request through the tunnel, so the app switches the mode to Off within seconds and shows a notice in the admin UI; if the app is offline, the request is applied when it connects again (within 7 days).

**Backup:** the certificate authority (including its private key) is stored in `/data/mtls`, i.e. in the `PLUGGIE_DIR` directory (bind mount), and never leaves it.
- Include the `PLUGGIE_DIR` directory in your backups.
- If the data is lost, client certificates are off and the admin UI shows a notice: restore the directory or set up client certificates again.
- After restoring an older backup, revoke again every device certificate you revoked after that backup was made.
- "Delete all certificates" removes the authority together with all device certificates.


## Account recovery

If you lose access to your Pluggie account (password and two-factor authentication), contact Pluggie support. Support gives you a one-time code. Open the admin UI → **Show Advanced Settings → Account Recovery**, enter the code and click **Verify**. The app proves to Pluggie that it holds this tunnel's access key, and you get a link that lets you set a new password and turns off two-factor authentication on your account. The link is shown only once and is not stored or logged by the app. The code is valid for a limited time and can be used once; after 5 attempts within 15 minutes you have to wait.


## Security

**Do not expose the admin UI port (default 8678) directly to the public Internet without authentication.** The admin UI exposes endpoints that return your Pluggie `Access Key` in plaintext. If the port is reachable from the Internet without an authenticating reverse proxy, basic auth, or firewall rule restricting source IPs, the credential can be read by anyone who finds the host. Bind the port to localhost (the default `PLUGGIE_HOST=127.0.0.1` in the installation script already does this), place a reverse proxy with authentication in front of it, or restrict access via firewall.


## Support

- Got questions? Visit our website at [https://pluggie.io](https://pluggie.io)
- Need help? Contact our support team through support@pluggie.net
