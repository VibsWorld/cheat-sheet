# How to Reclaim Disk Space from Docker (Windows & Linux)

On Windows, Docker Desktop stores all your containers, images, and volumes inside virtual hard disks (`.vhdx` files). While these files expand automatically as you pull images and write data, **they never automatically shrink** when you delete containers or clean up files.

Follow this guide to compact your virtual disks and recover tens of gigabytes of storage.

---

## Windows (10 & 11 — Home, Pro, and Enterprise)

Follow these steps in the exact sequence below.

### 1. Clean Up Unused Docker Data (Optional)
> ⚠️ **Warning:** Skip this step if you want to keep all your existing stopped containers, images, and volumes. Even without pruning, compacting the disk will still reclaim space from files you deleted *previously*.

* **Safe Cleanup** (removes only dangling/untagged images and build cache):
  ```bash
  docker image prune -f && docker builder prune -f
  ```
* **Aggressive Cleanup** (deletes **all** stopped containers, unused images, networks, and volumes):
  ```bash
  docker system prune -a --volumes -f
  ```

### 2. Exit Docker Desktop Gracefully
Before compacting the virtual disks, you must completely stop Docker Desktop so it releases file locks and stops restarting WSL in the background.

```powershell
docker desktop stop
```

> **Note:** If you are running an older version of Docker Desktop that does not support the `docker desktop` subcommand, run this in PowerShell to shut it down cleanly:
> ```powershell
> # 1. Gracefully stop all running containers
> docker ps -q | ForEach-Object { docker stop $_ }
> 
> # 2. Request the Docker Desktop window to close cleanly
> (Get-Process "Docker Desktop" -ErrorAction SilentlyContinue).CloseMainWindow()
> ```

### 3. Shut Down WSL
Stop all WSL instances completely so Windows unlocks the `.vhdx` files:

```powershell
wsl --shutdown
```

Verify that all distributions show a `Stopped` state before continuing:
```powershell
wsl --list --verbose
```

### 4. Locate and Compact Your Virtual Disks
Open your Docker data directory (default is `%LOCALAPPDATA%\Docker\`) and look for `.vhdx` files. Depending on your Docker Desktop version, you will typically see one or both of these:
* `%LOCALAPPDATA%\Docker\disk\docker_data.vhdx`
* `%LOCALAPPDATA%\Docker\main\ext4.vhdx` *(or `data\ext4.vhdx` on older versions)*

Choose **Option A** if you use Windows 11 Pro/Enterprise, or **Option B** for Windows Home (or if Hyper-V tools aren't enabled).

#### Option A: Windows 11 Pro / Enterprise (Faster — via PowerShell)
1. Open **PowerShell as Administrator**.
2. Run `Optimize-VHD` for each `.vhdx` file you found:
   ```powershell
   Optimize-VHD -Path "$env:LOCALAPPDATA\Docker\disk\docker_data.vhdx" -Mode Full
   Optimize-VHD -Path "$env:LOCALAPPDATA\Docker\main\ext4.vhdx" -Mode Full
   ```

#### Option B: Windows Home & Pro (Universal — via `diskpart`)
1. Open **Command Prompt** or **PowerShell as Administrator** and launch `diskpart`:
   ```cmd
   diskpart
   ```
2. Select and compact `docker_data.vhdx`:
   ```cmd
   select vdisk file="%LOCALAPPDATA%\Docker\disk\docker_data.vhdx"
   compact vdisk
   ```
3. Select and compact `ext4.vhdx`:
   ```cmd
   select vdisk file="%LOCALAPPDATA%\Docker\main\ext4.vhdx"
   compact vdisk
   ```
4. Exit the utility:
   ```cmd
   exit
   ```

### 5. Restart Docker Desktop
Launch Docker Desktop again from your Start Menu. Check the file size of your `.vhdx` files in File Explorer—you should see a dramatic drop in disk usage.

---

## What About Linux?

How you reclaim space on Linux depends on whether you run native **Docker Engine** or **Docker Desktop for Linux**.

### 1. Native Docker Engine (Most Common on Linux)
Unlike Windows, native Docker on Linux does **not** use a virtual hard disk (`.vhdx`). It writes directly to your host filesystem (usually `/var/lib/docker` using the `overlay2` storage driver).

Because there is no virtual disk wrapper, **Linux immediately frees up host disk space** the moment you delete containers, images, or volumes:
```bash
# Reclaim space from unused images, containers, and build cache
docker system prune -a -f

# (Optional) Include unused volumes
docker volume prune -f
```
No shutdown or disk compaction step is required.

### 2. Docker Desktop for Linux
If you use **Docker Desktop** on Ubuntu, Debian, or Fedora, it runs inside a lightweight QEMU virtual machine and stores data in a virtual disk image (`qcow2` or `raw` format) located at:
```bash
~/.docker/desktop/vms/0/data/Docker.raw
```

To reclaim space on Docker Desktop for Linux:
1. Prune unused data inside Docker (`docker system prune -a -f`).
2. Quit Docker Desktop completely:
   ```bash
   systemctl --user stop docker-desktop
   ```
3. **If using `.raw`:** Linux sparse files don't always shrink automatically. You can punch holes into unused zeroed blocks using `fallocate`:
   ```bash
   fallocate --dig-holes ~/.docker/desktop/vms/0/data/Docker.raw
   ```
4. **If using `.qcow2`:** Shrink the image using `qemu-img`:
   ```bash
   qemu-img convert -O qcow2 ~/.docker/desktop/vms/0/data/Docker.qcow2 ~/.docker/desktop/vms/0/data/Docker_compact.qcow2
   mv ~/.docker/desktop/vms/0/data/Docker_compact.qcow2 ~/.docker/desktop/vms/0/data/Docker.qcow2
   ```
5. Start Docker Desktop again:
   ```bash
   systemctl --user start docker-desktop
   ```
