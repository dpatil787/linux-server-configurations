# NFS Server Configuration

![NFS](https://img.shields.io/badge/Linux-NFS_Server-blue?style=for-the-badge&logo=linux)
![Status](https://img.shields.io/badge/Status-Completed-success?style=for-the-badge)

---

## What is NFS?

NFS (Network File System) is a protocol that allows one system to share its files and directories with other systems over a network. A remote directory on another machine behaves like it is part of your local system. So instead of copying files between servers, multiple systems can directly access the same shared storage.

- **Port:** 2049
- **Protocol Type:** Client-Server Model
- **Use Case:** Linux-to-Linux file sharing *(Samba/CIFS is used when Windows clients need to access Linux shares)*

---

## Why NFS is Used in Production

**a) Shared Application Files**
When multiple web servers run the same application, they need to access shared files like user uploads, configuration files, or cached data. NFS provides a central location where all servers can read and write these files.

**b) Centralized Storage**
NFS allows one server to host files that many clients need. This reduces disk usage, simplifies backups, and ensures everyone accesses the same version of files. Instead of duplicating data everywhere, NFS provides one central location.

**c) Home Directories**
In environments with many Linux servers, user home directories are often stored on an NFS server. Users can log into any server and find their files exactly as they left them.

---

## How NFS Works

NFS works on a **Client-Server Model:**

| Component | Role |
|-----------|------|
| NFS Server | Shares a directory over the network and controls access (who can connect, read/write permissions) |
| NFS Client | Connects to the server, mounts the shared directory, and uses it like a local folder |

---

## Key Components

| Component | Description |
|-----------|-------------|
| `/etc/exports` | Defines which directories are shared and who can access them |
| `mount` | Used by client to attach the remote directory to a local path |
| NFS Service | Handles file sharing between server and client |
| Permissions | Server controls who can access and whether access is read-only (ro) or read-write (rw) |
| Stateless Nature | Server does not track client state — each request is independent |

---

## Key Configuration Files

**a) `/etc/exports` (Server Side)**
Defines which directories are shared and who can access them. Each line specifies a directory to export, the client or network that can access it, and the permissions.

**b) `/etc/fstab` (Client Side)**
Defines which NFS shares should be mounted automatically at boot. Each entry includes the server address, export path, mount point, file system type, and options.

**c) `/etc/idmapd.conf`**
Handles mapping of user and group IDs between server and client.

---

## Important Export Options

| Option | Description |
|--------|-------------|
| `ro` | Read-only — clients can read but not write. Used for static content |
| `rw` | Read-write — clients can read and write. Used for application data, home directories |
| `sync` | Writes confirmed to disk before server responds. Slower but safe. Use for critical data |
| `async` | Writes confirmed before fully written to disk. Faster but risks data loss on crash |
| `root_squash` | Maps root user on client to nobody on server. Prevents root privileges. **Default and recommended** |
| `no_root_squash` | Allows root on client to have root access on server. **Security risk — avoid** |
| `no_subtree_check` | Disables subtree checking for better performance. Recommended for most exports |
| `no_all_squash` | Regular users keep their own UID when accessing files |
| `all_squash` | All users mapped to nobody — maximum restriction |

---

## Advantages

- Easy to set up and configure
- Centralized data management
- Reduces data duplication across servers

---

## Limitations

- **Performance:** Depends on network — file access becomes slow if network is slow
- **Single Point of Failure:** If NFS server goes down, all clients lose access. Logs stop writing and applications depending on NFS will fail
- **Security:** By default, NFS is based on IP trust with no strong authentication and no encryption

---

## Modern Alternatives to NFS

| Service | Description |
|---------|-------------|
| AWS EFS | AWS managed NFS |
| S3 | Object storage |
| EBS | Block storage |

---

## NFS in Kubernetes

NFS is commonly used as a **Persistent Volume** in Kubernetes:

- An NFS export is defined as a `PersistentVolume`
- Pods claim it with a `PersistentVolumeClaim`
- Provides simple persistent storage for stateful applications without complex storage solutions
- The Kubernetes control plane does **not** manage NFS — it simply mounts the NFS share on the node where the Pod runs
- The NFS server must be maintained separately

---

## Hands-On: NFS Server Setup

### Step 1 — Configure NFS Server

```bash
# Set hostname
hostnamectl set-hostname NFSServer
hostname

# Install NFS
dnf install nfs-utils -y

# Start and enable NFS service
systemctl start nfs-server
systemctl enable nfs-server
systemctl status nfs-server
```

![Service Running](images/1-Service.png)

---

### Step 2 — Create and Configure Share Directory

```bash
# Create share directory
mkdir -p /mnt/nfs_shares/docs
ls -ld /mnt/nfs_shares/docs
```

We can see owner and group is root. We need to change owner to `nobody` because it is a special system user with minimal privileges, mapped via `root_squash` to prevent root-level access from client systems.

```bash
# Change owner to nobody
chown -R nobody: /mnt/nfs_shares/docs
ls -ld /mnt/nfs_shares/docs
# Output: owner and group is now nobody nobody

# Check nobody user details
cat /etc/passwd | grep nobody
# Output: nobody:x:65534:65534:Kernel Overflow User:/:/sbin/nologin
# nobody = special system user with UID 65534, no login shell, minimal privileges

# Give full permissions for read-write access
chmod -R 777 /mnt/nfs_shares/docs
ls -ld /mnt/nfs_shares/docs
# Output: drwxrwxrwx — full access permissions

# Restart NFS
systemctl restart nfs-server
```

---

### Step 3 — Configure /etc/exports

```bash
vi /etc/exports
```

Add the following configuration:

```
# Allow specific IPs
/mnt/nfs_shares/docs 192.168.253.129(rw,sync,no_all_squash,root_squash) 192.168.253.130(rw,sync,no_all_squash,root_squash)

# OR allow entire network
/mnt/nfs_shares/docs 192.168.253.0/24(rw,sync,no_all_squash,root_squash)
```

**Options explained:**
- `rw` — read-write access
- `sync` — server confirms write is physically on disk before responding. Can be slow but safe
- `no_all_squash` — regular users on client keep their own UID. If client user has UID 1001, actions on server are performed as UID 1001
- `root_squash` — root user on client is mapped to nobody on server (UID 65534). Prevents client root from having full server control

```bash
# Restart NFS and apply exports
systemctl restart nfs-server.service
exportfs -arv
# Output:
# exporting 192.168.253.129:/mnt/nfs_shares/docs
# exporting 192.168.253.130:/mnt/nfs_shares/docs
```

> `exportfs -arv` re-reads `/etc/exports` and applies changes immediately without restarting the NFS service

![Export Configuration](images/2-export.png)

![Exportfs Output](images/3-exportfs.png)

---

### Step 4 — Configure Firewall

```bash
# Add NFS service
firewall-cmd --permanent --add-service=nfs --zone=public

# Add rpcbind — redirects client to port where NFS is running
firewall-cmd --permanent --add-service=rpc-bind --zone=public

# Add mountd — checks client permissions when mounting
firewall-cmd --permanent --add-service=mountd --zone=public

firewall-cmd --reload
firewall-cmd --list-all
```

> Note: Even if you add only NFSv4, rpcbind and mountd will automatically get added in firewall

---

### Step 5 — Configure NFS Client

```bash
# Set hostname
hostnamectl set-hostname NFSCLIENT
hostname
reboot

# Install NFS utils
dnf install nfs-utils -y

# Verify exports from server
showmount -e 192.168.253.128
# Output:
# Export list for 192.168.253.128:
# /mnt/nfs_shares/docs 192.168.253.130,192.168.253.129

# Create mount point
mkdir -p /mnt/client_share
cd /mnt

# Mount the share
mount -t nfs 192.168.253.128:/mnt/nfs_shares/docs /mnt/client_share

# Verify mount
df -hT
# Output: 192.168.253.128:/mnt/nfs_shares/docs nfs4 17G 2.0G 15G 12% /mnt/client_share
```

![Mount Verification](images/4-mounts.png)

---

### Step 6 — Test File Access

**On NFS Server — create a file:**
```bash
cd /mnt/nfs_shares/docs
echo "Dnyanesh is learning DevOps and Linux Administration" > servernfs.txt
ls -lrth
```

**On NFS Client — verify access:**
```bash
cd /mnt/client_share
ls -lrth
```

Expected output shows ownership behavior:
```
-rw-r--r--. 1 root    root    30  servernfs.txt   # Created on server — root owner, not affected by NFS options
-rwxrwxrwx. 1 nobody  nobody  61  nfsclient.txt   # Created by client root — mapped to nobody (root_squash active)
-rw-r--r--. 1 dnyanesh dnyanesh 26 usernfs.txt    # Created by user — actual username shown (no_all_squash active)
```

---

## root_squash Behavior Demonstration

### With root_squash (Default — Recommended)

Client root → mapped to nobody on server

![Root Squash Behavior](images/5-rootsquash.png)

---

### With no_root_squash (Security Risk)

```bash
# On NFS Server — disable root_squash
vi /etc/exports
# Change to: no_root_squash
systemctl restart nfs-server.service

# On Client — create file as root
touch norootsquash.txt
ls -lrth
# Output: -rw-r--r--. 1 root root 0 norootsquash.txt
# Owner is root — not nobody. Security risk!
```

![No Root Squash Behavior](images/6-norootsquash.png)

> **Not recommended** — root user has full privileges and can do anything on server. Re-enable root_squash after testing.

---

### With all_squash

```bash
# On NFS Server
vi /etc/exports
# Change to: all_squash,root_squash
systemctl restart nfs-server.service

# On Client — create file as regular user
vi allsquash.txt
ls -lrth
# Output: owner is nobody — all users mapped to nobody
```

![All Squash Behavior](images/7-allsquash.png)

---

## Permanent Mount via /etc/fstab

To mount NFS share automatically after reboot:

```bash
vi /etc/fstab
```

Add this line at the end:
```
192.168.253.128:/mnt/nfs_shares/docs /mnt/client_share nfs defaults,_netdev 0 0
```

> `_netdev` ensures the system waits for network to be available before attempting the mount

---

## Unmount Share

```bash
umount /mnt/client_share
df -hT  # Verify unmounted
```

---

## Interview Questions & Answers

**Q: What is NFS and why would you use it?**

NFS is a protocol for sharing files over a network. I would use it when multiple servers need access to the same files, such as shared home directories, web application assets, or backup storage. It is simple to configure and works well for environments that do not need complex distributed file system features.

---

**Q: How do you export a directory on an NFS server?**

I add an entry in `/etc/exports` specifying the directory, allowed clients, and options. Then I run `exportfs -a` to apply the changes and restart the nfs-server service.

---

**Q: How do you mount an NFS share permanently on a client?**

I add an entry in `/etc/fstab` with the server IP, export path, mount point, filesystem type nfs, and options like `defaults,_netdev`. The `_netdev` option ensures the system waits for the network before attempting the mount.

---

**Q: What is root_squash and why is it important?**

`root_squash` maps the root user on the client to the nobody user on the server. This prevents a client with root access from having root privileges on the server. It is an important security feature that limits the impact of a compromised client.

---

**Q: What is the difference between sync and async?**

`sync` confirms writes to disk before responding, ensuring data integrity. `async` is faster but risks data loss if the server crashes. `sync` is used for critical data, `async` for performance-sensitive temporary data.

---

**Q: How do you check what directories an NFS server is sharing?**

I use the command `showmount -e server-ip`. This lists all exports available from that server.

---

**Q: What is a stale file handle error?**

It occurs when a client tries to access a file that was removed or changed on the server while the client still had it mounted. It is fixed by unmounting and remounting the share on the client.

---

## Troubleshooting

| Issue | Cause | Fix |
|-------|-------|-----|
| Permission Denied | Client IP not in `/etc/exports` or options too restrictive | Check `/etc/exports` and run `exportfs -a` |
| Stale File Handle | Directory on server removed/changed while client had it mounted | Unmount and remount the share |
| Connection Refused | NFS service not running or firewall blocking ports | Check service status and firewall rules |
| Slow Performance | Network latency, sync option, or slow disk I/O on server | Consider using async or improving network |

---

*Document prepared as part of DevOps Home Lab — Linux Server Configuration Series*

