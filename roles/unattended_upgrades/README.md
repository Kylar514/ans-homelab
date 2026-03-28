# Unattended Upgrades Role

## Configures automatic package updates on Debian/Ubuntu systems.

## Variables

| Variable              | Default | Description                                      |
| --------------------- | ------- | ------------------------------------------------ |
| apt_update_period     | 1       | How often to update package lists (days)         |
| apt_download_upgrades | 1       | How often to download upgradable packages (days) |
| autoclean_interval    | 7       | How often to autoclean (days)                    |
| unattended_upgrade    | 1       | Enable unattended-upgrades (1 = yes, 0 = no)     |

You can override these in `group_vars` or `host_vars` as needed.
