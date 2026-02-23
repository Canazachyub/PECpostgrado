# Posgrado PEC - Plataforma Educativa

**Dominio:** https://posgradopec.com
**Repositorio:** https://github.com/Canazachyub/PECpostgrado
**Hosting:** GitHub Pages
**DNS:** Hostinger (posgradopec.com)

---

## Estructura del Sitio

```
posgradopec.com/
├── index.html              → Redirect a inicio.html
├── inicio.html             → Pagina principal (landing page)
├── nosotros.html           → Quienes somos, mision, vision
├── diplomados.html         → Catalogo de 7 diplomados
├── testimonios.html        → Testimonios de egresados
├── contacto.html           → Formulario de contacto + mapa
├── login.html              → Login de estudiantes (API REST)
├── aula-virtual.html       → Pantalla de transicion post-login
├── cursos.html             → Aula Virtual Kanban (API REST + Admin Auth)
├── padlet.html             → Muro de notas digitales
├── au.html                 → Login alternativo (legacy Firebase)
├── CNAME                   → Dominio personalizado GitHub Pages
├── .gitignore              → Excluye .gs y .md de documentacion
└── diplomados-img/         → Imagenes de diplomados
    ├── 1.jpg   (Medicina Legal)
    ├── 3.jpg   (Emergencias Medicas)
    ├── 4.jpg   (Ecografia)
    ├── 6.jpg   (Tesis en Salud)
    └── 7.jpg   (SSOMA)
```

### Archivos locales (NO en GitHub, excluidos por .gitignore)
```
├── AppScript-Login-API.gs              → Codigo backend del Login
├── AppScript-AulaVirtual-API.gs        → Codigo backend del Aula Virtual
├── Codigo loging.md                    → Documentacion del login original
└── Codigo cursos propiamente dichos.md → Documentacion del aula original
```

---

## Diplomados (7)

| # | Nombre | Abreviatura |
|---|--------|-------------|
| 1 | DIPLOMADO MEDICINA LEGAL Y CIENCIAS FORENSES | MLCF |
| 2 | DIPLOMADO EN MEDICINA ALTERNATIVA Y COMPLEMENTARIA | MAC |
| 3 | DIPLOMADO EN EMERGENCIAS Y URGENCIAS MEDICAS | EUM |
| 4 | DIPLOMADO EN ECOGRAFIA GENERAL Y VELOCIMETRIA DOPPLER | EGVD |
| 5 | DIPLOMADO EN REDACCION Y PUBLICACION DE ARTICULOS CIENTIFICOS | RPAC |
| 6 | DIPLOMADO EN ASESORIA Y ELABORACION DE TESIS EN SALUD | AETS |
| 7 | DIPLOMADO EN SEGURIDAD, SALUD OCUPACIONAL Y MEDIO AMBIENTE | SSOMA |

---

## Arquitectura Backend

El backend usa **Google Apps Script** como API REST. No hay servidor propio. Apps Script recibe peticiones HTTP, consulta Google Sheets y devuelve JSON.

### Dos APIs independientes

| Sistema | Apps Script URL | Google Sheet ID |
|---------|----------------|-----------------|
| **Login** | `https://script.google.com/macros/s/AKfycbyw5cBjX3asld5ehIymh95uW_swvXIiz_GHiVTaqwW7PBip1nzB64tIUgeGXYZBRYLD/exec` | `12iiOf8OKDs5fCxc2av5aq4u-FocS-MlYbUgA4DZP40k` |
| **Aula Virtual** | `https://script.google.com/macros/s/AKfycbxwzymlbFFt3Eu2xF_yfZtIuXzqoThpHvLh5R0ICTii4GfYD7QmL3ypc3KfMp_4B9pz1g/exec` | `1RnfjExIWeZKGvFuZDyxn5D-q0XBcf-QhhXjl1SucZ6g` |

### Nota tecnica: CORS con Apps Script
- Se usa `Content-Type: text/plain` en POST (NO `application/json`) para evitar preflight OPTIONS
- El body sigue siendo JSON valido, Apps Script lo parsea con `JSON.parse(e.postData.contents)`
- Deployment con acceso "Cualquier persona"

---

## Flujo de Login

```
Usuario visita login.html
        |
        v
Pagina carga → fetch GET a Login API → action=getDiplomados
        |
        v
Se llena el dropdown con los diplomados disponibles
        |
        v
Usuario escribe su Email + selecciona Diplomado
        |
        v
Click "Acceder" → fetch POST a Login API → {action: "validarAcceso", correo, diplomado}
        |
        v
Apps Script busca en Google Sheets (USUARIOS_ACTIVOS):
   ¿Existe correo + diplomado?
      NO → {success: false, codigo: "USUARIO_NO_ENCONTRADO"}
      SI → ¿Estado ACTIVO?
         NO → {success: false, codigo: "USUARIO_INACTIVO"}
         SI → ¿Fecha no expirada?
            NO → {success: false, codigo: "ACCESO_EXPIRADO"}
            SI → {success: true, nombre, link, diasRestantes}
        |
        v
Frontend recibe JSON:
   Error → Alerta roja con mensaje + boton WhatsApp
   Exito → "Bienvenido, Juan! (25 dias restantes)"
           → Redirect a aula-virtual.html?redirect={url} en 2.5s
        |
        v
aula-virtual.html extrae ?diplomado=N del redirect
        → Redirige a cursos.html?diplomado=N
```

### API Login - Endpoints

**GET** `?action=getDiplomados`
```json
{
  "success": true,
  "diplomados": ["DIPLOMADO MEDICINA LEGAL...", "DIPLOMADO EN SSOMA..."]
}
```

**POST** `{action: "validarAcceso", correo: "email", diplomado: "nombre"}`
```json
{
  "success": true,
  "nombre": "Juan Perez",
  "link": "https://script.google.com/.../exec?diplomado=3",
  "diasRestantes": 25
}
```

---

## Flujo del Aula Virtual

```
cursos.html?diplomado=N
        |
        v
Pagina carga → checkAdminSession() (verifica localStorage)
             → fetch GET a Aula API → action=getDiplomadoInfo&diplomado=N
             → fetch GET a Aula API → action=getAll&diplomado=N
        |
        v
Se carga interfaz Kanban con columnas y tarjetas del diplomado
        |
        v
Segun el ROL del usuario:

[ESTUDIANTE - por defecto]
   - Ver contenido (texto, links, videos, PDFs, imagenes)
   - Buscar contenido (Ctrl+K)
   - Ver estadisticas
   - Marcar materiales como completados (checkbox por tarjeta)
   - Ver progreso en header y modal de resumen por columna
   - NO puede crear/editar/eliminar nada
   - NO puede arrastrar tarjetas
   - Ve icono de candado para login admin

[ADMIN - tras autenticarse]
   - Todo lo del estudiante, mas:
   - Crear/editar/eliminar columnas
   - Crear/editar/eliminar tarjetas
   - Arrastrar y reordenar (drag & drop con SortableJS)
   - Subir archivos (se guardan en Google Drive)
   - Badge "ADMIN" dorado visible en header
```

### Sistema de Autenticacion Admin

```
Admin hace clic en icono candado (header)
        |
        v
Modal pide contrasena
        |
        v
POST a Aula API → {action: "verifyAdmin", password: "xxx"}
        |
        v
Apps Script:
   1. Hashea password con SHA-256
   2. Compara con ADMIN_PASSWORD_HASH en hoja Configuracion
   3. Si coincide → {success: true, adminToken: "TOKEN_SECRETO"}
   4. Si no → {success: false, error: "Contrasena incorrecta"}
        |
        v
Frontend guarda token en localStorage (expira en 24h)
        |
        v
Todos los POST futuros incluyen adminToken automaticamente
        |
        v
Apps Script valida token antes de ejecutar cualquier escritura
   Sin token valido → {success: false, error: "No autorizado"}
```

**Credenciales admin:**
- Contrasena: `admin2026`
- Hash y token se generan ejecutando `CONFIGURAR_ADMIN()` en Apps Script

**Seguridad en dos capas:**
1. **Frontend (UX):** Oculta/muestra controles segun rol. Clase CSS `is-admin` en body.
2. **Backend (seguridad real):** Todas las operaciones POST requieren `adminToken`. Sin token valido = rechazado.

### API Aula Virtual - Endpoints

**GET (publicos, sin autenticacion):**
| Action | Descripcion |
|--------|-------------|
| `getDiplomados` | Lista todos los diplomados |
| `getDiplomadoInfo&diplomado=N` | Info de un diplomado especifico |
| `getAll&diplomado=N` | Todas las columnas y tarjetas |
| `getColumnas&diplomado=N` | Solo columnas |
| `getTarjetas&diplomado=N` | Solo tarjetas |
| `getStats` | Estadisticas del sistema |

**POST (requieren adminToken, excepto verifyAdmin):**
| Action | Descripcion |
|--------|-------------|
| `verifyAdmin` | Verificar contrasena admin (NO requiere token) |
| `createColumna` | Crear nueva columna |
| `updateColumna` | Actualizar columna |
| `deleteColumna` | Eliminar columna (solo si esta vacia) |
| `reorderColumnas` | Reordenar columnas |
| `createTarjeta` | Crear tarjeta (texto, link, video, archivo) |
| `updateTarjeta` | Actualizar tarjeta |
| `deleteTarjeta` | Eliminar tarjeta |
| `moveTarjeta` | Mover tarjeta entre columnas |
| `reorderTarjetas` | Reordenar tarjetas dentro de columna |
| `uploadFile` | Subir archivo (base64 → Google Drive) |
| `deleteFile` | Eliminar archivo de Drive |

### Estructura Google Sheets (Aula Virtual)

Cada diplomado tiene sus propias hojas:
```
Diplomado1_Columnas  → [ID, Titulo, Orden, Color, Activa, FechaCreacion]
Diplomado1_Tarjetas  → [ID, ColumnaID, Tipo, Titulo, Descripcion, Contenido, URL, DriveFileID, ThumbnailURL, Posicion, FechaCreacion, UltimaModificacion, Autor, Etiquetas]
Diplomado2_Columnas
Diplomado2_Tarjetas
...
Diplomado7_Columnas
Diplomado7_Tarjetas
Configuracion        → [Clave, Valor, Descripcion]
  - ADMIN_PASSWORD_HASH  → Hash SHA-256 de la contrasena admin
  - ADMIN_TOKEN          → Token secreto para operaciones POST
  - DRIVE_FOLDER_ID      → ID de la carpeta principal en Drive
  - MAX_FILE_SIZE_MB     → Tamano maximo de archivo (50MB)
  - ALLOWED_FILE_TYPES   → Tipos MIME permitidos
Cache                → [Clave, Valor, Timestamp]
Logs                 → [Timestamp, Usuario, Accion, Detalles, UserAgent]
```

### Estructura Google Drive
```
Aula-Virtual-Files/
├── Diplomado-1-MLCF/
│   ├── documentos/
│   ├── imagenes/
│   ├── videos/
│   └── thumbnails/
├── Diplomado-2-MAC/
│   └── ...
└── Diplomado-7-SSOMA/
    └── ...
```

### Tipos de tarjeta soportados
| Tipo | Descripcion | Color borde izquierdo |
|------|-------------|----------------------|
| **texto** | Contenido de texto libre | Verde (#059669) |
| **link** | URL con preview automatico (og:title, og:image) | Azul (#3B82F6) |
| **video** | YouTube, Vimeo, Google Drive (embed automatico) | Rojo (#E32B2B) |
| **pdf** | Archivo PDF con thumbnail de Drive | Rojo oscuro (#991B1B) |
| **imagen** | Imagen con thumbnail | Violeta (#8B5CF6) |
| **archivo** | Cualquier archivo permitido | Gris (#6B7280) |

### Diseño Visual del Aula Virtual
- **Header:** Gradiente oscuro (#1B3A57 → #0F2438), compacto 64px
- **Paleta:** Rojo PEC (#E32B2B), Azul oscuro (#1B3A57), Dorado acento (#FFC107)
- **Fuentes:** Montserrat (titulos), Inter (cuerpo)
- **Tarjetas:** Borde izquierdo de color segun tipo, sombras suaves
- **Columnas:** Bordes redondeados, hover con elevacion, estado vacio con icono
- **Loading:** Gradiente oscuro con spinner y animacion de pulso
- **Modales:** Backdrop blur, sombras profundas
- **Fondo:** Patron sutil de puntos
- **Notificaciones:** Gradientes (verde exito, rojo error)

### Sistema de Progreso del Estudiante

Permite a los estudiantes trackear su avance en el material del diplomado. Usa **localStorage** (sin backend).

```
Estudiante entra a cursos.html?diplomado=N
        |
        v
loadProgress() lee localStorage (clave: pec_progress_N)
        |
        v
Cada tarjeta muestra checkbox circular a la izquierda de la fecha
Header muestra barra de progreso "X/Y completados"
        |
        v
Click en checkbox → toggleProgress(cardId)
        → Checkbox se pone verde con animacion
        → Tarjeta se marca visualmente (borde verde, fondo verde sutil)
        → Barra en header se actualiza
        → saveProgress() guarda en localStorage
        |
        v
Click en barra de progreso del header
        → Abre modal con resumen detallado:
           - Porcentaje total con barra grande
           - Progreso por columna (nombre + barra + "X/Y")
           - Mensaje motivacional al 50% y 100%
           - Boton "Reiniciar progreso"
```

**Almacenamiento:**
- Clave localStorage: `pec_progress_[diplomadoId]` (independiente por diplomado)
- Formato: `{ "card-uuid": { done: true, at: timestamp } }`
- Persiste entre sesiones en el mismo navegador/dispositivo

**Elementos visuales:**
| Elemento | Descripcion |
|----------|-------------|
| **Checkbox** | Circulo 22px, gris sin completar, verde con check al completar |
| **Tarjeta completada** | Borde izquierdo verde, fondo verde sutil, titulo verde, contenido 75% opacidad |
| **Barra header** | Mini barra 80px + texto "X/Y", visible solo para estudiantes |
| **Modal progreso** | Porcentaje grande, barra total, desglose por columna, boton reiniciar |

**Nota:** El progreso NO se ve en modo admin (los checkboxes se reemplazan por botones eliminar).

---

## Tecnologias

| Componente | Tecnologia |
|------------|-----------|
| Frontend | HTML5, CSS3, JavaScript vanilla |
| Estilos | Tailwind CSS (CDN), CSS custom |
| Iconos | Font Awesome 6.4 |
| Fuentes | Montserrat, Inter |
| Drag & Drop | SortableJS (solo admin) |
| Seguridad XSS | DOMPurify |
| Auth Admin | SHA-256 hash + token en localStorage (24h) |
| Progreso Estudiante | localStorage por diplomado (sin backend) |
| Backend | Google Apps Script |
| Base de datos | Google Sheets |
| Almacenamiento | Google Drive |
| Formulario contacto | Formspree |
| Hosting | GitHub Pages |
| Dominio | Hostinger (posgradopec.com) |

---

## Datos de Contacto (en el sitio)

- **Telefono:** +51 958 146 863
- **Email:** posgradopec@gmail.com
- **WhatsApp:** wa.me/51958146863
- **Direccion:** Av. El Sol N° 1109 - 2do Piso, Puno, Peru
- **Horario:** Lunes - Viernes: 8:00 - 15:00
- **Facebook:** facebook.com/posgradopec
- **YouTube:** youtube.com/@POSGRADOPEC
- **Instagram:** instagram.com/posgrado.pec

---

## Despliegue

### Para actualizar el sitio:
```bash
cd "c:\Users\HAPPY TEC\Desktop\PEC"
git add -A
git commit -m "Descripcion del cambio"
git push origin main
```
GitHub Pages redespliega automaticamente en ~1 minuto.

### Para modificar el backend Login:
1. Abrir script.google.com (proyecto del Login)
2. Editar el codigo
3. Deploy > Manage deployments > Editar > Nueva version > Guardar

### Para modificar el backend Aula Virtual:
1. Abrir script.google.com (proyecto del Aula Virtual)
2. Pegar el codigo de `AppScript-AulaVirtual-API.gs`
3. Deploy > Manage deployments > Editar > Nueva version > Guardar

### Para inicializar/reparar el Aula Virtual:
1. En Apps Script, ejecutar la funcion `CONFIGURAR_AULA_VIRTUAL`
2. Esto crea las hojas, carpetas en Drive y configuracion inicial

### Para configurar/resetear admin:
1. En Apps Script, ejecutar la funcion `CONFIGURAR_ADMIN`
2. Esto genera el hash SHA-256 de "admin2026" y un token aleatorio
3. Ambos se guardan en la hoja Configuracion
4. Hacer nuevo deployment despues de cambios en el codigo

### Para cambiar la contrasena admin:
1. Editar la linea `const password = 'admin2026';` en `CONFIGURAR_ADMIN()`
2. Ejecutar `CONFIGURAR_ADMIN()` de nuevo
3. Hacer nuevo deployment

---

## DNS (Hostinger)

| Tipo | Nombre | Contenido |
|------|--------|-----------|
| CNAME | www | canazachyub.github.io |
| A | @ | 185.199.108.153 |
| A | @ | 185.199.109.153 |
| A | @ | 185.199.110.153 |
| A | @ | 185.199.111.153 |
