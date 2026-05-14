# PlOrDiCo

Gestor de tareas personal. Reemplaza Microsoft To Do con acceso multi-dispositivo, construido como un único archivo `index.html` (CSS y JS inline) sobre Firebase (Firestore + Google Auth).

**URL:** https://enmarman.github.io/PlOrDiCo  
**Proyecto Firebase:** `plordico-8f546`  
**GitHub:** ENMarMan

---

## Funcionalidades actuales

- Login con Google (signInWithPopup)
- CRUD completo de tareas con listeners en tiempo real (Firestore)
- Campos por tarea: título, descripción, estado (Pendiente / En curso / Completada), prioridad (Alta/Baja), tipo (Estratégica/Operacional), fecha de vencimiento, etiquetas, recurrencia, dependencias
- Recurrencia al completar: días hábiles, cada N días, semanal, mensual, anual (con fecha de fin opcional)
- Checklists por tarea
- **Pestaña Lista:** panel lateral derecho fijo (500 px) para edición; se cierra automáticamente al crear una tarea
- **Pestaña Matriz de Eisenhower:** barra de búsqueda, sidebar de filtros colapsable, drag & drop entre cuadrantes (actualiza `priority` + `type` en Firestore)
- **Pestaña Etiquetas:** renombrado inline y eliminación masiva
- Menú contextual con clic derecho en tarjetas (Lista y Matriz): cambio de estado/prioridad/tipo, selector de fecha, gestión de etiquetas, duplicar, marcar como hoy
- Índice compuesto en Firestore: `uid` (asc) + `createdAt` (desc)

---

## Arquitectura

| Elemento | Detalle |
|---|---|
| Estructura | Archivo único `index.html` (sin build tools, sin bundler) |
| Autenticación | Firebase Auth — Google, modo compat CDN |
| Base de datos | Cloud Firestore |
| Hosting | GitHub Pages (reemplazo manual de `index.html` vía interfaz web de GitHub) |
| Tema visual | Oscuro, acento ámbar, Space Mono (etiquetas/identificadores), DM Sans (cuerpo) |

---

## Despliegue

Reemplazar `index.html` directamente desde la interfaz web de GitHub. No hay pipeline de CI/CD.

---

## Diagnóstico de errores comunes

### "Acceso denegado" después del login

**Causa más probable:** las reglas de seguridad de Firestore expiraron o fueron modificadas.

**Síntoma en consola del navegador:**
```
FirebaseError: [code=permission-denied]: Missing or insufficient permissions.
```

**Solución:**  
1. Ir a [Firebase Console](https://console.firebase.google.com/) → proyecto `plordico-8f546` → Firestore Database → pestaña **Rules**
2. Verificar que las reglas no tengan fecha de vencimiento (`request.time < timestamp.date(...)`)
3. Publicar las siguientes reglas permanentes:

```js
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /tasks/{taskId} {
      allow read, write: if request.auth != null
                         && request.auth.uid == resource.data.uid;
      allow create: if request.auth != null
                    && request.auth.uid == request.resource.data.uid;
    }
  }
}
```

4. Hacer clic en **Publish** y recargar la app.

> **Nota:** los errores de "Tracking Prevention" y "Cross-Origin-Opener-Policy" que aparecen en consola son ruido del navegador (Edge/Chrome con protección de rastreo activada). No causan el bloqueo de acceso y pueden ignorarse.

---

### Login no abre el popup de Google

**Causa probable:** el dominio no está en la lista de dominios autorizados de Firebase Auth.

**Solución:**  
Firebase Console → Authentication → Settings → **Authorized domains** → agregar el dominio correspondiente.

---

### Tareas que no cargan (sin error de permisos)

Verificar que el índice compuesto esté creado en Firestore:  
Firebase Console → Firestore → **Indexes** → confirmar índice compuesto `uid` (asc) + `createdAt` (desc) en la colección `tasks`.

---

## Modelo de datos — tarea

```js
{
  uid: string,           // UID del usuario autenticado
  title: string,
  description: string,
  status: 'Pendiente' | 'En curso' | 'Completada',
  priority: 'High' | 'Low',
  type: 'Strategic' | 'Operational',
  dueDate: string,       // ISO 8601
  tags: string[],
  recurrence: object,    // { type, interval, endDate, ... }
  dependencies: string[], // IDs de otras tareas
  checklist: object[],
  reminder: {            // pendiente de implementación
    enabled: boolean,
    datetime: string,
    snoozedUntil: string
  },
  createdAt: Timestamp
}
```

---

## Roadmap

1. **Sistema de recordatorios** — popup in-app con snooze/confirmar/cancelar; campo `reminder` ya definido en el modelo
2. **Vista de calendario mensual** — nueva pestaña (entre Matriz y Gantt) con tareas por `dueDate` y ocurrencias de recurrencia calculadas
3. **Pestaña Gantt**
4. **Pestaña diagrama de dependencias**

---

## Convenciones de desarrollo

- Todo el CSS y JS van inline en `index.html`. Sin archivos separados, sin bundler.
- Antes de entregar cualquier versión nueva, validar la sintaxis JS con `node --check index.html` (o equivalente). Un error de sintaxis silencioso bloquea toda ejecución de scripts, incluyendo el login.
- La pestaña Matriz usa el modal original para edición. La pestaña Lista usa el panel lateral fijo. Esta diferencia es intencional.
- El menú contextual usa `useCapture: true` para interceptar el evento antes de que los elementos hijos lo consuman.
- El panel lateral fijo usa `position: fixed` con offset explícito (topbar 52 px + nav 42 px = 94 px). La lista compensa con `paddingRight` dinámico.
