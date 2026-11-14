* Prune Unused Docker Data
```
docker system prune -a --volumes -f
```
* Shut Down WSL
```
wsl --shutdown
```
* Compact Disks
  * Find your docker data folder (filenames might vary but do look for `.vhdx`, `.vhd` file extensions. 
    * `docker_data.vhdx` (e.g `%LOCALAPPDATA%\Docker\disk\docker_data.vhdx`) or your custom folder.     
    * `ext4.vhdx` (e.g `%LOCALAPPDATA%\Docker\main\ext4.vhdx`) or your custom folder
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
