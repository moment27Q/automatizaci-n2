# Cobrapp

Automatización de registro de pagos con OCR, API en FastAPI y flujo orquestado con n8n.

## Descripción

Este proyecto permite procesar imágenes de comprobantes de pago (por ejemplo, Yape o Plin), extraer datos clave con OCR (Tesseract) y exponerlos mediante una API.

Componentes principales:
- `python-api`: servicio FastAPI que recibe una imagen, aplica preprocesamiento y extrae datos.
- `n8n`: motor de automatización para integrar flujos (webhooks, validación, almacenamiento, etc.).
- `pagos/`: carpeta compartida para almacenar o procesar archivos de pagos.

## Arquitectura

- `n8n` corre en `http://localhost:5678`
- `python-api` corre en `http://localhost:8000`
- Ambos servicios se levantan con `docker-compose`.
- La carpeta local `./pagos` se monta dentro del contenedor `python-api` en `/app/pagos`.

## Requisitos

- Docker
- Docker Compose

## Puesta en marcha

Desde la raíz del proyecto:

```bash
docker compose up --build
```

Servicios esperados:
- n8n: `http://localhost:5678`
- API FastAPI: `http://localhost:8000`

## Configuración actual de n8n

Variables definidas en `docker-compose.yml`:
- `N8N_BASIC_AUTH_ACTIVE=true`
- `N8N_BASIC_AUTH_USER=admin`
- `N8N_BASIC_AUTH_PASSWORD=admin123`
- `WEBHOOK_URL=https://expenses-turban-generous.ngrok-free.dev`

Nota: para producción, cambia credenciales y URL de webhook por valores seguros.

## API: procesamiento de imagen

### Endpoint

`POST /procesar-imagen`

### Tipo de entrada

`multipart/form-data` con campo:
- `file`: imagen del comprobante.

### Ejemplo con cURL

```bash
curl -X POST "http://localhost:8000/procesar-imagen" \
  -H "accept: application/json" \
  -H "Content-Type: multipart/form-data" \
  -F "file=@comprobante.jpg"
```

### Respuesta esperada (ejemplo)

```json
{
  "monto": "25.00",
  "operacion": "123456789",
  "fecha": "11/05/2026",
  "tipo": "Yape",
  "texto_raw": "...",
  "valido": true
}
```

## Lógica de extracción (python-api)

El servicio hace lo siguiente:
1. Lee la imagen enviada.
2. Preprocesa con OpenCV:
- escala de grises
- umbral binario
3. Ejecuta OCR con `pytesseract` en español (`lang='spa'`).
4. Extrae:
- `monto` (regex)
- `operacion` (regex)
- `fecha` (regex, o fecha actual si no aparece)
- `tipo` (`Yape`, `Plin` o `Desconocido`)
5. Marca `valido=true` si detecta monto.

## Estructura del proyecto

```text
cobrapp/
├─ docker-compose.yml
├─ pagos/
└─ python-api/
   ├─ Dockerfile
   ├─ main.py
   └─ requirements.txt
```

## Dependencias Python

Definidas en `python-api/requirements.txt`:
- fastapi
- uvicorn
- pytesseract
- Pillow
- opencv-python-headless
- python-multipart
- openpyxl

## Notas técnicas

- El contenedor instala `tesseract-ocr` y `tesseract-ocr-spa`.
- Si OCR falla por calidad de imagen, mejora iluminación, nitidez o resolución del comprobante.
- Si cambias expresiones de formato (banco/app), ajusta las regex en `extraer_datos()`.

## Próximas mejoras sugeridas

- Validación de esquema de respuesta con Pydantic.
- Manejo de errores y logs estructurados.
- Tests automáticos para OCR y parsing.
- Endpoints adicionales para guardar resultados en Excel/DB.
- Variables sensibles con `.env`.
