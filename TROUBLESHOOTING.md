### 🧩 Post-Remediation Note: GUI Disabled

After running the OpenSCAP STIG remediation playbook, the default boot target was changed from `graphical.target` to `multi-user.target`, resulting in a missing GUI on reboot.i

I checked the current state via systemctl commands, which configmed multi-user.target: [No Gui](/screenshots/machine_noGui.png)
```bash
sudo systemctl get-default
```

To restore the graphical desktop environment: [With Gui](/screenshots/machine_withGui.png)

```bash
sudo systemctl set-default graphical.target
sudo reboot
```

