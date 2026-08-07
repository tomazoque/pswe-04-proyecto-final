# Guía de implementación para el arquitecto de C4 Code

## Propósito y alcance

Este documento entrega el contexto arquitectónico vigente para completar o ajustar las páginas `C4 Code` de Movix.

- L1, L2 y L3 están aprobados por Architecture III.
- Las páginas `C4 Code` pueden detallar clases, módulos, interfaces y paquetes, pero no deben alterar los límites, flujos ni responsabilidades definidos en L2 y L3 sin una revisión arquitectónica.
- Los contenidos existentes de Code no han sido revisados ni modificados como parte de esta aprobación. Solo se actualizaron los nombres de pestañas y los enlaces de navegación.

## Navegación del archivo Draw.io

Las pestañas siguen esta jerarquía:

```text
C4 Context
  └── C4 Container
        ├── C4 Component Web Application
        │     └── C4 Code Web Application
        ├── C4 Component Mobile Application
        │     └── C4 Code Mobile Application
        ├── C4 Component API Gateway
        │     └── C4 Code API Gateway
        ├── C4 Component Main Service
        │     └── C4 Code Main Service
        ├── C4 Component Operations Service
        │     └── C4 Code Operations Service
        └── C4 Component Interaction Service
              └── C4 Code Interaction Service
```

Los enlaces internos son válidos. En cada página `C4 Component ...`, el rectángulo blanco grande del contenedor enlaza a la página `C4 Code ...` correspondiente.

## Flujo arquitectónico vigente

```text
Web Application ─┐
                 ├──> API Gateway ───> Main Service
Mobile Application┘                    └──> Operations Service

API Gateway ───> Proveedor de Identidad (validación JWT por OIDC/JWKS)
Main Service ───> Proveedor de Identidad (gestión de identidades y roles)

Operations Service ───> Bus de Eventos (Kafka) ───> Interaction Service
Interaction Service ───> Servicio de Notificaciones
```

Regla principal: Web Application y Mobile Application se comunican con el backend por API Gateway. No se debe introducir una llamada directa desde los frontends hacia el Proveedor de Identidad.

## Contratos por contenedor

### Web Application

- Tecnología: React + TypeScript.
- Componentes L3: configuración de rutas y choferes, administración de usuarios, dashboard operativo, gestión de incidentes, gestor de sesión/autenticación y cliente HTTP.
- Todo acceso al backend se encapsula en `Cliente API (HTTP)` hacia API Gateway.
- El gestor de sesión entrega la autenticación a través de API Gateway; no integra directamente con el proveedor de identidad.
- No agregar responsabilidad de vehículos/flota a este contenedor sin actualizar L2/L3.

### Mobile Application

- Tecnología: React Native.
- Componentes L3: ejecución de ruta, reporte de incidentes, seguimiento/notificaciones, solicitudes especiales, sincronización local y cliente HTTP.
- `Gestor de Sincronización Local` usa WatermelonDB para almacenamiento local y sincronización al recuperar conectividad.
- El cliente HTTP se comunica exclusivamente con API Gateway.

### API Gateway

- Tecnología: AWS API Gateway.
- Flujo: autorización JWT → validación de solicitud → planes/cuotas → enrutamiento → integración backend.
- El autorizador valida JWT contra el Proveedor de Identidad mediante OIDC/JWKS y propaga los claims/contexto del operador.
- Solo enruta solicitudes hacia Main Service y Operations Service.
- Interaction Service es dirigido por eventos; no agregarle una ruta síncrona desde API Gateway sin modificar L2/L3.

### Main Service

- Tecnología: Node.js microservice + PostgreSQL.
- Entrada: `API REST / Capa de Aplicación` recibe solicitudes autenticadas desde API Gateway.
- Componentes de dominio: operadores, usuarios y roles, clientes/pasajeros, choferes, perfiles médicos/veterinarios y configuración.
- Persistencia: los módulos de dominio usan `Capa de Acceso a Datos (Repository)`, que lee/escribe en la base de datos del servicio.
- Integración externa: `Gestión de Usuarios y Roles` gestiona identidades y roles con el Proveedor de Identidad mediante REST API.
- La elección concreta del proveedor (y sus capacidades de provisión/roles) es una decisión de implementación posterior.
- No publicar eventos desde Main Service ni introducir un Outbox en este servicio: L2 no declara ese flujo.
- No incorporar vehículos/flota como responsabilidad de Main Service sin revisión arquitectónica.

### Operations Service

- Tecnología: Node.js microservice + PostgreSQL.
- Entrada: `API REST / Capa de Aplicación` desde API Gateway.
- Componentes de dominio: planificación de rutas, GPS en tiempo real, abordajes/entregas, incidentes y solicitudes especiales.
- Usa el servicio externo de mapas/geolocalización para geocodificación y cálculo de rutas.
- Persistencia mediante Repository hacia la base de datos de Operaciones.
- `Publicador de Eventos` publica eventos de abordaje, entrega, solicitud e incidente al Bus de Eventos Kafka.
- Mantener este flujo asincrónico: Operations Service → Kafka → Interaction Service.

### Interaction Service

- Tecnología: Node.js microservice + PostgreSQL para bitácora de auditoría.
- `Consumidor de Eventos` consume eventos desde Kafka.
- `Orquestador de Notificaciones` determina canal y contenido.
- `Adaptador de Integración Externa` envía Push, SMS o correo mediante el Servicio de Notificaciones.
- `Registro de Estado de Envío` actualiza estado y escribe la bitácora de auditoría.
- No asumir que API Gateway invoca este servicio: su entrada actual es el Bus de Eventos.

## Reglas para los diagramas C4 Code

- Usar la página Code para detallar la implementación del contenedor de su página Component asociada; no para agregar nuevos contenedores.
- Conservar los nombres y responsabilidades de los componentes L3 como referencia para paquetes, módulos, clases o interfaces.
- Mostrar dependencias de implementación relevantes: controladores/handlers, servicios de aplicación, repositorios, adaptadores, consumidores/publicadores y clientes externos.
- Mantener las direcciones de dependencia alineadas con L3. Por ejemplo, el Repository depende de PostgreSQL; los módulos de dominio usan el Repository.
- Etiquetar las relaciones con intención concreta; evitar términos ambiguos como `usa` cuando pueda indicarse `lee/escribe`, `publica eventos`, `valida JWT` o `envía notificación`.
- No introducir llamadas frontend → Proveedor de Identidad, rutas API Gateway → Interaction Service, ni publicación de eventos desde Main Service sin una revisión de L2 y L3.

## Cambios recientes que afectan Code

1. Se consolidó API Gateway como único punto de entrada de Web y Mobile.
2. La validación JWT quedó explícita en API Gateway; Main Service conserva la gestión de identidades y roles.
3. Se eliminó de Main Service la publicación de eventos/Outbox. La publicación de eventos pertenece a Operations Service.
4. Se alineó la persistencia por servicio: Main, Operations e Interaction tienen su propia base de datos; Mobile mantiene persistencia local mediante WatermelonDB.
5. Se corrigió la relación de Main Service: Operadores y Perfiles Médicos/Veterinarios acceden independientemente al Repository; no existe una dependencia directa entre esos módulos.
6. Se añadieron enlaces de navegación Container → Component y Component → Code, además de la nomenclatura uniforme de pestañas.

## Checklist antes de aprobar un cambio de Code

- [ ] ¿El cambio permanece dentro de la responsabilidad del contenedor actual?
- [ ] ¿Respeta el flujo Web/Mobile → API Gateway?
- [ ] ¿Respeta que Interaction Service consume eventos, en lugar de ser invocado por API Gateway?
- [ ] ¿La persistencia queda en la base de datos correcta del servicio?
- [ ] ¿La comunicación con servicios externos se hace mediante el componente/adaptador definido en L3?
- [ ] ¿Un nuevo evento es publicado por Operations Service y consumido por Interaction Service?
- [ ] ¿Un cambio de roles/identidad considera la capacidad real del proveedor seleccionado?
- [ ] ¿El detalle Code sigue siendo consistente con su página Component y con C4 Container?
