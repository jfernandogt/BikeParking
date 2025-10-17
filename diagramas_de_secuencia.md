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
