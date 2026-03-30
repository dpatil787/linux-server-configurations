## NFS Server

NFS stands for Network File System.

NFS (Network File System) is a protocol that allows one system to share its files and directories with other systems over a network. NFS makes a remote directory on another machine behave as if it is part of your local system. Instead of copying files between servers, multiple systems can directly access the same shared storage.

- **Port number of NFS:** 2049
- NFS is used for Linux-to-Linux file sharing.
- **Samba / CIFS** is used when Windows clients need to access Linux shares.

---

## Why NFS is Used

In production environments, multiple servers often need access to the same data. For example:

### a) Shared Application Data
When multiple web servers run the same application, they often need to access shared files like user uploads, configuration files, or cached data. NFS provides a central location where all servers can read and write these files.

### b) Centralized Storage
NFS allows one server to host files that many clients need. This reduces disk usage, simplifies backups, and ensures everyone accesses the same version of files. Instead of duplicating data everywhere, NFS provides one central location.

### c) Shared Home Directories
In environments with many Linux servers, user home directories are often stored on an NFS server. Users can log into any server and find their files exactly as they left them.

---

## How NFS Works

NFS works on a **Client-Server Model**.

| Component | Role |
|-----------|------|
| **NFS Server** | Shares a directory over the network and controls access (who can connect, read-write permissions) |
| **NFS Client** | Connects to the server, mounts the shared directory, and uses it like a local folder |

---

## Key Components of NFS

### a) /etc/exports
This file defines which directories are shared and who can access them. The server decides which directories to export, which clients can access them, and what permissions those clients have.

### b) mount
This command is used by the client to attach a remote directory. The client connects and attaches the remote directory to a local path. Once mounting is done, remote files behave like local files. Normal commands like `ls`, `cat`, etc. work on them.

### c) NFS Service
Handles file sharing between server and clients.

### d) Permissions
Permissions are important. The server controls who can access and which users have read-only (`ro`) or read-write (`rw`) access. If permissions are wrong, clients may not be able to access files, write logs, or applications may fail.

### e) Stateless Nature
The server does not track client state. It does not remember which file a client opened or what the client did before. Each request is independent.

---

## Key Configuration Files

### a) /etc/exports (Server Side)
This file defines which directories are shared and who can access them. Each line specifies a directory to export, the client or network that can access it, and the options.

### b) /etc/fstab (Client Side)
In this file we can define which NFS shares should be mounted automatically at boot. Each entry includes the server address, export path, mount point, file system type, and options.

### c) /etc/idmapd.conf
This file handles mapping of user and group IDs between server and client.

---

## Important Export Options

| Option | Description |
|--------|-------------|
| **ro (read-only)** | Clients can read files but cannot write or modify them. Used for static content. |
| **rw (read-write)** | Clients can read and write files. The common setting for application data and home directories. |
| **sync** | Writes are confirmed to disk before the server responds. Can slow performance. Mainly used for critical data. |
| **async** | Writes are confirmed before they are fully written to disk. Faster but risks data loss if server crashes. Use for performance-sensitive temporary data. |
| **root_squash** | Maps the root user on the client to the `nobody` user on the server. Prevents client root from having root access on the server. This is the default and recommended for security. |
| **no_root_squash** | Allows root on the client to have root access on the server. Rarely used due to security risks. |
| **no_subtree_check** | Disables subtree checking for better performance. Recommended for most exports. |

---

## Advantages of Using NFS

- Easy to Set Up
- Centralized Data Management
- Reduces Data Duplication

---

## Limitations of NFS

- **Performance depends on network:** File access becomes slow, application response time increases, and system may appear laggy.
**Single point of failure:** If NFS goes down, all clients lose access. Logs stop writing and applications depending on NFS will fail.
- **Security:** By default, NFS is not much secure, as it is based on IP trust, with no strong authentication and data is not encrypted.
---

## NFS is Mostly Used In

- Small environments
-Simple shared storage
---

## Modern Alternatives to NFS

| Service | Purpose |
|---------|---------|
| **AWS EFS** | Managed NFS service by AWS |
| **S3** | Object Storage |
| **EBS** | Block Storage |

---

## NFS in Kubernetes

NFS is commonly used as a persistent volume in Kubernetes. An NFS export is defined as a PersistentVolume, and Pods claim it with a PersistentVolumeClaim.

This provides a simple way to give stateful applications persistent storage without deploying complex storage solutions.

The Kubernetes control plane does not manage NFS. It simply mounts the NFS share on the node where the Pod runs.

The NFS server must be maintained separately.

---

## Interview Questions

### What is NFS and why would you use it?

NFS is a protocol for sharing files over a network. I would use it when multiple servers need access to the same files, such as shared home directories, web application assets, or backup storage. 
It is simple to configure and works well for environments that do not need complex distributed filesystem features.

### How do you export a directory on an NFS server?

I add an entry in `/etc/exports` specifying the directory, allowed clients, and options. Then I run `exportfs -a` to apply the changes and restart the nfs-server service.

### How do you mount an NFS share permanently on a client?

I add an entry in `/etc/fstab` with the server IP, export path, mount point, filesystem type `nfs`, and options like `defaults,_netdev`. The `_netdev` option ensures the system waits for the network before attempting the mount.

### What is root_squash and why is it important?

`root_squash` maps the root user on the client to the `nobody` user on the server. This prevents a client with root access from having root privileges on the server. It is an important security feature that limits the impact of a compromised client.

### What is the difference between sync and async?

`sync` confirms writes to disk before responding, ensuring data integrity. `async` is faster but risks data loss if the server crashes. `sync` is used for critical data, `async` for performance-sensitive temporary data.

### How do you check what directories an NFS server is sharing?

I use the command `showmount -e server-ip`. This lists all exports available from that server.

### What is a stale file handle error?

It occurs when a client tries to access a file that was removed or changed on the server while the client still had it mounted. It is fixed by unmounting and remounting the share on the client.

---

## Troubleshooting Common Issues

### Permission Denied

The client IP is not allowed in `/etc/exports`, or export options are too restrictive. Check `/etc/exports` and run `exportfs -a` after changes.

### Stale File Handle

The directory on the server was removed or changed while the client still had it mounted. Unmount and remount the share to fix.

### Connection Refused

NFS service is not running or firewall is blocking ports. Check service status and firewall rules.

### Slow Performance

Caused by network latency, `sync` option, or slow disk I/O on server. Consider using `async` or improving network.

---

## Hands-on Practical Steps

### Step 1: Configure NFS Server

Set hostname:

```bash
hostnamectl set-hostname NFSServer
hostname

### Install NFS Server & enable NFS Service 

dnf install nfs-utils -y
systemctl start nfs-server
systemctl enable nfs-server
systemctl status nfs-server

![Service Running](images/1-Service.png)

Step 2: Create a Share Directory

mkdir -p /mnt/nfs_shares/docs
ls -ld /mnt/nfs_shares/docs

We can see owner and group is root for our share. We want to make user nobody as owner of this directory so we need to change owner and permissions.

chown -R nobody:nobody /mnt/nfs_shares/docs

(Here we used user nobody because special system user with minimal privileges & mapped via root_squash to prevent root level access from client system.
We don't use our regular user here as it's not suitable for NFS exports that multiple clients access.)

ls -ld /mnt/nfs_shares/docs

Output: We can now see owner and group is now user nobody (nobody nobody).

To see more details about user nobody: cat /etc/passwd | grep nobody

Output: nobody:x:65534:65534:Kernel Overflow User:/:/sbin/nologin

This means nobody is a special system user with UID 65534, no login shell, and minimal privileges. When user has ID 65534, it means it is a user with low privileges.

Now we want to use this share as read-write access, so client machine users can create files as well. We need to give them full permissions.





