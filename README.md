# Install PAL Next Password on Synology

This guide installs PAL Next Password from its public GitHub Container Registry image. It does not require SSH or command-line access. The application source repository remains private.

## Requirements

- Synology DSM 7.2 or newer with Container Manager installed
- an Intel/AMD 64-bit or ARM64 NAS supported by Container Manager
- a trusted HTTPS hostname for the NAS
- the PAL Next Password iPhone app

## Tailscale is optional

Tailscale is not required. PAL Next Password only requires a trusted HTTPS address that the iPhone can reach. You can use:

- Synology's built-in reverse proxy with your own domain and trusted certificate;
- Tailscale or another private VPN;
- a trusted HTTPS reverse proxy elsewhere on your network.

The steps below use Synology's built-in reverse proxy. Do not connect the iPhone app directly to an HTTP address or continue past a certificate warning.

The public image is:

```text
ghcr.io/pal-paul/pal-next-password-server:latest
```

The image contains the compiled server only. It contains no passwords, setup tokens, private keys, or preconfigured database.

## 1. Prepare three values

| Setting | Example | Requirement |
| --- | --- | --- |
| Server ID | `home-nas-01` | Keep it unchanged for this installation. |
| Public URL | `https://passwords.example.com` | Must be trusted HTTPS and reachable from the iPhone. |
| Setup token | A random string of at least 32 characters | Generate it in a password manager and use it only for first setup. |

Do not use an online token generator. Keep the setup token temporarily because it is needed to open the one-time setup page.

## 2. Create the Container Manager project

1. Open **Package Center** and install **Container Manager**.
2. Download [compose.yaml](compose.yaml).
3. Open **Container Manager > Project** and select **Create**.
4. Set the project name to `pal-next-password`.
5. Select or create a project folder such as `docker/pal-next-password`.
6. Choose **Create docker-compose.yml** as the source and paste the downloaded file into the editor.
7. Replace `CHANGE_ME_STABLE_SERVER_ID`, `https://passwords.example.com`, and `CHANGE_ME_RANDOM_SETUP_TOKEN` with the values prepared above.
8. Select **Next**, confirm the settings, and select **Done**.

Container Manager downloads the correct Intel/AMD or ARM64 image automatically. The project is ready when the `pal-next-password` container reports **Healthy**.

## 3. Configure trusted HTTPS

The container listens only on NAS loopback port `18080`. Do not expose that HTTP port directly to the internet.

1. Open **Control Panel > Login Portal > Advanced > Reverse Proxy**.
2. Create a rule whose source is HTTPS with the hostname from `PAL_PUBLIC_URL` and port `443`.
3. Set the destination protocol to HTTP, hostname to `localhost`, and port to `18080`.
4. Open **Control Panel > Security > Certificate** and assign a trusted certificate to that hostname.
5. Allow HTTPS port `443` through the DSM firewall only from networks that should reach the vault.

A private VPN address is preferable to public internet exposure. The certificate must be trusted by iOS; certificate-warning pages will be rejected by the app.

Open `https://passwords.example.com/readyz` on the iPhone. It should return `{"status":"ready"}` without a certificate warning.

## 4. Connect the first iPhone

1. Open `https://passwords.example.com/setup?token=YOUR_SETUP_TOKEN` in Safari, substituting your hostname and token.
2. Open PAL Next Password and create and unlock a local vault.
3. Open **Devices > Connect server > Scan setup code**.
4. Scan the QR code displayed in Safari and select **Connect**.
5. Add a test login and run **Sync**.

The setup page returns 404 after the first device registers. This is intentional. Additional devices must be approved from an already enrolled device.

## 5. Disable first-device setup

1. Open the project in Container Manager and edit its Compose configuration.
2. Set `PAL_BOOTSTRAP_TOKEN` to an empty value: `PAL_BOOTSTRAP_TOKEN: ""`.
3. Apply or rebuild the project.
4. Confirm the container returns to **Healthy** and the iPhone still synchronizes.

Do not change `PAL_SERVER_ID`, `PAL_PUBLIC_URL`, or the `pal-next-password-data` volume name.

## Upgrade and backup

Back up the `pal-next-password-data` volume before upgrading. In **Container Manager > Registry**, download the newer image and rebuild or recreate the project. Never delete the named volume during an upgrade.

For predictable production upgrades, replace `latest` with a published version tag. Encrypt and access-control every backup even though vault records are encrypted.

## Troubleshooting

- **Image download fails:** confirm the NAS can reach `ghcr.io` over HTTPS.
- **Container repeatedly stops:** check the project log and verify all three placeholder values were replaced.
- **`/readyz` works locally but not on iPhone:** verify the reverse proxy, certificate, firewall, DNS, and VPN.
- **Setup page returns 404:** verify the token; if one device already connected, use approved-device enrollment.
- **Permission error for `/data`:** keep the named volume from the supplied Compose file.
