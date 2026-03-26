# Sistema de Mensajería P2P

> Planteamiento formal del sistema — v1.0

---

## Tabla de Contenidos

1. [Visión General](#1-visión-general)
2. [Modelo Arquitectónico](#2-modelo-arquitectónico)
3. [Descomposición de Componentes](#3-descomposición-de-componentes)
   - [3.1 Mapa de Responsabilidades](#31-mapa-de-responsabilidades)
   - [3.2 Interfaces Comunes](#32-interfaces-comunes)
4. [Modelo de Mensaje](#4-modelo-de-mensaje)
5. [Enrutamiento con Pares Intermedios](#5-enrutamiento-con-pares-intermedios)
   - [5.1 Tabla de Enrutamiento Distribuida](#51-tabla-de-enrutamiento-distribuida)
   - [5.2 Route Discovery con visited_set](#52-route-discovery-con-visited_set)
   - [5.3 Frames en Tránsito](#53-frames-en-tránsito)
   - [5.4 Lógica del MessageRouter](#54-lógica-del-messagerouter)
6. [Control de Bucles](#6-control-de-bucles)
   - [6.1 Decisión de diseño: visited_set vs TTL](#61-decisión-de-diseño-visited_set-vs-ttl)
   - [6.2 Los tres mecanismos combinados](#62-los-tres-mecanismos-combinados)
   - [6.3 RouteRequestCache](#63-routerequestcache)
7. [Ciclo de Vida de una Sesión P2P](#7-ciclo-de-vida-de-una-sesión-p2p)
8. [Transferencia de Archivos](#8-transferencia-de-archivos)
9. [API Gateway](#9-api-gateway)
   - [9.1 Endpoints](#91-endpoints)
   - [9.2 Separación de capas WebSocket](#92-separación-de-capas-websocket)
10. [Estructura de Módulos](#10-estructura-de-módulos)
11. [Decisiones de Diseño](#11-decisiones-de-diseño)

---

## 1. Visión General

El sistema es una plataforma de mensajería punto a punto donde cada instancia del software actúa simultáneamente como **cliente y servidor** (arquitectura simétrica). FastAPI expone el núcleo lógico al frontend React, mientras que el transporte entre pares se realiza íntegramente sobre WebSockets.

El frontend nunca establece conexiones directas con pares remotos. Todo pasa por el núcleo local, lo que centraliza el control de acceso, el registro de eventos y la gestión de cifrado en un único punto por instancia.

---

## 2. Modelo Arquitectónico

```
┌─────────────────────────────────────────────────────────────┐
│                        INSTANCIA A                          │
│                                                             │
│  ┌──────────┐     ┌─────────────┐     ┌─────────────────┐  │
│  │ Frontend │────▶│  FastAPI    │────▶│   Core Engine   │  │
│  │  React   │◀────│  HTTP/WS    │◀────│                 │  │
│  └──────────┘     └─────────────┘     └────────┬────────┘  │
│                                                │            │
└────────────────────────────────────────────────┼────────────┘
                                                 │ WebSocket
                                    ┌────────────▼────────────┐
                                    │       RED P2P           │
                                    └────────────┬────────────┘
                                                 │ WebSocket
┌────────────────────────────────────────────────┼────────────┐
│                        INSTANCIA B             │            │
│                                                │            │
│  ┌──────────┐     ┌─────────────┐     ┌────────▼────────┐  │
│  │ Frontend │────▶│  FastAPI    │────▶│   Core Engine   │  │
│  │  React   │◀────│  HTTP/WS    │◀────│                 │  │
│  └──────────┘     └─────────────┘     └─────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Descomposición de Componentes

### 3.1 Mapa de Responsabilidades

| Componente | Capa | Responsabilidad única |
|---|---|---|
| `PeerConnector` | Red | Establecer y mantener conexiones WS con pares remotos |
| `MessageRouter` | Core | Decidir a qué par entregar un frame; gestiona cola de espera de rutas pendientes |
| `RoutingTable` | Core | Tabla local `dest_id → next_hop_id` con número de saltos y timestamp |
| `RouteDiscovery` | Core | Emisión y recepción de `route_request` / `route_reply` usando `visited_set` |
| `RouteRequestCache` | Core | Deduplicación de `rreq_id` con expiración temporal |
| `MessageSerializer` | Core | Codificar/decodificar `NetworkFrame` a formato de red (JSON envolvente + payload binario) |
| `FileTransferManager` | Core | Fragmentar, enviar, reensamblar y verificar archivos (chunks de 64 KB, hash SHA-256) |
| `SessionManager` | Core | Ciclo de vida de sesiones: handshake, identidad, expiración |
| `ConversationStore` | Persistencia | Historial de mensajes y metadatos de archivos (SQLite vía aiosqlite) |
| `EventBus` | Core | Canal interno pub/sub que desacopla los módulos entre sí |
| `APIGateway` | Exposición | Endpoints FastAPI HTTP + WebSocket que conectan el Core al Frontend |
| `Frontend` | Presentación | Cliente React que consume únicamente la `APIGateway` local |

### 3.2 Interfaces Comunes

Todos los módulos se comunican a través de contratos formales. Ningún módulo depende de la implementación concreta de otro.

```python
# Toda unidad que viaja por la red
class NetworkFrame(Protocol):
    frame_id:    str
    frame_type:  Literal["message", "file_chunk", "control"]
    sender_id:   str          # origen real
    receiver_id: str          # destino final
    route:       list[str]    # ruta completa, ej. ["A", "C", "D"]
    hop_index:   int          # posición actual del frame en route
    timestamp:   float
    payload:     bytes

# Frame de descubrimiento de ruta
class RouteRequestFrame:
    rreq_id:   str            # UUID único por búsqueda
    origin_id: str
    target_id: str
    visited:   set[str]       # nodos que ya procesaron este request
    path:      list[str]      # ruta acumulada (para construir route_reply)

# Todo módulo que emite o recibe eventos internos
class EventSubscriber(Protocol):
    def on_event(self, event_type: str, data: dict) -> None: ...

# Todo módulo de almacenamiento
class StoreAdapter(Protocol):
    def save(self, key: str, value: Any) -> None: ...
    def load(self, key: str) -> Any: ...
    def query(self, filters: dict) -> list: ...

# Todo gestor de transporte externo
class TransportHandler(Protocol):
    async def connect(self, uri: str) -> None: ...
    async def send(self, frame: NetworkFrame) -> None: ...
    async def disconnect(self, peer_id: str) -> None: ...
    def on_receive(self, callback: Callable) -> None: ...
```

---

## 4. Modelo de Mensaje

Cada unidad de comunicación tiene dos partes: un **sobre JSON** y un **payload** cuyo esquema varía según el tipo de frame.

```
┌─────────────────────────── NetworkFrame ──────────────────────────┐
│  ENVELOPE (JSON)                                                   │
│  ┌────────────┬────────────┬──────────┬──────────┬─────────────┐  │
│  │ frame_id   │ frame_type │sender_id │recv_id   │ timestamp   │  │
│  └────────────┴────────────┴──────────┴──────────┴─────────────┘  │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  route: ["A", "C", "D"]          hop_index: 1               │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  PAYLOAD (bytes, interpretado según frame_type)                    │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ "message"    → { text: str, reply_to?: str }               │  │
│  │ "file_chunk" → { file_id, chunk_index, total, data: b64 }  │  │
│  │ "control"    → { action: str, ...campos específicos }      │  │
│  └─────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────┘
```

**Acciones de control disponibles:**

| Acción | Dirección | Descripción |
|---|---|---|
| `hello` | peer ↔ peer | Handshake de sesión |
| `bye` | peer → peer | Cierre de sesión |
| `ack` | receptor → emisor | Confirmación de frame recibido |
| `route_request` | flooding | Búsqueda de ruta hacia un destino |
| `route_reply` | destino → origen | Respuesta con ruta encontrada |
| `route_error` | origen → frontend | Destino inalcanzable (timeout) |
| `file_offer` | emisor → receptor | Propuesta de transferencia de archivo |
| `file_accept` | receptor → emisor | Aceptación de la transferencia |
| `file_complete` | receptor → emisor | Confirmación de reensamblaje exitoso |
| `error` | cualquiera | Error genérico de protocolo |

---

## 5. Enrutamiento con Pares Intermedios

En una red P2P real, no todos los nodos se conocen directamente. El nodo A puede conocer a B y C, pero querer enviar un mensaje a D, que solo conoce C. El sistema debe descubrir rutas y reenviar frames a través de intermediarios sin que A tenga que conocer la topología completa.

### 5.1 Tabla de Enrutamiento Distribuida

Cada nodo mantiene una tabla de enrutamiento local con el **siguiente salto** hacia cada destino conocido:

| `dest_id` | `next_hop_id` | `hops` | `last_seen` |
|---|---|---|---|
| `peer_B` | `peer_B` | 1 | t₀ |
| `peer_C` | `peer_C` | 1 | t₁ |
| `peer_D` | `peer_C` | 2 | t₂ |

El campo clave es `next_hop_id`: A no necesita conocer la topología completa hacia D, solo necesita saber que debe enviar el frame a C, y C sabrá qué hacer con él.

### 5.2 Route Discovery con `visited_set`

Cuando A quiere enviar a D y no tiene ruta registrada, emite un `route_request`:

```
control {
  action:    "route_request",
  origin_id: "A",
  target_id: "D",
  rreq_id:   "<uuid>",
  visited:   ["A"],
  path:      ["A"]
}
```

Cada nodo que recibe el frame ejecuta la siguiente lógica:

```
recibo route_request
    │
    ├── ¿mi ID está en visited?
    │       └── SÍ → descartar (ya procesado por otra ruta)
    │
    ├── agregar mi ID a visited y a path
    │
    ├── ¿soy el target?
    │       └── SÍ → emitir route_reply con path acumulado
    │
    └── vecinos_pendientes = mis_vecinos - visited
            ├── vacío   → flooding terminó aquí (red completamente cubierta)
            └── no vacío → reenviar a cada vecino en vecinos_pendientes
```

Al recibir el `route_reply`, cada nodo en el camino de regreso registra la ruta en su tabla local. Todos los intermediarios aprenden, no solo el origen.

### 5.3 Frames en Tránsito

Una vez conocida la ruta, el frame viaja salto a salto usando los campos `route` y `hop_index`:

```
A ──[hop_index=0]──▶ C ──[hop_index=1]──▶ D
```

Cada nodo intermedio incrementa `hop_index` y reenvía al siguiente nodo en `route`. Un frame cuyo `route[hop_index]` no coincide con el ID del nodo receptor es descartado silenciosamente.

### 5.4 Lógica del MessageRouter

```
recibo frame entrante
    │
    ├── ¿soy el receiver_id final?
    │       └── SÍ → entregar al EventBus local (mensaje entregado)
    │
    └── NO → ¿tengo ruta en RoutingTable?
                ├── SÍ → reenviar al next_hop
                └── NO → iniciar RouteDiscovery
                          encolar frame hasta recibir route_reply
                          (o descartar si llega route_error)
```

---

## 6. Control de Bucles

### 6.1 Decisión de Diseño: `visited_set` vs TTL

El mecanismo TTL fue descartado porque introduce un **universo local** implícito: con `TTL=n`, un nodo solo puede comunicarse con pares a distancia máxima n saltos, federando la red de forma no intencionada.

El `visited_set` reemplaza al TTL como mecanismo de terminación. En lugar de contar saltos hacia abajo, el frame carga el conjunto de nodos que ya lo procesaron. Cada nodo verifica si todos sus vecinos están en ese conjunto antes de reenviar. Si no quedan vecinos nuevos, el frame muere naturalmente sin restricción de profundidad.

La red se cubre **exactamente una vez por nodo**, sin importar su tamaño ni su topología.

> **Nota sobre escala:** el flooding con `visited_set` tiene costo O(N) en el peor caso (red donde el destino no existe). Para escala universitaria o de laboratorio es perfectamente aceptable. A mayor escala, el paso siguiente sería adoptar una DHT (ej. Kademlia) con lookup O(log N).

### 6.2 Los Tres Mecanismos Combinados

| Mecanismo | Qué evita | Dónde vive |
|---|---|---|
| `visited_set` en el frame | Reenvío a nodos ya cubiertos; cada nodo procesa el request exactamente una vez por búsqueda | `RouteRequestFrame.visited` |
| `rreq_id` cache por nodo | Un nodo procesa el mismo `rreq_id` máximo una vez aunque llegue por múltiples rutas simultáneas | `RouteRequestCache` |
| Timeout + `route_error` | Espera indefinida en el origen cuando el destino no existe en la red | `MessageRouter` (timer interno) |

Los tres mecanismos son complementarios y ninguno es redundante.

### 6.3 RouteRequestCache

```python
class RouteRequestCache:
    _seen: dict[str, float]  # rreq_id → timestamp de primera recepción
    TTL_SECONDS = 30         # expiración para liberar memoria en nodos de larga ejecución

    def is_duplicate(self, rreq_id: str) -> bool:
        self._evict_expired()
        return rreq_id in self._seen

    def mark_seen(self, rreq_id: str) -> None:
        self._seen[rreq_id] = time.time()
```

**Ciclo de vida de un Route Request en el origen:**

```
emit route_request ──▶ iniciar timer (ej. 8 s)
                            │
                ┌───────────┴───────────┐
          route_reply               timeout
          recibido                  expirado
                │                       │
        registrar ruta          emitir route_error
        liberar cola            descartar cola
        continuar envío         notificar frontend
                                → peer_unreachable
```

---

## 7. Ciclo de Vida de una Sesión P2P

```
A (iniciador)                               B (receptor)
     │                                           │
     │── WS connect ──────────────────────────▶ │
     │── control {action: "hello",              │
     │            peer_id, display_name} ──────▶│
     │                                           │── validar / registrar sesión
     │◀─ control {action: "hello", ...} ─────── │
     │                                           │
     │   ═══════════════ SESIÓN ACTIVA ═════════│
     │                                           │
     │── message / file_chunk ────────────────▶ │
     │◀─ control {action: "ack", frame_id} ──── │
     │                                           │
     │── control {action: "bye"} ─────────────▶ │
     │   [WS close]                              │
```

---

## 8. Transferencia de Archivos

El `FileTransferManager` fragmenta el archivo en chunks de **64 KB** por defecto, los envía secuencialmente como frames `file_chunk`, y el receptor los reensambla verificando un hash **SHA-256** del archivo completo.

```
Emisor                                          Receptor
  │                                                 │
  │── control {action: "file_offer",               │
  │            file_id, name, size, chunks} ──────▶│
  │                                                 │
  │◀─ control {action: "file_accept", file_id} ────│
  │                                                 │
  │── file_chunk {index: 0, data: ...} ───────────▶│
  │── file_chunk {index: 1, data: ...} ───────────▶│
  │              ...                                │
  │── file_chunk {index: N, data: ...} ───────────▶│
  │                                                 │── reensamblar chunks
  │                                                 │── verificar SHA-256
  │◀─ control {action: "file_complete",             │
  │            file_id, checksum_ok: true} ─────────│
```

Los chunks viajan como frames ordinarios a través de la tabla de enrutamiento. Si el receptor no es un par directo, cada chunk sigue la ruta establecida por el `RouteDiscovery` de la misma forma que cualquier otro frame.

---

## 9. API Gateway

### 9.1 Endpoints

| Endpoint | Tipo | Descripción |
|---|---|---|
| `GET /peers` | HTTP | Lista de pares conocidos y su estado de conexión |
| `POST /peers/connect` | HTTP | Conectar a un par por URI WebSocket |
| `DELETE /peers/{id}` | HTTP | Desconectar un par |
| `GET /messages/{peer_id}` | HTTP | Historial de conversación con un par |
| `POST /files/send` | HTTP | Iniciar transferencia de archivo a un par |
| `GET /files/{file_id}` | HTTP | Descargar archivo recibido |
| `WS /ws/client` | WebSocket | Canal en tiempo real: mensajes entrantes, eventos de red, progreso de archivos |

### 9.2 Separación de Capas WebSocket

El sistema mantiene dos capas WebSocket completamente independientes:

```
Frontend ◀──── WS /ws/client ────▶ APIGateway (local)
                                          │
                                    Core Engine
                                          │
                        PeerConnector ◀──WS──▶ Par remoto
```

Esta separación garantiza que el frontend nunca interactúe directamente con la red P2P, permitiendo agregar cifrado, logging y control de flujo en el núcleo local sin modificar el cliente.

---

## 10. Estructura de Módulos

```
p2p_messenger/
├── core/
│   ├── interfaces.py           # Protocolos y tipos compartidos
│   ├── event_bus.py            # EventBus pub/sub interno
│   ├── message_router.py       # Enrutamiento next-hop + cola de espera
│   ├── routing_table.py        # Tabla dest_id → next_hop_id
│   ├── route_discovery.py      # Emisión/recepción rreq / route_reply
│   ├── route_request_cache.py  # Deduplicación con expiración temporal
│   ├── message_serializer.py   # NetworkFrame ↔ bytes
│   ├── file_transfer.py        # Fragmentación y reensamblaje
│   └── session_manager.py      # Handshake e identidad de pares
│
├── network/
│   └── peer_connector.py       # WebSocket client/server hacia pares remotos
│
├── storage/
│   └── conversation_store.py   # Persistencia SQLite vía aiosqlite
│
├── api/
│   ├── gateway.py              # FastAPI app + registro de routers
│   ├── routes_http.py          # Endpoints REST
│   └── routes_ws.py            # Endpoint WS /ws/client → Frontend
│
└── frontend/                   # React + TypeScript + Tailwind CSS
```

---

## 11. Decisiones de Diseño

### EventBus interno para desacoplamiento total

Los módulos del core no se invocan directamente entre sí; todos publican y consumen eventos a través del `EventBus`. Esto permite reemplazar o extender cualquier módulo (por ejemplo, el `ConversationStore`) sin tocar el resto del sistema.

### Dos capas WebSocket independientes

La conexión `PeerConnector ↔ par remoto` y la conexión `APIGateway ↔ Frontend` son completamente independientes. El frontend nunca habla directamente con otro par, lo que centraliza el control de acceso y facilita la adición de cifrado o logging en un único punto.

### `visited_set` en lugar de TTL

El TTL fue descartado por introducir un universo local: con `TTL=n`, un nodo solo puede alcanzar pares a distancia máxima n, federando la red de forma no intencionada. El `visited_set` garantiza cobertura exacta de toda la red alcanzable sin restricción de profundidad, manteniendo el espíritu simétrico del modelo P2P.

### Chunks fijos de 64 KB para transferencia de archivos

Permite control de progreso granular, posibilidad de reenvío selectivo de chunks fallidos, y evita saturar el buffer del WebSocket con binarios de gran tamaño. El hash SHA-256 al finalizar la transferencia garantiza integridad.

### Tres niveles de protección anti-bucle complementarios

`visited_set` (cobertura por nodo), `rreq_id` cache (deduplicación por llegada múltiple) y timeout con `route_error` (protección del origen ante destinos inexistentes) actúan en capas distintas y ninguno hace redundante a los otros.

### SQLite vía aiosqlite para persistencia

Apropiado para la escala objetivo del sistema (entorno universitario o de laboratorio). No introduce dependencias de servidor externo y es suficientemente capaz para historial de mensajes y metadatos de archivos transferidos.

---

*Documento generado a partir del diseño iterativo del sistema. Estado: planteamiento formal completo, pendiente de implementación.*
