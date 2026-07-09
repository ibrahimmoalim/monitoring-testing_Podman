
The `podman-restart` service specifically filters for `restart-policy=always`. When you use `podman-compose`, it translates `unless-stopped` into a format that systemd misses. Furthermore, if you are running this as a normal user without **lingering** enabled, systemd doesn't even attempt to run it until you log back in.

### here's how to automate restart

Since there already is a `compose.monitoring.yml` file, you can generate a native systemd service directly from it using a tool called `podman-compose-systemd` or by wrapping the compose command in a standard systemd service file.

Here is how to wrap the exact compose file into a systemd service that *actually* works on boot:

1. **Create a user systemd directory:**
```bash
mkdir -p ~/.config/systemd/user/
```


2. **Create the service file:**
Create a file named `~/.config/systemd/user/monitoring-containers.service` and paste this inside:
```ini
[Unit]
Description=Podman Compose Monitoring Stack
After=local-fs.target network-online.target

[Service]
Type=oneshot
RemainAfterExit=yes
# the '%h' translates to /home/<current-user>
# use full path /home/<user>/... to be specific
WorkingDirectory=/home/ibrahim/podman-RHEL-monitoring
ExecStart=/usr/bin/podman-compose -f compose.monitoring.yml up -d
ExecStop=/usr/bin/podman-compose -f compose.monitoring.yml down

[Install]
WantedBy=default.target

```


3. **Enable and Start it:**
```bash
systemctl --user daemon-reload
systemctl --user enable --now monitoring-containers.service
systemctl --user status monitoring-containers.service
```

Now, systemd explicitly tracks the entire compose stack. It will run `up -d` on boot and cleanly run `down` if the server gracefully shuts down.

---

### The Absolute Must-Do Step: Enable Lingering

Whether you use the method above or try to fix the restart policy, **the containers will still fail to boot on a headless server without this command:**

```bash
sudo loginctl enable-linger $USER
```
Or type user's name directly
```bash
sudo loginctl enable-linger ibrahim
```

#### How to Double-Check the Work

Systemd stores lingering profiles inside a specific system folder. You can instantly verify if it worked by running this command:
```bash
ls /var/lib/systemd/linger/
```

If you see a file named `ibrahim` inside that directory, you nailed it. the containers are now officially bulletproof and will spin up on system boot before anyone even touches a keyboard.

**Why this is mandatory:** By default, Debian and RHEL kill all user processes the moment a user logs out. `enable-linger` tells systemd: *"Spawn this user's helper process at system boot, and never kill their background apps."* This is the secret sauce for rootless enterprise automation.
