# Diagramas de Secuencia

## Buscar y Localizar parqueos cercanos

Diagrama de secuencia de la funcionalidad "Buscar y localizar parqueos cercanos" en una aplicación móvil para ciclistas

```mermaid
sequenceDiagram
autonumber
participant U as User (Cyclist)
participant APP as BikeParking App (Mobile)
participant AUTH as Auth Service
participant DB as Parking DB (Firestore)
participant MOD as Admin Moderation (Panel/Functions)
participant PUSH as Push Service (FCM)

    %% UC-01: Buscar y localizar parqueos cercanos (con datos de la comunidad)
    U->>APP: Abrir app
    APP->>AUTH: Sign-in (Email/Google/Anon)
    AUTH-->>APP: 200 OK {uid}

    U->>APP: Permitir ubicación
    APP->>APP: Obtener GPS (lat,lng)

    par Consultar parqueos aprobados
        APP->>DB: GET /parkings?near={lat,lng}&status=approved
        DB-->>APP: 200 OK [parkings...]
    and Renderizar mapa
        APP-->>U: Mostrar marcadores y filtros
    end

    alt Aplicar filtros (seguridad/costo/abierto ahora)
        U->>APP: Ajustar filtros
        APP->>DB: GET /parkings?filters=...
        DB-->>APP: 200 OK [filtrados...]
    else Sin filtros
        Note over APP: Mantener resultado inicial
    end

    U->>APP: Tocar un marcador
    APP->>DB: GET /parking/{id}/details
    DB-->>APP: 200 OK {seguridad, costo, horario, fotos, lastUpdate, fuente=comunidad}
    APP-->>U: Ver ficha del parqueo
```

## Navegar hacia un parqueo

Diagrama de secuencia que de la funcionalidad de redirigir al usuario hacia una aplicación de rutas

```mermaid
sequenceDiagram
    autonumber
    participant U as User (Cyclist)
    participant APP as BikeParking App (Mobile)
    participant DB as Parking DB (Firestore)
    participant NAV as External Nav App (Waze/Google/Apple Maps)
    participant BROWSER as Web Browser (fallback)

    %% UC-02 (revisado): Ir al parqueo usando app de navegación externa
    U->>APP: Tap "Cómo llegar"
    APP->>DB: GET /parking/{id} {lat,lng,name}
    DB-->>APP: 200 OK {lat, lng, name}

    APP-->>U: Mostrar opciones de navegación (Waze, Google Maps, Apple Maps, Otros)
    U->>APP: Seleccionar app preferida

    alt Waze instalado
        APP->>NAV: Launch waze://?ll={lat},{lng}&navigate=yes
        NAV-->>U: Abre navegación en Waze
    else Google Maps instalado
        APP->>NAV: Launch comgooglemaps://?daddr={lat},{lng}&directionsmode=bicycling
        NAV-->>U: Abre navegación en Google Maps
    else iOS Apple Maps (por defecto)
        APP->>NAV: Launch http(s)://maps.apple.com/?daddr={lat},{lng}&dirflg=w
        NAV-->>U: Abre navegación en Apple Maps
    else Fallback a navegador web
        APP->>BROWSER: Open https://www.google.com/maps/dir/?api=1&destination={lat},{lng}&travelmode=bicycling
        BROWSER-->>U: Abre rutas en Google Maps Web
    end
```

## Creación de un nuevo parqueo

Diagrama que explica el proceso de creación de un nuevo parqueo

```mermaid
sequenceDiagram
    autonumber
    participant U as User (Cyclist)
    participant APP as BikeParking App (Mobile)
    participant AUTH as Auth Service
    participant DB as Parking DB
    participant MOD as Admin Moderation
    participant PUSH as Push Service

    %% UC-03: Alta de nuevo parqueo por la comunidad (1 por centro comercial)
    U->>APP: Crear parqueo (Centro Comercial X)
    APP->>AUTH: Verificar sesión (uid)
    AUTH-->>APP: OK

    Note over U,APP: Completar formulario
    APP->>DB: POST /parkings {name, location, mall=true, uniqueKey=slug/geo, seguridad, horario, costo, fotos, createdBy=uid, status=pending}
    DB-->>APP: 201 Created {parkingId, status=pending}

    APP-->>U: Confirmación "En revisión"

    %% Moderación y unicidad
    DB-->>MOD: Evento onCreate parkings(status=pending)
    MOD->>DB: Check duplicados (misma área/nombre) y validar contenido
    alt Duplicado o inválido
        MOD->>DB: PATCH /parkings/{id} {status=rejected, reason}
        DB-->>MOD: 200 OK
        APP-->>U: Notificación rechazo con motivo
    else Válido
        MOD->>DB: PATCH /parkings/{id} {status=approved, approvedAt}
        DB-->>MOD: 200 OK
        DB-->>APP: Nuevo parqueo visible en búsquedas
        PUSH-->>U: Notificación: "Parqueo aprobado"
    end

```

## Envío de reseña

Diagrama que muestra el flujo a la hora de que el usuario sube una reseña

```mermaid
sequenceDiagram
    autonumber
    participant U as User (Cyclist)
    participant APP as BikeParking App
    participant AUTH as Auth Service
    participant DB as Parking DB
    participant MOD as Admin Moderation

    %% UC-04: Reseñas y calificaciones (comunidad)
    U->>APP: Abrir ficha → pestaña Reseñas
    APP->>DB: GET /reviews?parkingId={id}&status=approved&limit=20
    DB-->>APP: 200 OK [reseñas...]

    U->>APP: Enviar reseña (rating, texto, fotos)
    APP->>AUTH: Verificar uid y límites
    AUTH-->>APP: OK
    APP->>DB: POST /reviews {parkingId, uid, rating, text, photos, status=pending}
    DB-->>APP: 201 Created

    DB-->>MOD: Evento onCreate reviews(status=pending)
    alt Aprobada
        MOD->>DB: PATCH /reviews/{id} {status=approved}
        DB-->>MOD: 200 OK
        APP-->>U: Reseña publicada
    else Rechazada
        MOD->>DB: PATCH /reviews/{id} {status=rejected, reason}
        DB-->>MOD: 200 OK
        APP-->>U: Notificación de rechazo
    end
```

## Diagrama de secuencia para cambios en parqueos

```mermaid
sequenceDiagram
    autonumber
    participant U as User (Cyclist)
    participant APP as BikeParking App
    participant AUTH as Auth Service
    participant DB as Parking DB
    participant MOD as Admin Moderation

    %% UC-07: Edición comunitaria y control de versiones (soft-edit)
    U->>APP: Proponer edición a ficha (horario, seguridad, costo, foto)
    APP->>AUTH: Verificar uid
    AUTH-->>APP: OK
    APP->>DB: POST /parkingEdits {parkingId, uid, patch, evidence, status=pending}
    DB-->>APP: 201 Created

    DB-->>MOD: Evento onCreate parkingEdits
    alt Aprobada (merge)
        MOD->>DB: PATCH /parkings/{id} {apply:patch, updatedAt, lastEditor=uid}
        DB-->>MOD: 200 OK
        MOD->>DB: PATCH /parkingEdits/{editId} {status=approved}
        DB-->>MOD: 200 OK
        APP-->>U: Cambios publicados
    else Rechazada
        MOD->>DB: PATCH /parkingEdits/{editId} {status=rejected, reason}
        DB-->>MOD: 200 OK
        APP-->>U: Notificación de rechazo
    end
```
