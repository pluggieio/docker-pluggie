# Pluggie Docker Edition


## Installation

Detailed step-by-step instructions are shown on your account page at
[https://my.pluggie.net](https://my.pluggie.net) under "Installation on Docker".
The short version:

1. Create a directory for Pluggie container data files, for example:
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


## Security

**Do not expose the admin UI port (default 8678) directly to the public Internet without authentication.** The admin UI exposes endpoints that return your Pluggie `Access Key` in plaintext. If the port is reachable from the Internet without an authenticating reverse proxy, basic auth, or firewall rule restricting source IPs, the credential can be read by anyone who finds the host. Bind the port to localhost (the default `PLUGGIE_HOST=127.0.0.1` in the installation script already does this), place a reverse proxy with authentication in front of it, or restrict access via firewall.


## Support

- Got questions? Visit our website at [https://pluggie.io](https://pluggie.io)
- Need help? Contact our support team through support@pluggie.net
