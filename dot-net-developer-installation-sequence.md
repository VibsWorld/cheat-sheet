# Installation Sequence for a Senior Devs Engineer after complete format   

1. Device Drivers
    1. Bios + Chipset
    2. Management Controller
    3. Graphics
    4. Audio e.g. Realtek
    5. Dell Command Update
    6. Dashboard SSD
        1. [WD Green SSD Dashboard](https://wddashboarddownloads.wdc.com/wdDashboard/DashboardSetupSA.exe)
        2. [Transcend SSD Scope](https://www.transcend-info.com/Support/Software-10/)
2. [Bloatware Uninstall from Windows 10](scripts/windows/windows10_bloatware_uninstall.md)
    1. `Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass`
    2. Uninstall default installed onedrive
4. Update Windows
5. Update MS Store
6. Install Winget - [App Installer - Microsoft Apps](https://apps.microsoft.com/detail/9nblggh4nns1?rtc=1&hl=en-in&gl=IN#activetab=pivot:overviewtab)
7. Install new windows powershell - `winget install --id Microsoft.PowerShell --source winget`
    ```
    winget upgrade --all --allow-reboot --nowarn --ignore-warnings --ignore-security-hash --wait --include-unknown
    ```
8.  Install [Onedrive](https://www.onedrive.com/download) followed by ~~Office 365~~ - [Libre Office](https://www.libreoffice.org/download/download-libreoffice/)
9.  Notepad++ - [Downloads | Notepad++ (notepad-plus-plus.org)](https://notepad-plus-plus.org/downloads/)
10. Adobe Reader - [https://get.adobe.com/reader/](https://get.adobe.com/reader/)
11. Kdiff - [KDiff3 - Homepage (sourceforge.net)](https://kdiff3.sourceforge.net/)
12. Git SCM - [Git - Downloads (git-scm.com)](https://git-scm.com/downloads)
13. Run powershell as administrator and type `wsl --install` and reboot.
14. Docker Desktop - [Install Docker Desktop on Windows | Docker Docs](https://docs.docker.com/desktop/install/windows-install/)
15. Remaining dot net sdks if any
    1. Dot net 6 SDK - [Download .NET 6.0 (Linux, macOS, and Windows) (microsoft.com)](https://dotnet.microsoft.com/en-us/download/dotnet/6.0)
    2. Dot net 8 SDK - [https://dotnet.microsoft.com/en-us/download/dotnet/8.0](https://dotnet.microsoft.com/en-us/download/dotnet/8.0)
16. Visual Studio - [Home - Visual Studio Subscriptions Portal](https://my.visualstudio.com/?auth_redirect=true)
17. `dotnet tool install -g csharpier`
    1. csharpier extension for VS studio - [https://marketplace.visualstudio.com/items?itemName=csharpier.CSharpier](https://marketplace.visualstudio.com/items?itemName=csharpier.CSharpier)
19. Azure CLI [https://go.microsoft.com/fwlink/?linkid=872496](https://go.microsoft.com/fwlink/?linkid=872496)
20. Kubernets [https://go.microsoft.com/fwlink/?linkid=2233742](https://go.microsoft.com/fwlink/?linkid=2233742)
21. Add Workplace Account
22. `az aks install-cli`
    ```plain
    az login
    az account set --subscription 5f6e519f-d331-4cc1-b5d9-cb21d8b92c42
    az aks get-credentials --resource-group dev-aks --name dev-aks
    az aks get-credentials --resource-group prod-aks --name prod-aks
    kubectl get deployments --all-namespaces=true
    ```    
23. Install K6 `winget install -e --id k6.k6`
24. Management Studio [https://aka.ms/ssmsfullsetup](https://aka.ms/ssmsfullsetup)
    1. Alternatively you can use docker `docker run --name sqlpad -p 3000:3000 -e "SQLPAD_ADMIN=admin@sqlpad.com" -e "SQLPAD_ADMIN_PASSWORD=admin" sqlpad/sqlpad:latest`
25. PgAdmin - [https://www.pgadmin.org/download/](https://www.pgadmin.org/download/) (Not recommended as it is slow and takes C Drive space)
    1. Installation of PG Admin via docker (PREFERRED if you can move your docker images to another drive)
       ```
       docker run --name pgadmin2 -p 5050:80 -e 'PGADMIN_DEFAULT_EMAIL=pgadmin4@pgadmin.org' -e 'PGADMIN_DEFAULT_PASSWORD=admin' -e 'PGADMIN_CONFIG_SERVER_MODE=True' -e 'PGADMIN_CONFIG_UPGRADE_CHECK_ENABLED=False' -d dpage/pgadmin4
       ``` 
27. Mongo db Compass - [https://www.mongodb.com/try/download/compass](https://www.mongodb.com/try/download/compass)
28. Communication
    1. Zoom - [https://zoom.us/download#client\_4meeting](https://zoom.us/download#client_4meeting)
    2. Slack - [Windows | Downloads | Slack](https://slack.com/intl/en-gb/downloads/windows)
    3. Clickup - [Download ClickUp™ | Mobile & Desktop App, Chrome, Alexa, Google Home](https://clickup.com/download)
29. [VS Code](https://code.visualstudio.com/download)
30. One Drive Business
31. Forticlient - [https://www.fortinet.com/support/product-downloads](https://www.fortinet.com/support/product-downloads)
    1. Login Information - Restore Configuration. `C:\Users\vibs2\OneDrive\Installs\Forticlient VPN` (password - `vibs*****` ) 
32. 7 Zip - [Download (7-zip.org)](https://www.7-zip.org/download.html)
33. Postman Windows Application [Download Postman | Get Started for Free](https://www.postman.com/downloads/)
