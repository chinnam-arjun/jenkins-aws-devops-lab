# INTERNAL_STORAGE_ALLOCATION.md

# Jenkins Internal Storage Allocation and Temporary Directory Management on AWS EC2

This document explains a storage allocation issue encountered while running Jenkins on an AWS EC2 Ubuntu instance.

The issue occurred even though the EC2 instance had sufficient free space on its root filesystem.

Jenkins reported insufficient temporary directory storage because `/tmp` was mounted as a memory-backed `tmpfs` filesystem with a size limit below the required threshold.

This guide explains:

- How Linux filesystem allocation works.
- Difference between EBS storage and `tmpfs`.
- Why Jenkins reports insufficient temporary storage.
- How to inspect storage allocation.
- How to increase `/tmp` capacity.
- Temporary and persistent solutions.
- Common mistakes and troubleshooting.
- Production-oriented storage recommendations.

---

# 1. Problem Overview

While configuring the Jenkins controller on an AWS EC2 Ubuntu instance, Jenkins displayed a warning indicating insufficient temporary directory storage.

The warning indicated that the available temporary storage was below the recommended 1 GiB threshold.

However, checking the root filesystem showed sufficient free disk space.

### Observed Environment

| Resource | Observed Value |
|---|---|
| Operating System | Ubuntu |
| Root filesystem | Approximately 19 GB |
| Root filesystem free space | Approximately 16 GB |
| `/tmp` filesystem | tmpfs |
| `/tmp` total allocation | Approximately 953 MiB |
| Jenkins temporary storage threshold | 1 GiB |
| Available `/tmp` space | Approximately 947.83 MiB |
| RAM | Approximately 1.9 GiB |
| Swap | Not configured |

### The Problem

The root filesystem had enough available storage, but Jenkins was checking the filesystem used for its temporary directory.

The `/tmp` directory was mounted using `tmpfs`, which is backed by memory rather than the root EBS filesystem.

Therefore:

```text
Root filesystem free space != /tmp available space
```

This was a filesystem allocation issue, not necessarily an EBS disk capacity issue.

---

# 2. Understanding Linux Storage Architecture

Before resolving the issue, it is important to understand how Linux handles storage.

A typical AWS EC2 Ubuntu instance may have:

- EBS-backed root filesystem.
- Additional EBS volumes.
- Memory-backed `tmpfs` filesystems.
- Temporary directories.
- Swap space, if configured.

These resources serve different purposes.

## 2.1 EBS-backed root filesystem

Amazon Elastic Block Store (EBS) provides persistent block storage to EC2 instances.

Ubuntu commonly uses the root EBS volume for:

- Operating system files.
- Installed applications.
- Jenkins installation.
- Jenkins configuration.
- Jenkins jobs and workspaces.
- Application data.
- Logs.

Example:

```text
/dev/root       19G
```

Inspect the root filesystem:

```bash
df -h /
```

Example output:

```text
Filesystem      Size  Used Avail Use% Mounted on
/dev/root        19G   3G   16G   16% /
```

This indicates that the root filesystem has approximately 16 GB available.

However, this does not guarantee that every directory has the same available capacity.

## 2.2 Understanding tmpfs

`tmpfs` is a temporary filesystem that uses virtual memory.

It is commonly used for:

- `/tmp`
- `/run`
- `/dev/shm`

Unlike an EBS filesystem, `tmpfs` does not directly consume a dedicated persistent disk allocation.

Its contents reside in memory and may use swap when swap is available and the system needs it.

Example:

```text
tmpfs           953M   1M   952M   1% /tmp
```

This means `/tmp` has a maximum configured capacity of approximately 953 MiB.

Even if the root filesystem has 16 GB free, `/tmp` cannot automatically use that space beyond its configured limit.

## 2.3 Understanding swap

Swap is disk-backed virtual memory used when RAM is under pressure.

It is different from:

- Root filesystem storage.
- `/tmp` filesystem capacity.
- RAM capacity.

If an instance has no swap, memory-backed temporary storage and application workloads compete for the available physical memory.

Adding swap can provide additional memory-pressure tolerance, but it does not increase physical RAM or guarantee good performance.

---

# 3. Why Jenkins Requires Temporary Storage

Jenkins uses temporary directories during its execution.

Temporary storage may be used for:

- Pipeline execution.
- Temporary files.
- Plugin operations.
- Build-related processing.
- Java temporary files.
- File extraction and processing.

Jenkins runs on the Java Virtual Machine (JVM), which also uses a temporary directory.

The default Java temporary directory can be inspected using:

```bash
java -XshowSettings:properties -version 2>&1 | grep java.io.tmpdir
```

However, this command displays the default temporary directory for the Java invocation, not necessarily the exact temporary directory used by the running Jenkins process.

Inspect the Jenkins process:

```bash
ps -ef | grep '[j]enkins'
```

Look for:

```text
-Djava.io.tmpdir=...
```

Also inspect the running service:

```bash
sudo systemctl show jenkins --property=Environment
```

And inspect the Jenkins service configuration:

```bash
sudo systemctl cat jenkins
```

The effective temporary directory should be verified before making changes.

---

# 4. Diagnose the Storage Allocation Issue

Use these commands whenever Jenkins reports insufficient temporary storage.

## 4.1 Inspect all mounted filesystems

```bash
df -h
```

This displays:

- Filesystem size.
- Used storage.
- Available storage.
- Usage percentage.
- Mount point.

Pay particular attention to:

```text
/
 /tmp
 /run
 /dev/shm
```

## 4.2 Inspect the `/tmp` filesystem

```bash
df -h /tmp
```

Example:

```text
Filesystem      Size  Used Avail Use% Mounted on
tmpfs           953M   1M   952M   1% /tmp
```

## 4.3 Identify filesystem type

```bash
findmnt /tmp
```

Example:

```text
TARGET SOURCE FSTYPE OPTIONS
/tmp   tmpfs  tmpfs  rw,nosuid,nodev
```

This confirms that `/tmp` is using `tmpfs`.

## 4.4 Inspect memory allocation

```bash
free -h
```

Example:

```text
               total       used       free
Mem:           1.9Gi       ...
Swap:             0B         0B
```

## 4.5 Inspect mounted filesystems in detail

```bash
mount | grep tmpfs
```

This displays the active `tmpfs` mounts and their options.

## 4.6 Inspect disk partitions

```bash
lsblk
```

This displays block devices and their mount points.

For example:

```text
NAME     SIZE TYPE MOUNTPOINTS
nvme0n1   20G disk
└─...     20G part /
```

This helps distinguish persistent EBS storage from memory-backed filesystems.

---

# 5. Solution 1: Temporarily Increase `/tmp` Allocation

The immediate solution is to increase the size limit of the existing `tmpfs` mount.

In the encountered environment, `/tmp` was remounted with a 2 GB size limit.

Execute on the Jenkins Controller.

```bash
sudo mount -o remount,size=2G /tmp
```

Verify:

```bash
df -h /tmp
```

Expected output:

```text
Filesystem      Size  Used Avail Use% Mounted on
tmpfs           2.0G     0  2.0G   0% /tmp
```

Jenkins can now use the increased temporary directory capacity, subject to available system memory and other resource constraints.

### Important

This change is temporary.

If the instance is restarted, the `/tmp` mount configuration may return to its previous allocation.

The 2 GB size is also a maximum capacity, not an immediate reservation of 2 GB of physical RAM.

---

# 6. Solution 2: Make `/tmp` Allocation Persistent

To preserve the allocation across reboots, configure the mount through `/etc/fstab`.

Before modifying the file, create a backup:

```bash
sudo cp /etc/fstab /etc/fstab.backup
```

Open:

```bash
sudo nano /etc/fstab
```

Add:

```fstab
tmpfs /tmp tmpfs defaults,nosuid,nodev,size=2G,mode=1777 0 0
```

Save the file.

The configuration means:

| Option | Purpose |
|---|---|
| tmpfs | Use a memory-backed filesystem |
| /tmp | Mount location |
| defaults | Default mount options |
| nosuid | Ignore set-user-ID and set-group-ID bits |
| nodev | Prevent device interpretation |
| size=2G | Maximum filesystem capacity |
| mode=1777 | Standard temporary-directory permissions |
| 0 0 | No dump or filesystem check |

Apply the configuration:

```bash
sudo mount -o remount /tmp
```

Verify:

```bash
df -h /tmp
```

Verify mount options:

```bash
findmnt /tmp
```

### Validate before rebooting

Check the configuration:

```bash
sudo findmnt --verify --verbose
```

If the configuration is valid, schedule a controlled reboot or verify persistence during the next planned restart.

Important: Preserve any existing `/tmp` mount options that are required by your security configuration. Do not blindly add duplicate mount entries.

---

# 7. Alternative: Move Jenkins Temporary Files to EBS Storage

Increasing `tmpfs` is not always the best choice.

For an EC2 instance with limited RAM, a disk-backed temporary directory may be a better option.

This allows temporary files to consume EBS storage rather than relying entirely on memory-backed capacity.

## 7.1 Create a dedicated temporary directory

On the Jenkins Controller:

```bash
sudo mkdir -p /var/cache/jenkins/tmp
```

Set ownership:

```bash
sudo chown jenkins:jenkins /var/cache/jenkins/tmp
```

Set permissions:

```bash
sudo chmod 700 /var/cache/jenkins/tmp
```

## 7.2 Configure the Jenkins JVM temporary directory

Jenkins on Ubuntu packages commonly use:

```text
/etc/default/jenkins
```

Inspect the active configuration first:

```bash
sudo systemctl cat jenkins
```

Check whether the service uses `/etc/default/jenkins` or another environment configuration mechanism.

If the active service configuration supports `JAVA_OPTS` through `/etc/default/jenkins`, add:

```bash
JAVA_OPTS="$JAVA_OPTS -Djava.io.tmpdir=/var/cache/jenkins/tmp"
```

If the service uses a different configuration mechanism, add the JVM option through the supported systemd environment configuration instead.

Restart Jenkins:

```bash
sudo systemctl restart jenkins
```

Verify:

```bash
ps -ef | grep '[j]enkins'
```

Look for:

```text
-Djava.io.tmpdir=/var/cache/jenkins/tmp
```

Check the directory:

```bash
df -h /var/cache/jenkins/tmp
```

Because this directory resides on the root EBS filesystem, it can use the available disk capacity rather than the `/tmp` tmpfs limit.

### Important distinction

Changing Java's `java.io.tmpdir` changes the JVM temporary directory.

It does not automatically change every temporary directory used by shell commands, plugins, agents, or other services.

Verify the effective temporary directory for the specific workload.

---

# 8. Choosing the Right Storage Strategy

There are three possible approaches.

| Approach | Storage backing | Persistence | Best use |
|---|---|---|---|
| Increase `/tmp` tmpfs | RAM / virtual memory | No, unless configured | Small temporary workloads |
| Use EBS-backed temporary directory | EBS | Yes | Memory-constrained EC2 |
| Dedicated EBS volume | Separate persistent block storage | Yes | Larger Jenkins workloads |

### Recommended decision

For a small Jenkins controller:

- Keep the default `/tmp` configuration if it satisfies Jenkins requirements.
- If temporary storage is insufficient and RAM is constrained, consider a disk-backed JVM temporary directory.
- For larger workloads, use a dedicated EBS volume for Jenkins data and temporary files.

Avoid allocating a large `tmpfs` on an instance with only approximately 2 GB of RAM without monitoring memory consumption.

---

# 9. Common Mistakes and Troubleshooting

## 9.1 Root filesystem has free space but Jenkins still reports insufficient storage

**Cause:**

Jenkins is checking a different filesystem, such as `/tmp`.

**Resolution:**

```bash
df -h /
df -h /tmp
findmnt /tmp
```

Check the filesystem used by the Jenkins process.

## 9.2 `/tmp` allocation resets after reboot

**Cause:**

The remount command was temporary.

**Resolution:**

Configure `/etc/fstab` and validate the mount configuration.

## 9.3 Increasing tmpfs causes memory pressure

**Cause:**

A larger `tmpfs` allows more data to occupy memory-backed storage.

**Resolution:**

Inspect:

```bash
free -h

df -h /tmp

ps aux --sort=-%mem | head
```

Consider moving Jenkins temporary files to EBS-backed storage.

## 9.4 Jenkins cannot write to the new temporary directory

**Cause:**

Incorrect ownership or permissions.

**Resolution:**

```bash
sudo chown jenkins:jenkins /var/cache/jenkins/tmp

sudo chmod 700 /var/cache/jenkins/tmp
```

## 9.5 Disk-backed temporary directory becomes full

**Cause:**

Temporary files accumulate on the root EBS filesystem.

**Resolution:**

```bash
df -h /

sudo du -sh /var/cache/jenkins/tmp
```

Inspect old temporary files and clean them carefully.

Do not blindly delete files belonging to active Jenkins processes.

## 9.6 Jenkins still reports insufficient temporary storage after changing the mount

**Cause:**

The Jenkins JVM may be using a different temporary directory, or the service may not have restarted.

**Resolution:**

```bash
ps -ef | grep '[j]enkins'

sudo systemctl restart jenkins

df -h /tmp
```

Check the actual JVM `java.io.tmpdir` value.

---

# 10. Monitoring and Maintenance

Storage monitoring should be part of routine Jenkins maintenance.

### Check filesystem utilization

```bash
df -h
```

### Check inode utilization

```bash
df -i
```

### Check Jenkins disk usage

```bash
sudo du -sh /var/lib/jenkins
```

### Check Jenkins workspace usage

```bash
sudo du -sh /var/lib/jenkins/workspace
```

### Check temporary directory usage

```bash
sudo du -sh /tmp
```

### Check Docker storage on the Docker Agent

```bash
docker system df
```

### Inspect large directories

```bash
sudo du -xhd1 /var/lib/jenkins | sort -h
```

These checks help identify whether storage consumption comes from:

- Jenkins workspaces.
- Build artifacts.
- Jenkins logs.
- Temporary files.
- Docker images and containers.

For a distributed CI/CD environment, remember that the Docker agent and Jenkins controller have separate storage requirements.

Docker build cache and image layers primarily consume storage on the Docker agent, not automatically on the controller.

---

# 11. Production-Oriented Storage Recommendations

For production-oriented Jenkins infrastructure, storage should be designed according to workload requirements.

### Jenkins Controller

Recommended storage considerations:

- Dedicated EBS volume for Jenkins home.
- Adequate root filesystem capacity.
- Persistent storage for job configurations and build metadata.
- Monitoring of workspace and artifact growth.
- Appropriate backup strategy.
- Temporary directory allocation based on JVM requirements.

### Docker Agent

Recommended storage considerations:

- Adequate EBS capacity for Docker images and build cache.
- Monitoring of `/var/lib/docker`.
- Controlled Docker image cleanup.
- Separate workspace and cache management where appropriate.
- Avoid unnecessary accumulation of temporary build artifacts.

### Deployment Server

Recommended storage considerations:

- Sufficient space for application images.
- Container log rotation.
- Image cleanup policies.
- Persistent application data volumes when required.
- Monitoring of filesystem utilization.

---

# 12. Final Verification Checklist

- [ ] Root filesystem inspected.
- [ ] `/tmp` filesystem inspected.
- [ ] Filesystem type identified.
- [ ] Jenkins JVM temporary directory identified.
- [ ] RAM and swap inspected.
- [ ] `/tmp` allocation adjusted if necessary.
- [ ] Persistent mount configuration validated if used.
- [ ] Jenkins service restarted when JVM configuration changed.
- [ ] Temporary directory permissions verified.
- [ ] Jenkins service status verified.
- [ ] Filesystem utilization monitored.
- [ ] Docker storage checked on the remote agent.

---

# Conclusion

The Jenkins temporary storage issue demonstrates why understanding Linux filesystem allocation is important when configuring CI/CD infrastructure on AWS.

Having sufficient EBS capacity does not guarantee that every mounted filesystem has sufficient available storage.

In this environment, the root EBS filesystem had approximately 16 GB free, but `/tmp` was limited to approximately 953 MiB because it used `tmpfs`.

Increasing the `/tmp` allocation to 2 GB resolved the immediate issue.

For resource-constrained EC2 instances, disk-backed temporary storage may be a more appropriate long-term solution than allocating a large memory-backed filesystem.

**Key takeaway: Always inspect the actual filesystem used by a process before assuming that the root disk is the source of a storage warning.**