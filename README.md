# API Gateway - Nginx

API Gateway centralizado para el sistema de gestión de restaurante, encargado de enrutar solicitudes a los microservicios correspondientes y validar tokens JWT.

## 🎯 Propósito

El API Gateway actúa como punto de entrada único para todas las solicitudes del frontend, proporcionando:

- **Enrutamiento centralizado**: Redirige solicitudes a los microservicios correctos
- **Validación JWT**: Valida tokens antes de reenviar solicitudes a servicios protegidos
- **Inyección de headers**: Agrega información del usuario (ID, rol, email) a las solicitudes
- **CORS**: Maneja políticas de CORS para todas las solicitudes
- **Seguridad**: Protege endpoints sensibles y centraliza la autenticación

## 📡 Rutas Configuradas

### Rutas Públicas (Sin validación JWT)

| Ruta | Destino | Descripción |
|------|---------|-------------|
| `/api/auth/*` | `backend-identidad:8001` | Endpoints de autenticación (login, logout, refresh) |
| `/health` | Nginx | Health check del gateway |

### Rutas Protegidas (Con validación JWT)

| Ruta | Destino | Descripción |
|------|---------|-------------|
| `/api/operaciones/*` | `backend-operaciones:8000` | Gestión de mesas, órdenes, inventario, reportes |
| `/api/pos/*` | `backend-pos:8002` | Sistema de punto de venta |

### Rutas de Frontend

| Ruta | Destino | Descripción |
|------|---------|-------------|
| `/` | `frontend:5173` | Aplicación SvelteKit |
| `/ws` | `frontend:5173` | WebSocket para HMR (Hot Module Replacement) |

## 🔐 Flujo de Validación JWT

```
┌─────────┐       ┌──────────┐       ┌─────────────┐       ┌────────────┐
│Frontend │──────▶│  Nginx   │──────▶│  Identity   │──────▶│ Operaciones│
│         │       │ Gateway  │       │  Service    │       │  Service   │
└─────────┘       └──────────┘       └─────────────┘       └────────────┘
                       │                    │
                       │  1. GET /api/operaciones/mesas
                       │     Authorization: Bearer <JWT>
                       │
                       ├─────────────────▶  │
                       │  2. POST /api/auth/validate
                       │     Authorization: Bearer <JWT>
                       │
                       │ ◀─────────────────┤
                       │  3. 200 OK
                       │     X-User-ID: 1
                       │     X-User-Role: admin
                       │     X-User-Email: admin@restaurant.com
                       │
                       ├──────────────────────────────────────▶
                       │  4. GET /api/mesas
                       │     X-User-ID: 1
                       │     X-User-Role: admin
                       │     X-User-Email: admin@restaurant.com
                       │
                       │ ◀──────────────────────────────────────
                       │  5. 200 OK + datos de mesas
                       │
```

### Pasos del Flujo

1. **Frontend envía solicitud**: Incluye `Authorization: Bearer <JWT>` en el header
2. **Gateway valida token**: Hace una solicitud interna a `/api/auth/validate` del servicio de identidad
3. **Servicio de identidad valida**: Verifica firma JWT, expiración, blacklist y retorna información del usuario
4. **Gateway inyecta headers**: Agrega `X-User-ID`, `X-User-Role`, `X-User-Email` a la solicitud
5. **Servicio backend recibe**: Obtiene solicitud con headers de usuario sin necesidad de validar JWT nuevamente

## 🛠️ Configuración

### Variables de Entorno

No requiere variables de entorno específicas, pero depende de que los servicios estén accesibles:

- `backend-identidad:8001` - Servicio de identidad
- `backend-operaciones:8000` - Servicio de operaciones
- `backend-pos:8002` - Servicio POS
- `frontend:5173` - Frontend SvelteKit

### Docker Compose

```yaml
nginx:
  image: nginx:alpine
  container_name: api-gateway
  ports:
    - "80:80"
  volumes:
    - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
  depends_on:
    - backend-identidad
    - backend-operaciones
    - backend-pos
    - frontend
  networks:
    - restaurant-network
```

## 🔒 Seguridad

### Headers Inyectados

El gateway inyecta los siguientes headers en todas las solicitudes a servicios protegidos:

- `X-User-ID`: ID del usuario autenticado
- `X-User-Role`: Rol del usuario (admin, cajero, mesero, cocinero)
- `X-User-Email`: Email del usuario

### Endpoint de Validación Interno

El endpoint `/auth/validate` es **internal**, lo que significa que:

- ❌ No es accesible desde el exterior
- ✅ Solo puede ser llamado por Nginx internamente
- ✅ Se usa automáticamente con la directiva `auth_request`

### CORS

El gateway maneja CORS para todos los servicios:

- Permite credenciales (`Access-Control-Allow-Credentials: true`)
- Permite métodos comunes (GET, POST, PUT, DELETE, PATCH, OPTIONS)
- Maneja preflight requests (OPTIONS)

## 📊 Monitoreo

### Health Check

```bash
curl http://localhost/health
# Respuesta: OK
```

### Logs

Los logs de Nginx se encuentran en:

- `/var/log/nginx/access.log` - Logs de acceso
- `/var/log/nginx/error.log` - Logs de errores

Para ver los logs en tiempo real:

```bash
docker logs -f api-gateway
```

## 🚀 Uso

### Desde el Frontend

El frontend debe configurar `USE_GATEWAY = true` en `config.js`:

```javascript
export const USE_GATEWAY = true; // Usar API Gateway
export const API_GATEWAY_URL = 'http://localhost';
```

### Ejemplo de Solicitud

```javascript
// Login (ruta pública)
fetch('http://localhost/api/auth/login', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({
    email: 'admin@restaurant.com',
    password: 'admin123'
  })
});

// Obtener mesas (ruta protegida)
fetch('http://localhost/api/operaciones/mesas', {
  method: 'GET',
  headers: {
    'Authorization': `Bearer ${accessToken}`,
    'Content-Type': 'application/json'
  }
});
```

## 🐛 Troubleshooting

### Error 401 en rutas protegidas

**Problema**: El gateway retorna 401 Unauthorized

**Solución**:
1. Verificar que el token JWT esté en el header `Authorization: Bearer <token>`
2. Verificar que el servicio de identidad esté corriendo (`backend-identidad:8001`)
3. Verificar que el endpoint `/api/auth/validate` funcione correctamente

### Error 502 Bad Gateway

**Problema**: El servicio de destino no está disponible

**Solución**:
1. Verificar que todos los servicios estén corriendo
2. Verificar la configuración de upstream en `nginx.conf`
3. Verificar que los nombres de servicio en Docker Compose coincidan

### Frontend no carga

**Problema**: La aplicación frontend no se muestra

**Solución**:
1. Verificar que el servicio frontend esté corriendo en el puerto 5173
2. Verificar la configuración de proxy en la ruta `/`
3. Verificar que WebSocket esté configurado para HMR (`/ws`)

## 📚 Referencias

- [Nginx Auth Request Module](http://nginx.org/en/docs/http/ngx_http_auth_request_module.html)
- [Nginx Proxy Module](http://nginx.org/en/docs/http/ngx_http_proxy_module.html)
- [Nginx Core Module](http://nginx.org/en/docs/http/ngx_http_core_module.html)
