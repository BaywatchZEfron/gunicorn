# Despliegue y Configuración de la API TicTacToe

## Memoria Técnica

## 1. Introducción

El objetivo de esta práctica es desplegar la aplicación API TicTacToe en un entorno de producción real utilizando únicamente una máquina Ubuntu sin contenedores Docker.
Para ello, se han empleado las herramientas y tecnologías vistas en clase: Git para obtener el proyecto, uv como entorno de ejecución de Python, Gunicorn como servidor WSGI y Nginx como reverse proxy.

La intención es partir de un entorno limpio y realizar una instalación y configuración manual, replicando un escenario profesional de despliegue sin depender de imágenes preconfiguradas.

---

## 2. Entorno utilizado

* Máquina virtual Ubuntu 22.04 LTS
* Despliegue sin Docker
* Proyecto clonado desde GitHub
* Python gestionado mediante uv (entorno virtual ligero)
* Gunicorn como servidor de aplicaciones
* Nginx como reverse proxy
* Port forwarding activado desde la máquina anfitriona al puerto 3000 de la máquina virtual
* Editor utilizado: Hugo para generar la documentación

---

## 3. Preparación del entorno y obtención del proyecto

### 3.1 Instalación de dependencias básicas

Se instaló el gestor de paquetes uv, utilizado como alternativa a pip y venv:

```
curl -LsSf https://astral.sh/uv/install.sh | sh
source ~/.profile
```

Se instaló Nginx desde los repositorios oficiales:

```
sudo apt update
sudo apt install nginx -y
```

### 3.2 Clonado del repositorio TicTacToe

```
git clone https://github.com/usuario/tictactoe-back.git
cd tictactoe-back
```

### 3.3 Creación del entorno virtual con uv

```
uv venv
source .venv/bin/activate
```

### 3.4 Instalación de dependencias del proyecto

```
uv pip install flask gunicorn
```

### Capturas recomendadas para esta sección

* Terminal mostrando la instalación de uv
* Terminal mostrando la instalación de Nginx
* Captura del directorio tras clonar el repositorio
* Comprobación del entorno uv activado

---

## 4. Pruebas locales en modo desarrollo (fase 1 ya completada)

Se verificó que la API funcionaba correctamente antes de proceder a la fase de despliegue:

```
uv run flask run --host=0.0.0.0 --port=8000
```

Posteriormente se probó con un cliente HTTP:

```
curl http://localhost:8000/
```

### Capturas recomendadas

* Ejecución de Flask en modo desarrollo
* Respuesta de curl o captura del navegador

---

## 5. Instalación del servidor de aplicaciones (Fase 2)

La instalación del servidor se realizó con Gunicorn, ya que se trata de un servidor WSGI recomendado para aplicaciones en Flask.

### 5.1 Ejecución de Gunicorn

```
uv run gunicorn -b 0.0.0.0:8000 app:app
```

Se verificó que la API seguía respondiendo correctamente mediante curl:

```
curl http://localhost:8000
```

### 5.2 Justificación técnica de las herramientas utilizadas

**uv**
Se ha utilizado uv como entorno de ejecución moderno y eficiente, que sustituye tanto a pip como a virtualenv, simplificando la gestión de dependencias y entornos aislados.

**Gunicorn**
Gunicorn es un servidor WSGI ampliamente utilizado en entornos de producción con aplicaciones Flask. Permite ejecutar varios workers, gestionar peticiones concurrentes y comunicarse correctamente con un reverse proxy como Nginx.

**Nginx**
Se utiliza Nginx como reverse proxy para exponer la API en un puerto accesible desde el exterior (3000) y derivar las peticiones a Gunicorn en el puerto interno 8000.

### Capturas recomendadas

* Gunicorn ejecutándose
* Comandos de verificación con curl
* Comprobación del puerto 8000 activo con `ss -tulpn`

---

## 6. Configuración del servidor de aplicaciones (Fase 3)

### 6.1 Configuración de Nginx

Se creó el archivo de configuración:

`/etc/nginx/sites-available/tictactoe`

Contenido del archivo:

```
server {
    listen 3000;
    server_name _;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

Se activó el sitio mediante un enlace simbólico:

```
sudo ln -s /etc/nginx/sites-available/tictactoe /etc/nginx/sites-enabled/
```

Se comprobó que no había errores de sintaxis:

```
sudo nginx -t
```

Y finalmente se reinició el servicio:

```
sudo systemctl restart nginx
```

### Capturas recomendadas

* Archivo de configuración abierto en el editor
* Resultado del comando `nginx -t`
* Estado de Nginx con `systemctl status nginx`
* Prueba en `http://localhost:3000` desde la VM
* Prueba en el puerto forwardeado desde el navegador de la máquina anfitriona

---

## 7. Comunicación entre Gunicorn y Nginx

La API queda expuesta a través del puerto 3000 mediante Nginx, mientras que Gunicorn escucha en el puerto interno 8000.

Esquema del flujo:

1. Cliente accede a `http://localhost:3000`
2. Nginx recibe la petición
3. Nginx la reenvía hacia `http://127.0.0.1:8000`
4. Gunicorn procesa la lógica de la API
5. La respuesta vuelve a través de Nginx al cliente

Este modelo sigue el estándar de despliegue usado para aplicaciones basadas en Flask.

### Capturas recomendadas

* Navegador accediendo a la API en puerto 3000
* Curl desde el anfitrión mostrando la respuesta

---

## 8. Automatización del servicio con systemd (opcional pero recomendable)

Se creó la unidad:

`/etc/systemd/system/tictactoe.service`

```
[Unit]
Description=TicTacToe API Gunicorn Service
After=network.target

[Service]
User=ubuntu
WorkingDirectory=/home/ubuntu/tictactoe-back
Environment="PATH=/home/ubuntu/tictactoe-back/.venv/bin"
ExecStart=/home/ubuntu/tictactoe-back/.venv/bin/gunicorn -b 0.0.0.0:8000 app:app

[Install]
WantedBy=multi-user.target
```

Comandos utilizados:

```
sudo systemctl daemon-reload
sudo systemctl enable tictactoe
sudo systemctl start tictactoe
```

### Capturas recomendadas

* Archivo systemd abierto
* Estado activo del servicio con `systemctl status tictactoe`

---

## 9. Verificación final

Se probó el servicio desde:

* La propia máquina Ubuntu
* La máquina anfitriona mediante port forwarding

Ejemplo:

```
curl http://localhost:3000/api/status
```

Resultado correcto de la API.

---

## 10. Conclusiones y mejoras posibles

El despliegue del servidor TicTacToe utilizando uv, Gunicorn y Nginx ha permitido replicar un entorno de producción estándar. El proceso se ha realizado íntegramente en Ubuntu, sin contenedores Docker y construyendo la configuración desde cero.

Posibles mejoras:

* Añadir HTTPS mediante certificados Let's Encrypt
* Escalar Gunicorn con más workers
* Configurar logs rotativos para Nginx y Gunicorn
* Implementar la fase avanzada con dominio local personalizado
* Estudiar la posibilidad de ejecutar varias instancias balanceadas con upstream de Nginx

---

## 11. Archivos incluidos en la entrega

* Código completo del proyecto TicTacToe
* Archivo de configuración Nginx
* Unidad systemd
* Carpetas generadas por uv
* Documentación en Hugo

---


