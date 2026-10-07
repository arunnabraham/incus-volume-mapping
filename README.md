# Incus Bind Mount with Host and Container User Ownership

This guide shows how to bind-mount a host directory into an Incus container so that:

- the directory is owned by one user on the **host**
- the same directory appears owned by a different user inside the **container**

Example setup:

```text
Host user:      devuser   UID 1000 / GID 1000
Container user: app       UID 1001 / GID 1001

Host path:      /home/devuser/projects/demo
Container path: /home/app/dev-work
Container name: php-web
```

The goal is:

```text
Host      -> devuser:devuser
Container -> app:app
```

## 1. Check the host user's UID and GID

Run on the host:

```bash
id devuser
```

Example output:

```text
uid=1000(devuser) gid=1000(devuser) groups=...
```

Record the UID and GID.

## 2. Check the container user's UID and GID

Run:

```bash
incus exec php-web -- id app
```

Example output:

```text
uid=1001(app) gid=1001(app) groups=...
```

Record the UID and GID.

## 3. Make sure the host directory belongs to the host user

```bash
sudo chown -R devuser:devuser /home/devuser/projects/demo
```

Verify:

```bash
ls -ldn /home/devuser/projects/demo
```

Example:

```text
1000 1000
```

## 4. Add the bind mount

Do **not** use `shift=true` when using this explicit `raw.idmap` setup.

```bash
incus config device add php-web dev-work disk \
  source=/home/devuser/projects/demo \
  path=/home/app/dev-work
```

The resulting device configuration should look like:

```yaml
dev-work:
  path: /home/app/dev-work
  source: /home/devuser/projects/demo
  type: disk
```

## 5. Add the explicit UID/GID mapping

For this example:

```text
Host UID/GID      1000:1000
Container UID/GID 1001:1001
```

Configure Incus:

```bash
incus config set php-web raw.idmap "uid 1000 1001
gid 1000 1001"
```

This means:

```text
Host UID 1000 -> Container UID 1001
Host GID 1000 -> Container GID 1001
```

## 6. Restart the container

```bash
incus restart php-web
```

## 7. Verify ownership on both sides

On the host:

```bash
ls -ld /home/devuser/projects/demo
```

Expected ownership:

```text
devuser devuser
```

Inside the container:

```bash
incus exec php-web -- ls -ld /home/app/dev-work
```

Expected ownership:

```text
app app
```

You can also verify the numeric IDs:

```bash
incus exec php-web -- ls -ldn /home/app/dev-work
```

Expected:

```text
1001 1001
```

## 8. Test write access as the container user

```bash
incus exec php-web -- su - app -c 'touch /home/app/dev-work/test-file'
```

Then check the file on the host:

```bash
ls -l /home/devuser/projects/demo/test-file
```

It should appear owned by:

```text
devuser devuser
```

Inside the container, the same file should appear owned by:

```text
app app
```

## 9. Check the current raw ID mapping

```bash
incus config get php-web raw.idmap
```

Example output:

```text
uid 1000 1001
gid 1000 1001
```

## 10. Generic template

Replace the values below with your own host/container names and IDs.

```bash
# Host user information
id HOST_USER

# Container user information
incus exec CONTAINER_NAME -- id CONTAINER_USER

# Add bind mount
incus config device add CONTAINER_NAME DEVICE_NAME disk \
  source=/host/path \
  path=/container/path

# Map host UID/GID to container UID/GID
incus config set CONTAINER_NAME raw.idmap "uid HOST_UID CONTAINER_UID
gid HOST_GID CONTAINER_GID"

# Restart
incus restart CONTAINER_NAME
```

Example mapping:

```text
HOST_USER       = devuser
HOST_UID        = 1000
HOST_GID        = 1000
CONTAINER_USER  = app
CONTAINER_UID   = 1001
CONTAINER_GID   = 1001
CONTAINER_NAME  = php-web
DEVICE_NAME     = dev-work
```

## Troubleshooting

### Directory appears as `nobody:nogroup`

Check the active mapping:

```bash
incus config get php-web raw.idmap
incus config get php-web volatile.idmap.current
```

Also verify that the host directory still has the expected numeric ownership:

```bash
ls -ldn /home/devuser/projects/demo
```

### Directory appears as another user inside the container

Compare numeric IDs instead of usernames:

```bash
id devuser
incus exec php-web -- id app
ls -ldn /home/devuser/projects/demo
incus exec php-web -- ls -ldn /home/app/dev-work
```

Usernames do not need to match between host and container; the UID/GID mapping determines ownership.

### `shift=true` is already configured

If using explicit `raw.idmap`, remove `shift` from the device:

```bash
incus config device unset php-web dev-work shift
```

Then restart the container:

```bash
incus restart php-web
```

## Final result

```text
Host filesystem
/home/devuser/projects/demo
owned by devuser:devuser (1000:1000)
             |
             | Incus raw.idmap
             v
Container
/home/app/dev-work
owned by app:app (1001:1001)
```

This lets both users work with the same files while preserving appropriate ownership on each side.
