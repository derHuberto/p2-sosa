## Prerrequisitos

### Actualizar el sistema

```
sudo apt update && sudo apt upgrade -y
```

### Instalar Docker y Docker Compose

```
sudo apt install -y ca-certificates curl gnupg lsb-release
sudo mkdir -p /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

---

## Limpiar Contenedores Antiguos (opcional)

```
docker stop $(docker ps -aq) 2>/dev/null
docker rm -f $(docker ps -aq) 2>/dev/null
docker rmi -f $(docker images -aq) 2>/dev/null
docker volume rm $(docker volume ls -q) 2>/dev/null
docker network rm $(docker network ls -q) 2>/dev/null
```

---

## Crear una Red Compartida

Esto permite que los contenedores se comuniquen entre sí de forma segura. Usaremos el nombre del archivo compose.

```
docker network create ollama-net
```

---

## Ejecutar Ollama (Entorno de Modelos)

Este comando está actualizado para usar el nuevo nombre de red y la política de reinicio `unless-stopped`. El mapeo de puertos (`-p 11434:11434`) se elimina, ya que el archivo compose implica que Ollama solo será accedido internamente por otros contenedores.

```
docker run -d \
  --name ollama \
  --network ollama-net \
  -v ollama:/root/.ollama \
  --restart unless-stopped \
  ollama/ollama:latest
```

**Explicación:**

- `-d` → modo desconectado (detached)
    
- `--name ollama` → nombre del contenedor
    
- `--network ollama-net` → se une a la red compartida de Docker
    
- `-v ollama:/root/.ollama` → almacenamiento persistente para modelos
    
- `--restart unless-stopped` → se reinicia automáticamente a menos que se detenga manually
    
- `ollama/ollama:latest` → nombre de la imagen de Docker Hub
    

---

## Ejecutar Open WebUI (Frontend + Backend)

Está actualizado con la nueva red, la política `unless-stopped` y variables de entorno adicionales del archivo compose. El puerto (`-p 3000:8080`) se elimina porque la configuración del compose está pensada para que se acceda solo a través del proxy inverso Nginx.

```
docker run -d \
  --name open-webui \
  --network ollama-net \
  -e OLLAMA_API_BASE=http://ollama:11434 \
  -e OLLAMA_BASE_URL=http://ollama:11434 \
  -v open-webui:/app/backend/data \
  --restart unless-stopped \
  ghcr.io/open-webui/open-webui:main
```

**Explicación:**

- OLLAMA_API_BASE / OLLAMA_BASE_URL
    
    ↳ Le dice al backend de WebUI que se conecte internamente al contenedor ollama.
    
- Otras opciones replican la configuración de Ollama.
    

---

## Ejecutar Glances (Monitorización)

```
docker run -d \
  --name glances \
  --network ollama-net \
  --restart unless-stopped \
  nicolargo/glances:latest \
  glances -w
```

**Explicación:**

- `--name glances` → nombre del contenedor
    
- `--network ollama-net` → se une a la red compartida de Docker
    
- `--restart unless-stopped` → se reinicia automáticamente a menos que se detenga manualmente
    
- `nicolargo/glances:latest` → nombre de la imagen
    
- `glances -w` → comando para ejecutar la interfaz del servidor web
    

---

## Descargar un Modelo

```
docker exec -it ollama ollama pull deepseek-r1:1.5b
```

```
docker exec -it ollama ollama list
```

---

## Añadir Proxy Inverso Nginx (opcional)

```
docker run -d \
  --name nginx \
  --network ollama-net \
  -p 80:80 \
  -v $(pwd)/default.conf:/etc/nginx/conf.d/default.conf:ro \
  --restart unless-stopped \
  nginx:latest
```

### Ejemplo de `default.conf`:

Este archivo de configuración reemplaza al antiguo `nginx.conf`. Debe llamarse `default.conf` y ubicarse en el directorio correcto para que coincida con el montaje del volumen. Esta versión actualizada redirige `/` a Open WebUI (puerto 8080) y añade la ubicación `/monitor/` para el proxy del nuevo servicio `glances` (puerto 61208).

```
server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://open-webui:8080;

        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_send_timeout 300s;
    }

    location /monitor/ {
        proxy_pass http://glances:61208/;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

---
