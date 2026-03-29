# nfs_mounts

Mount NFS shares on target hosts.

## Variables

- All mounts should be set through the variables `nfs_mounts`
- Format:

```yaml
nfs_mounts:
  - src: NFS server and export path (e.g., '10.0.40.3:/export/data')
    path: Local mount point on the host (e.g., '/var/lib/k3s/volumes/slowdata')
    opts: Mount options (e.g., 'rw,sync,hard,intr')
    owner: root # optional; directory owner, default is root
    group: root # optional; directory group, default is root
    mode: '0755' # optional; directory permissions, default is 0755
```

- Vars defined in `inventories/production/group_vars/all/nfs.yml`
- **Group vars overwrite all vars**
- Optional per-mount parameters (if desired):
