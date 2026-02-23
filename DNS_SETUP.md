# DNS setup for portainer.pixelia.cloud

ACME validation is failing with **NXDOMAIN** because `portainer.pixelia.cloud` has no DNS records.

## Steps

1. Log in to your DNS provider (where `pixelia.cloud` is managed).
2. Add an **A record**:
   - **Name**: `portainer` (or full `portainer.pixelia.cloud` if your provider requires it)
   - **Value**: Your server's public IP (same as `n8n-devtallez.pixelia.cloud`)
   - **TTL**: 300 or Auto
3. (Optional) Add an **AAAA record** if you use IPv6.
4. Save changes.
5. Wait for propagation (usually a few minutes). Verify with:
   ```bash
   dig +short portainer.pixelia.cloud A
   ```
   You should see your server's IP.

After DNS resolves, Traefik will be able to obtain the certificate (after the rate limit resets).
