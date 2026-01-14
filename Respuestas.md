# Tarea sobre Autenticación: Migración de LocalStorage a Cookies

Este repositorio contiene la implementación de un sistema de autenticación JWT, migrado de un almacenamiento en `localStorage` hacia un sistema de **Cookies seguras**.

## Cambios Realizados

Para mejorar la seguridad de la aplicación, se realizaron las siguientes modificaciones:

1.  **Frontend:** Se eliminó la dependencia de `localStorage.setItem()` y `getItem()`. Ahora, el token es gestionado directamente por el navegador mediante cookies.
2.  **Seguridad:** Se implementaron las banderas `HttpOnly` y `SameSite` para mitigar ataques XSS y CSRF.
3.  **Comunicación:** Se configuró el envío de credenciales en las peticiones `fetch` mediante la propiedad `credentials: 'include'`.

---

## Respuestas a las preguntas

### 1. Conceptuales

**¿Qué vulnerabilidades de seguridad previenen las cookies HTTP-only?**
Previenen principalmente el robo de identidad mediante ataques **XSS (Cross-Site Scripting)**. 
* **Analogía:** consideremos que el token es la llave de una casa. Usar `localStorage` es como dejar la llave sobre el tapete de la entrada; cualquier script malicioso que logre entrar a tu página puede verla y robársela. Una **Cookie HTTP-only** es como si la llave estuviera dentro de una caja fuerte empotrada: se puede usar para abrir la puerta (el navegador le envía al servidor), pero nadie puede tocarla ni verla desde JavaScript.

**Importancia del atributo `sameSite: 'strict'` y ataques CSRF**
Un ataque **CSRF** es como si un atacante te engañara para que firmaras un cheque sin que te dieras cuenta mientras estás distraído en otra pestaña. El atributo `strict` evita esto asegurando que el navegador solo envíe la "firma" (la cookie) si la petición se origina exactamente desde tu sitio oficial, bloqueando cualquier intento externo.

**¿Cuándo NO usar cookies?**
No son recomendables en **APIs que serán consumidas por aplicaciones móviles nativas o dispositivos IoT**, ya que estos sistemas no tienen una gestión de cookies tan estandarizada como los navegadores web. En esos casos, el uso de Headers (`Authorization: Bearer`) es más flexible.

### 2. Técnicas

**¿Qué pasa si olvidamos `credentials: 'include'`?**
El servidor rechazará todas las peticiones protegidas. Aunque el usuario esté logueado y la cookie exista, el navegador no la enviará automáticamente en el `fetch`, resultando en errores 401 (No autorizado).

**CORS y `credentials: true`**
Es fundamental porque la **Política de Mismo Origen (SOP)** del navegador prohíbe, por defecto, que un sitio web envíe credenciales a un dominio diferente. Activar esto en el backend es darle permiso explícito al navegador para compartir esa información sensible de forma segura.

**Separación de dominios (Cookies de terceros)**
Si el frontend está en `sitio-a.com` y el backend en `sitio-b.com`, las cookies se vuelven de "terceros". Muchos navegadores modernos las bloquean por privacidad. La solución técnica es usar subdominios del mismo sitio principal (ej. `api.misitio.com` y `web.misitio.com`).

### 3. Casos Prácticos

**Mecanismo de "Recordarme"**
* **Modificación:** Ajustaría el `maxAge` de la cookie para que dure días o semanas (ej. 30 días).
* **Seguridad:** No guardaría el token de acceso principal por tanto tiempo; usaría un "Refresh Token" rotativo para que, si alguien roba la cookie, el daño sea limitado.

**Manejo de expiración (UX)**
Lo ideal es mostrar un aviso al usuario antes de que expire su sesión. Para redirigir al login sin perder el contexto, guardaría la ruta donde estaba el usuario en la URL (ej. `/login?returnTo=/carrito`), para que al volver a entrar, el sistema lo lleve exactamente donde se quedó.

### 4. Debugging

**Error: "Cannot set headers after they are sent"**
Este error ocurre cuando intentas enviar una respuesta (como `res.json()`) después de que ya se envió otra o se intentó modificar la cookie. 
* **Orden correcto:** Siempre se debe ejecutar `res.cookie()` **antes** de `res.json()`.

**Cookies que no se guardan (3 causas):**
1.  **Protocolo inseguro:** Intentar usar la bandera `Secure` en una conexión HTTP normal (localhost sin certificados).
2.  **Dominio mal configurado:** Que el dominio definido en la cookie no coincida con el dominio desde donde se sirve la app.
3.  **Falta de SameSite=None:** En navegadores modernos, si quieres enviar cookies entre sitios distintos, necesitas `SameSite=None` y `Secure` obligatoriamente.

### 5. Arquitectura

**Comparativa: LocalStorage vs Cookies**

| Criterio | LocalStorage | Cookies (HTTP-only) |
| :--- | :--- | :--- |
| **Acceso desde JS** | Sí (Vulnerable a XSS) | No (Seguro contra XSS) |
| **Envío en peticiones** | Manual (Headers) | Automático por Navegador |
| **Capacidad** | ~5MB | ~4KB |
| **Persistencia** | Permanente hasta borrar | Configurable (Expira) |
| **Seguridad CSRF** | Inmune | Vulnerable (Requiere SameSite) |

**Estrategia de Migración en Producción**
Realizaría una **implementación dual**: el backend aceptaría durante un tiempo tanto el Token en el Header como en la Cookie. Una vez confirmado que todos los clientes han actualizado y están recibiendo la cookie correctamente, se eliminaría el soporte para `localStorage`. Para el rollback, mantendría la lógica de los Headers comentada en el código para reactivarla en minutos si algo falla.
