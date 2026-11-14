Follow in exact sequence given below

## 1. Prune Unused Docker Data
```
docker system prune -a --volumes -f
```
## 2. Exiting Docker Desktop Gracefully
```
docker desktop stop
```
>Note: If you are running an older version of Docker Desktop that does not support the docker desktop subcommand, you can trigger a graceful shutdown via PowerShell by first stopping your containers and then sending a >standard close signal to the main application window:
```
# 1. Gracefully stop all running containers
docker ps -q | ForEach-Object { docker stop $_ }

# 2. Request the Docker Desktop window to close cleanly
(Get-Process "Docker Desktop" -ErrorAction SilentlyContinue).CloseMainWindow()
```
## 3. Shut Down WSL
```
wsl --shutdown
```
## 4. Compact Disks
  * Find your docker data folder with filename extensions like `.vhdx`, `.vhd`. 
    * `docker_data.vhdx` (e.g `%LOCALAPPDATA%\Docker\disk\docker_data.vhdx`) or your custom folder.     
    * `ext4.vhdx` (e.g `%LOCALAPPDATA%\Docker\main\ext4.vhdx`) or your custom folder.
  * Command windows command line (or powershell)
  * run `diskpart`
  * Compact `docker_data.vhdx`
    * Select vdisk - e.g  `select vdisk file="%LOCALAPPDATA%\Docker\disk\docker_data.vhdx"`
    * followed by `compact vdisk`
  * Compact `ext4.vhdx`
    * Select vdisk - e.g `select vdisk file="%LOCALAPPDATA%\Docker\main\ext4.vhdx"`
    * followed again by `compact vdisk`
  * press `exit` to exit from `diskpart` utility.
* Restart Docker Desktop. 
