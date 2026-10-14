## Run Docker for Open WebUI
```
docker run -d -p 3000:8080 -e OLLAMA_BASE_URL=http://192.168.10.23:11434 -v open-webui:/app/backend/data --name open-webui --restart always ghcr.io/open-webui/open-webui:main
```
