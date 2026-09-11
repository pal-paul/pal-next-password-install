# Install PAL Next Password on Synology

This guide installs PAL Next Password from its public GitHub Container Registry image through Container Manager. The container installation does not require SSH. Option A requires one short SSH session only to enable Tailscale Serve; Option B requires no SSH.

## Requirements

- Synology DSM 7.2 or newer with Container Manager installed
- an Intel/AMD 64-bit or ARM64 NAS supported by Container Manager
- a trusted HTTPS hostname for the NAS
- the PAL Next Password iPhone app

## Choose how the iPhone connects

Tailscale is optional. Choose one connectivity option before creating the Container Manager project.

### Option A: Tailscale private access

Choose this option for private access without opening a router port or owning a domain. The NAS and iPhone must be signed into the same Tailscale network. Container installation uses the Synology interface, but enabling Tailscale Serve requires one short SSH session.

Use a public URL resembling:

```text
https://my-nas.example-tailnet.ts.net
```

### Option B: Synology reverse proxy without Tailscale

Choose this option when you have a domain name and trusted TLS certificate. The hostname must resolve to the NAS from every network where the iPhone will use synchronization. Remote access normally requires router and firewall configuration; LAN-only access can use local DNS.

Use a public URL resembling:

```text
https://passwords.example.com
```

Both options provide HTTPS in front of the container's local HTTP endpoint. Do not connect the iPhone app directly to HTTP or continue past a certificate warning.

The public image is:

```text
ghcr.io/pal-paul/pal-next-password-server:latest
```

The image contains the compiled server only. It contains no passwords, setup tokens, private keys, or preconfigured database.

## 1. Prepare three values

| Setting | Example | Requirement |
| --- | --- | --- |
| Server ID | `home-nas-01` | Keep it unchanged for this installation. |
| Public URL | Your Option A or Option B URL | Must be trusted HTTPS and reachable from the iPhone. |
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

### Option A: Tailscale private access

1. Install **Tailscale** from Synology Package Center and install the Tailscale app on the iPhone.
2. Sign both devices into the same tailnet.
3. Enable **MagicDNS** and **HTTPS Certificates** in the Tailscale admin console.
4. Confirm that `PAL_PUBLIC_URL` in the Compose configuration uses the NAS hostname ending in `.ts.net`.
5. Temporarily enable SSH under **Control Panel > Terminal & SNMP** and connect to the NAS.
6. Publish the loopback-only container through Tailscale Serve:

   ```sh
   sudo tailscale serve --bg http://127.0.0.1:18080
   sudo tailscale serve status
   ```

7. Disable SSH again. Keep Tailscale connected on the iPhone while using the server.

Do not enable Tailscale Funnel. Serve keeps the endpoint private to authenticated devices in the tailnet.

### Option B: Synology reverse proxy without Tailscale

1. Open **Control Panel > Login Portal > Advanced > Reverse Proxy**.
2. Create a rule whose source is HTTPS with the hostname from `PAL_PUBLIC_URL` and port `443`.
3. Set the destination protocol to HTTP, hostname to `localhost`, and port to `18080`.
4. Open **Control Panel > Security > Certificate** and assign a trusted certificate to that hostname.
5. Allow HTTPS port `443` through the DSM firewall only from networks that should reach the vault.
6. If access is required outside the home network, point the hostname to the router and forward only HTTPS port `443` to the NAS. Do not forward port `18080`.

Use a trusted certificate from a public certificate authority. Synology's built-in Let's Encrypt support is suitable when its validation requirements can be met.

### Verify either option

Open this address on the iPhone:

```text
YOUR_PUBLIC_URL/readyz
```

It should return `{"status":"ready"}` without a certificate warning.

## 4. Connect the first iPhone

1. Open the following URL in Safari, substituting the selected public URL and setup token:

   ```text
   YOUR_PUBLIC_URL/setup?token=YOUR_SETUP_TOKEN
   ```

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
- **`/readyz` works locally but not on iPhone:** verify the selected option's hostname, certificate, firewall, DNS, and VPN state.
- **Tailscale URL does not open:** verify both devices are in the same tailnet, MagicDNS and HTTPS are enabled, and `tailscale serve status` lists port `18080`.
- **Setup page returns 404:** verify the token; if one device already connected, use approved-device enrollment.
- **Permission error for `/data`:** keep the named volume from the supplied Compose file.
