# DOCUMENTO TÉCNICO DE DISEÑO
## Subsistema de Comunicaciones y Telemetría (COMMS)

**Proyecto:** CanSat Argentina Universitario 2026
**Equipo:** _(completar)_
**Autoría:** _(completar)_
**Versión:** 0.1, borrador para revisión previa al PDR
**Fecha:** 1 de noviembre de 2026
**Estado:** preliminar. Los supuestos abiertos están marcados **TBR** y listados en la [sección 11](#11-pendientes-y-consultas).

> **Nota del autor.** No vengo del área de telecomunicaciones. Este documento lo escribí aprendiendo el tema sobre la marcha: leí datasheets, normas argentinas y experiencias de otros equipos CanSat, hice las cuentas con fórmulas públicas y dejé escrito cómo las hice para que cualquiera pueda rehacerlas. Cuando algo es una estimación mía, o un dato que no pude verificar, lo marco. Cuando un concepto de radio hace falta para entender una decisión, lo explico en un recuadro **En criollo**. La idea es que Aviónica, Estructuras y Software puedan leerlo sin ser especialistas y entender qué se decidió y por qué. Si alguien que sabe de RF encuentra un error, es bienvenido: ya me pasó una vez durante la redacción (ver [sección 5.4](#54-presupuesto-de-enlace)).

**Cómo leer las marcas**

| Marca | Significado |
|---|---|
| **[F]** | Dato tomado de una fuente: datasheet, norma o ficha de venta. Las fuentes están en el [Anexo F](#anexo-f-fuentes). |
| **(est.)** | Cálculo o estimación propia. El método está explicado y, si es numérico, reproducible con el script del [Anexo B](#anexo-b-cálculos-reproducibles). |
| **(verificar)** | Dato de memoria o que una fuente da de forma ambigua. Confirmarlo antes de ponerlo en el PDR. |
| **TBR** | _To Be Reviewed_: supuesto de diseño que se resuelve con CONAE o con otro subsistema. |

---

## Índice

1. [Introducción y objetivos](#1-introducción-y-objetivos)
2. [Arquitectura del sistema de comunicaciones](#2-arquitectura-del-sistema-de-comunicaciones)
3. [Módulo de comunicación (radio)](#3-módulo-de-comunicación-radio)
4. [Trama de telemetría y telecomandos](#4-trama-de-telemetría-y-telecomandos)
5. [Enlace de radio: parámetros, tasa de datos y presupuesto](#5-enlace-de-radio-parámetros-tasa-de-datos-y-presupuesto)
6. [Antenas](#6-antenas)
7. [Estación terrena](#7-estación-terrena)
8. [Gestión de fallos y calidad de datos](#8-gestión-de-fallos-y-calidad-de-datos)
9. [Plan de ensayos](#9-plan-de-ensayos)
10. [Riesgos](#10-riesgos)
11. [Pendientes y consultas](#11-pendientes-y-consultas)
12. [Conclusiones](#12-conclusiones)

**Anexos:** [A. Código de referencia](#anexo-a-código-de-referencia) · [B. Cálculos reproducibles](#anexo-b-cálculos-reproducibles) · [C. Glosario](#anexo-c-glosario-en-criollo) · [D. Trazabilidad de requisitos](#anexo-d-trazabilidad-de-requisitos) · [E. Qué se reutiliza del COMMS V2](#anexo-e-qué-se-reutiliza-del-documento-comms-v2-del-cohete) · [F. Fuentes](#anexo-f-fuentes)

---

## 1. Introducción y objetivos

### 1.1 Contexto y propósito del subsistema

El subsistema COMMS es el único vínculo entre el CanSat y la estación terrena. Una vez que el CanSat se separa del cohete no hay cable ni contacto físico: todo lo que el equipo sepa del vuelo en tiempo real llega por radio, y todo lo que queramos pedirle al CanSat llega por radio.

El CanSat se integra dentro del cohete que provee CONAE, se despliega a una altitud superior a 500 m sobre el terreno (Bases §4) y desciende en dos etapas: primero a 15 ± 3 m/s y luego a 5 ± 3 m/s (C3, C4). Durante todo ese tiempo transmite un paquete de telemetría por segundo (X4, C5) y puede recibir telecomandos.

> **En criollo:** pensalo como un walkie-talkie que habla una vez por segundo ("estoy a tantos metros, así de inclinado, con esta batería") y que, entre frase y frase, se queda escuchando por si la estación le manda una orden.

### 1.2 Objetivos del subsistema

1. **Telemetría (CanSat → estación):** un paquete por segundo con tiempo de misión, contador de paquetes, altitud, presión, temperatura, aceleraciones y rotaciones en 3 ejes, GPS, voltaje y corriente de batería y estado del software de vuelo (X4, X5, G3, G7, G8, F1, F2).
2. **Telecomandos (estación → CanSat):** calibrar altitud a cero, configurar la hora, pedir estado de batería y paneles solares, y pedir información adicional (G1, F3, F4, X6, G15).
3. **Mensaje de comienzo de misión** automático al separarse del cohete (X3).
4. **Cumplir las reglas de radio:** banda 915–928 MHz o 2400–2483,5 MHz (X1) y sin modo broadcast (X2).
5. **Entregar los datos a la estación** de forma que se puedan mostrar, graficar y guardar en CSV (G2, G5 a G8).
6. **Funcionar al menos 2 h** integrado en el cohete (E7), dentro de masa (S1), volumen (S2, S3) y costo (C9).

### 1.3 Alcance

**Este documento cubre:** elección del módulo de radio, definición de la trama y de los comandos, parámetros del enlace y tasa de datos, antenas (CanSat y estación), interfaz radio ⇄ notebook, energía de la estación, software de la estación y plan de ensayos.

**No cubre (depende de otros subsistemas):**

| Tema | Responsable | Qué necesito de ellos |
|---|---|---|
| Microcontrolador de vuelo, sensores, pines SPI | Aviónica | Modelo de MCU, pines libres para el módulo LoRa, formato y rango de cada sensor |
| Batería y regulación | Aviónica | Tensión y corriente disponibles; que soporte picos de ≈ 120 mA |
| Máquina de estados de vuelo y detección de la separación | Aviónica | Definir y confirmar los estados de la [sección 4.4](#44-campos-de-estado-y-flags) |
| Contador de paquetes que persiste ante reinicios (F1) | Aviónica | Memoria no volátil con escrituras espaciadas (ver [4.8](#48-contadores-pérdidas-y-reinicios)) |
| Material del contenedor, espacio y soporte de la antena | Estructuras | Plástico simple (PLA/PETG), sin fibra de carbono ni relleno metálico |

### 1.4 Requisitos que mandan sobre el diseño

Resumen de los requisitos con impacto directo en la radio. La trazabilidad completa está en el [Anexo D](#anexo-d-trazabilidad-de-requisitos).

| Req. | Qué exige | Consecuencia de diseño |
|---|---|---|
| X1 | 915–928 MHz o 2400–2483,5 MHz | Se descartan 433 y 868 MHz. Se elige **915–928 MHz** |
| X2 | Sin modo broadcast | Cada paquete lleva destinatario; canal asignado; ID de equipo |
| X3 | Comienzo de misión al separarse | Flag de inicio de misión, repetido varios paquetes |
| X4, C5 | Un paquete por segundo | El tiempo en el aire debe ser bastante menor a 1 s |
| X5, G7, G8 | Contenido de la telemetría | Define el tamaño de la trama (52 B) |
| X6, F4, G1, F3, G15 | Telecomandos | La radio debe escuchar entre transmisiones (half-duplex) |
| F1, G8, G14 | Contador de transmitidos; recibidos y perdidos | Contador de 16 bits en cada paquete y CRC para validar |
| E7 | 2 h de funcionamiento integrado | Presupuesto de energía en recepción continua |
| E1, E2, M4 | Sin LiPo; sin contactos de resorte | La tensión disponible la define Aviónica |
| S1 a S8 | Masa, dimensiones, vibración, impacto, montaje fijo | Módulo soldado y antena con soporte mecánico |
| C9 | CanSat < $1.000.000 (la estación no cuenta) | Radio y antena del CanSat deben ser baratas |
| G9, G10, G11 | Notebook ≥ 2 h, sin corriente alterna, estación portátil | Radio de la estación alimentada por USB |
| G2, G4, G5, G6, G7, G8, G12, G13, G15 | Software de estación | [Sección 7](#7-estación-terrena) |

### 1.5 Resumen de decisiones

| # | Tema | Decisión | Alternativa principal | Sección |
|---|---|---|---|---|
| 1 | Banda y tecnología | **LoRa a 915–928 MHz** | 2,4 GHz (sin margen suficiente) | [3](#3-módulo-de-comunicación-radio) |
| 2 | Módulo de radio | **HopeRF RFM95W-915 (chip SX1276, interfaz SPI)** | EBYTE E32-915T20D / REYAX RYLR998 (UART) | [3.5](#35-elección) |
| 3 | Trama | **Binaria de 52 bytes, CRC-16, unidades SI escaladas** | ASCII/CSV de ≈ 110–121 bytes | [4](#4-trama-de-telemetría-y-telecomandos) |
| 4 | Parámetros LoRa | **BW 250 kHz, SF9, CR 4/5**: 164 ms en el aire | BW 125 kHz, SF9: 329 ms | [5.1](#51-parámetros-lora-elegidos) |
| 5 | **Tasa de datos** | **416 bit/s = 0,416 kb/s (0,052 kB/s)** a 1 Hz; ocupa el 16,4 % del canal | | [5.3](#53-tasa-de-datos-kb-por-segundo) |
| 6 | Antena del CanSat | **Monopolo λ/4 de alambre, ≈ 80 mm, dentro del contenedor** | Rubber duck comercial (plan B y referencia de ensayo) | [6.2](#62-antena-del-cansat) |
| 7 | Antena de la estación | **Omnidireccional vertical de 2–5 dBi en mástil** | Yagi de 10–11 dBi (opcional) | [6.3](#63-antena-de-la-estación-terrena) |
| 8 | Estación terrena | **Reutilizar y adaptar la estación existente (Python/FastAPI + interfaz web)**, con un núcleo de decodificación propio. **Condicionado a la consulta por G4** | Estación propia en PySide6 (plan de respaldo) | [7.4](#74-software-de-la-estación-cuatro-alternativas) |
| 9 | Interfaz radio ⇄ notebook | **Radio SPI + microcontrolador puente con USB (puerto serie virtual)** | Placa LoRa con USB integrado (Heltec, T-Beam) | [7.2](#72-hardware-de-la-estación-interfaz-radio--notebook) |
| 10 | Comandos | Opcode + argumentos + CRC; **confirmación por eco en la telemetría** | ACK de radio por paquete | [4.7](#47-telecomandos) |

### 1.6 Supuestos de diseño

Ninguno de estos valores sale de un documento de CONAE. Todos son supuestos míos y quedan marcados **TBR**.

| Supuesto | Valor | Justificación |
|---|---|---|
| Altitud máxima | 500 m (Bases) a 1 km (reunión) | Las Bases dicen solo "superior a 500 m". El 1 km salió de la reunión. Se diseña el margen con 1 km |
| Deriva horizontal | ≈ 1,7 km | Viento de 10 m/s y descenso de ≈ 170 s |
| Distancia en línea recta | ≈ 2 km | √(1² + 1,7²) |
| **Distancia de diseño del enlace** | **3 km** | Margen sobre los 2 km |
| Paquetes en el descenso | 56 a 208 | Según tolerancias de C3/C4 (a 1 Hz) |
| Transmisión en plataforma | Sí, desde STAND-BY | Ver [2.4](#24-estados-del-cansat-y-cuándo-transmite) |
| Ganancia efectiva de la antena del CanSat | −3 dBi (peor caso) | Dentro del contenedor, rotando |
| Ganancia de la antena de la estación | +3 dBi (omni) o +11 dBi (Yagi) | Dos casos |
| Pérdidas extra | 6 dB | Fuselaje, contenedor, polarización, conectores |
| Margen mínimo deseado | ≥ 10 dB, idealmente ≥ 15 dB | Práctica habitual |

---

## 2. Arquitectura del sistema de comunicaciones

### 2.1 Vista general

```
 CANSAT (segmento de vuelo)                         ESTACIÓN TERRENA (segmento tierra)
 ---------------------------                        ----------------------------------
 Sensores: baro, IMU, GPS, batería (V, I)           Antena omni 2-5 dBi en mástil o trípode
            |                                                   |  cable coaxil corto
 MCU de aviónica                                    Módulo LoRa RFM95W-915
            | SPI                                               | SPI
 Módulo LoRa RFM95W-915                             Microcontrolador puente (p. ej. ESP32-C3)
            | pigtail U.FL                                      | USB (puerto serie virtual)
 Antena λ/4 de alambre (~80 mm)                     Notebook
                                                       - Backend Python (FastAPI): decodifica,
                                                         cuenta pérdidas, guarda CSV x2
                                                       - Interfaz web: una sola pantalla
            )))--------- 915-928 MHz (LoRa) ---------(((
                 telemetría  ->  1 paquete/s (52 B)
                 <-  telecomandos (8 a 14 B)
```

### 2.2 Segmento de vuelo

- Módulo LoRa **RFM95W-915** conectado por SPI al microcontrolador de Aviónica.
- Antena monopolo de cuarto de onda de unos 80 mm, vertical, dentro del contenedor y lejos de las baterías.
- El firmware de radio arma la trama de 52 bytes, calcula el CRC, transmite, y vuelve a recepción para escuchar comandos.

### 2.3 Segmento terreno

- Antena omnidireccional de 2–5 dBi sobre mástil, con el coaxil más corto posible.
- Un segundo módulo **RFM95W-915** conectado a un microcontrolador **puente** que habla por USB con la notebook. El puente le entrega a la notebook cada paquete recibido junto con su RSSI y SNR (calidad de señal), y transmite los comandos que la notebook le pida.
- En la notebook corre el backend de la estación (Python/FastAPI) y la interfaz web. Detalle en la [sección 7](#7-estación-terrena).

> **En criollo:** el CanSat y la estación usan **el mismo módulo de radio**. Eso simplifica las compras (un repuesto sirve para los dos lados) y permite escribir un solo firmware base de radio para ambos extremos.

### 2.4 Estados del CanSat y cuándo transmite

La definición exacta de los estados la tiene que confirmar Aviónica (**TBR**). Propongo estos:

| Código | Estado | Cuándo | Transmite |
|---|---|---|---|
| 0 | STAND-BY | Encendido dentro del cohete, en plataforma | Sí, 1 Hz, y escucha comandos |
| 1 | ASCENSO | El cohete despega | Sí, 1 Hz |
| 2 | DESCENSO_1 | Tras la separación, primer sistema de descenso (≈ 15 m/s) | Sí, 1 Hz, con flag de inicio de misión |
| 3 | DESCENSO_2 | Segundo sistema de descenso (≈ 5 m/s) | Sí, 1 Hz |
| 4 | ATERRIZADO | En el suelo | Sí, 1 Hz mientras haya batería |
| 5 | ERROR | Falla de sensores o reinicio detectado | Sí, 1 Hz con flag de error |

**Por qué transmitir desde STAND-BY.** Las Bases tienen una ambigüedad: G5 pide mostrar telemetría en ascenso y descenso, mientras que X3 define el "comienzo de misión" en la separación. Mi lectura es que el CanSat está encendido y en STAND-BY dentro del cohete (necesita recibir los comandos G1 y F3 en plataforma) y que la misión "empieza" al separarse. Para cubrir ambas lecturas: transmitir a 1 Hz desde STAND-BY con un campo de estado en cada paquete, y al separarse cambiar de estado automáticamente y marcar los primeros paquetes con el flag **INICIO_MISIÓN**.

- **Costo de esta decisión:** transmitir durante las 2 h de E7 pesa en el presupuesto de energía (≈ 74 mAh, ver [5.5](#55-energía-del-radio)). Se lo paso a Aviónica.
- **Palanca de ahorro (TBR):** en STAND-BY se podría bajar a un paquete cada 5 s y volver a 1 Hz al despegar. No lo adopto por defecto para no sumar un estado más de lógica.
- **Detección de la separación:** la resuelve Aviónica. Una idea del research es usar el panel solar (adentro del fuselaje no hay luz, afuera sí). Es una lectura mía del texto, no está en las Bases.

### 2.5 Ciclo de 1 segundo (half-duplex)

El módulo no puede transmitir y recibir al mismo tiempo (**half-duplex**). La solución es ordenar el tiempo:

```
 0 ms           164 ms                                                    1000 ms
 |--- TX TLM ---|---------------- RX (escuchando) ---------------------|--- TX TLM ---> ...
                  ^ la estación manda el comando justo aquí (~175-260 ms)
                              ^ el CanSat responde con RSP (~280-400 ms), solo si hubo comando
```

| Ventana | Cuándo | Quién transmite | Duración (SF9/BW250) |
|---|---|---|---|
| Telemetría (TLM) | 0–164 ms | CanSat | 164 ms |
| Comando (CMD), solo si hay uno pendiente | ≈ 175–260 ms | Estación | 62–82 ms |
| Respuesta (RSP), solo si hubo comando que la pide | ≈ 280–400 ms | CanSat | ≈ 70–95 ms |
| Escucha | el resto del segundo | nadie | |

**Regla de oro:** la estación manda el comando **inmediatamente después de recibir un paquete de telemetría**. Así sabe que el CanSat acaba de transmitir y está por escuchar, no hay que adivinar el momento y no hay colisiones. El CanSat permanece en recepción todo el tiempo en que no transmite.

---

## 3. Módulo de comunicación (radio)

### 3.1 Qué le pedimos al módulo

| Criterio | Por qué importa |
|---|---|
| Opera en 915–928 MHz y permite elegir el canal | X1 y asignación de frecuencia de CONAE (Bases §6) |
| Permite direccionar a un destinatario (o lo podemos hacer nosotros) | X2 |
| Puede recibir comandos entre transmisiones | G1, F3, X6, F4, G15 |
| Tiempo en el aire bien menor a 1 s | X4, C5 |
| Margen de enlace ≥ 10 dB a 3 km | Distancia de diseño ([1.6](#16-supuestos-de-diseño)) |
| Consumo bajo en recepción continua | E7 (2 h) |
| Masa y tamaño chicos; se puede soldar | S1, S2, S3, S4, S5, S8 |
| Barato y disponible en Argentina | C9 y plazos del cronograma |

### 3.2 Las cuatro alternativas

Las cuatro son compatibles con la banda argentina si se configuran bien. Dejé afuera lo que no cumple X1 (433 y 868 MHz) y las familias de 2,4 GHz, que se justifican en la [3.6](#36-alternativas-descartadas).

#### A. HopeRF RFM95W-915 (chip SX1276, LoRa por SPI)

Es un módulo "pelado": el microcontrolador le habla por SPI y nuestro firmware decide todo (frecuencia, potencia, qué se transmite y cuándo).

- **Pros:** el más barato, liviano (≈ 3 g) y de menor consumo de los cuatro; control total del tiempo en el aire; mucha documentación y librerías abiertas (`LoRa` de Sandeep Mistry, RadioHead, RadioLib); se consigue en Argentina.
- **Contras:** hay que escribir el direccionamiento y la validación de paquetes (en este diseño ya están resueltos en la trama, [sección 4](#4-trama-de-telemetría-y-telecomandos)); no trae certificación; no hay que transmitir nunca sin antena conectada.
- **Alcance reportado:** unos 2 km en línea de vista con un alambre simple como antena; hasta 20 km con antenas direccionales y configuración óptima [F].

#### B. EBYTE E32-915T20D (SX1276, LoRa por UART)

Un módulo "con cerebro propio": el microcontrolador le manda bytes por una UART y el módulo se ocupa de la radio.

- **Pros:** el más fácil de integrar; direccionamiento por modo "fijo" (dirección + canal); cambio de canal por comando; FEC incluido.
- **Contras:** el tiempo en el aire no se controla con precisión (buffer interno, parte los datos en subpaquetes de 58 B) y el fabricante no lo publica: hay que medirlo con el pin AUX; **de fábrica la dirección 0x0000 es de difusión** (rompe X2); el cambio de canal exige acceso a su UART en modo "sleep".
- **Alcance reportado:** ≈ 3 km con antena de 5 dBi a 2,5 m, paquetes de 30 B cada 2 s y pérdidas < 5 % [F].

#### C. REYAX RYLR998 (LoRa por UART con comandos AT)

Parecido al E32, pero se maneja con comandos de texto tipo `AT+SEND=...`.

- **Pros:** el más chico y liviano (2 g, 32 × 14 mm); direccionamiento y red incluidos (`AT+ADDRESS`, `AT+NETWORKID`); fácil con Arduino/ESP32; certificaciones FCC/NCC/CE declaradas [F].
- **Contras:** peor sensibilidad de ficha (−129 dBm); no controlamos el paquete (los comandos AT agregan encabezado); menos documentación sobre tiempo en el aire.

#### D. Digi XBee-PRO 900HP, variante Australia (915–927 MHz)

El módulo "profesional" de la familia XBee: salto de frecuencia, direccionamiento y reintentos integrados.

- **Pros:** muy probado y robusto; el salto de frecuencia ayuda a convivir con otros equipos; unicast de 64 bits con ACK y reintentos.
- **Contras:** **la variante apta para Argentina (‑022) es difícil de conseguir** y la versión US (902–928 MHz) sale de la banda permitida; plazo de 17 a 25 semanas en kits [F]; ≈ 4 a 6 veces más caro; en un ensayo con tramas chicas y espaciadas se vio degradación de rendimiento (foro de Digi) [F].

### 3.3 Tabla comparativa

| | **A. RFM95W-915** | **B. E32-915T20D** | **C. RYLR998** | **D. XBee-PRO 900HP (AU)** |
|---|---|---|---|---|
| Tecnología | LoRa / FSK (SX1276) | LoRa (SX1276) | LoRa | FSK con salto de frecuencia |
| Interfaz | SPI | UART 3,3 V | UART + comandos AT | UART o SPI (modo AT o API) |
| Banda y canales | Cualquier canal por software | 900–931 MHz, 32 canales | 820–960 MHz | 915–927 MHz (AU), FHSS |
| Potencia máx. | +20 dBm (100 mW) | +20 dBm | +22 dBm | +24 dBm |
| Sensibilidad | −126,5 dBm en nuestra config. (est.); hasta −148 dBm a tasas muy bajas [F] | −144 a −148 dBm a 0,3 kb/s; peor a 2,4 kb/s [F] | −129 dBm [F] | −110 dBm (10 kb/s); −101 dBm (200 kb/s) [F] |
| Velocidad en el aire | 3,5 kb/s en nuestra config. (LoRa hasta 37,5 kb/s) | 0,3–19,2 kb/s (2,4 por defecto) | según `AT+PARAMETER` | 10 o 200 kb/s |
| Aire de nuestra trama (52 B) | **164 ms**, calculado y controlable | ≈ 230–250 ms a 2,4 kb/s (est., medir) | no publicado | ≈ 3 ms (200 kb/s) o ≈ 55 ms (10 kb/s) (est.) |
| Dúplex | Half (lo conmuta el firmware) | Half automático | Half | Half (lo gestiona el módulo) |
| Direccionamiento (X2) | Por software: byte de destino + sync word + canal | Modo fijo (dirección + canal). **Evitar 0x0000/0xFFFF** | `AT+ADDRESS` + `AT+NETWORKID` (unicast) | Unicast de 64 bits con ACK. **Evitar 0x…FFFF** |
| Cambio de canal | Por firmware | Comando por UART (modo 3) | Por AT | Por AT / XCTU |
| Alimentación | 1,8–3,7 V | 2,3–5,2 V | 1,8–3,6 V | 2,1–3,6 V |
| Corriente RX | 10–12 mA | 13–15 mA | 17,5 mA | 29 mA |
| Corriente TX | 90 mA (+17 dBm) / 120 mA (+20 dBm) | 110–130 mA | 140 mA (+22 dBm) | 215 mA |
| Masa y tamaño | ≈ 3 g; 16 × 16 × 2 mm | 6,7 g; 21 × 36 mm | 2 g; 32 × 14 × 2,2 mm | n/d; 20 pines |
| Precio | USD 6,5–11 [F]; en Mercado Libre AR ≈ $20.000–61.000 según versión | ≈ USD 10 [F] (≈ $15.000, est.) | USD 11,99 [F] (≈ $18.500, est.) | ≈ USD 61 [F] (≈ $94.000, est.) |
| Disponibilidad | Stock local (Mercado Libre) | Importación | Importación | Variante AU difícil; plazos de meses |
| Margen a 3 km (est.) | ≈ 39 dB | ≈ 42 dB | ≈ 44 dB | ≈ 18–27 dB |

_Fuentes: research 01 (secciones 7 y 8) y 03; precios en pesos con dólar oficial ≈ $1.540 (20/10/2026), sin envío ni impuestos de importación; reverificar el día de comprar. Los márgenes se calculan con el modelo de la [sección 5.4](#54-presupuesto-de-enlace) (omni de 3 dBi en la estación)._

### 3.4 Matriz de decisión

Puntajes de 1 a 5 (juicio mío, tomado del research 01 §12). Los pesos suman 100. **Es una herramienta para descartar y ordenar prioridades, no para desempatar a los cuatro primeros** (quedan dentro de 7 puntos entre sí y los puntajes son subjetivos).

| Criterio | Peso | A. RFM95W | B. E32-915T20D | C. RYLR998 | D. XBee-PRO 900HP |
|---|---:|---:|---:|---:|---:|
| Margen de enlace a 3 km | 20 | 5 | 4 | 4 | 3 |
| Costo y disponibilidad en Argentina | 20 | 5 | 4 | 4 | 2 |
| Facilidad de integración y desarrollo | 15 | 3 | 5 | 5 | 5 |
| Consumo y masa | 10 | 5 | 4 | 5 | 4 |
| Cumplir X2 con poco trabajo | 10 | 3 | 4 | 4 | 5 |
| Recibir comandos mientras se transmite | 10 | 3 | 4 | 4 | 4 |
| Convivencia con otros equipos en el campo | 10 | 4 | 3 | 3 | 5 |
| Evidencia de uso en telemetría similar | 5 | 4 | 3 | 2 | 5 |
| **Total (sobre 100)** | | **83** | **80** | **81** | **76** |

### 3.5 Elección

**Elegido: A. HopeRF RFM95W-915 (SX1276, SPI), con firmware propio.**

Los puntajes no alcanzan para desempatar. Lo que inclina la balanza hacia la A es, en orden de importancia:

1. **Ya tenemos la experiencia y el código.** El subsistema COMMS V2 del cohete usa un transceptor de la misma familia (SX127x) por SPI con la librería `LoRa` de Sandeep Mistry y una trama binaria con CRC-16. Pasar a 915 MHz es cambiar el módulo, la frecuencia y la trama; no es empezar de cero (ver [Anexo E](#anexo-e-qué-se-reutiliza-del-documento-comms-v2-del-cohete)).
2. **Control del tiempo en el aire.** El E32 y el RYLR998 esconden el paquete detrás de un buffer o de comandos AT; con la A sabemos exactamente qué sale y en cuánto tiempo (164 ms), y eso importa para la ventana de comandos.
3. **Mismo módulo en los dos extremos.** El puente de la estación usa el mismo RFM95W: un repuesto sirve para ambos lados y el firmware de radio es casi el mismo.
4. **Mejor relación masa, consumo y costo.** Es el más barato y el que menos consume; el costo de radio más antena del CanSat queda entre ≈ 2,5 % y ≈ 13 % del tope de C9 (est., peor caso con la antena cara).
5. **El costo es desarrollo, no dinero.** Hay que escribir direccionamiento y validación, pero en este diseño ya están resueltos en la trama ([sección 4](#4-trama-de-telemetría-y-telecomandos)).

**Plan B (si el desarrollo se atrasa):** E32-915T20D o RYLR998. Se integran más rápido pero perdemos control fino. Con el E32 hay que cambiar la dirección de fábrica y usar modo fijo. **Plan C (si hay plata y plazo):** XBee-PRO 900HP variante AU.

> **En criollo:** la A es como armar el walkie-talkie con piezas (más trabajo, pero sabemos qué hace cada una) y la B y la C son walkie-talkies casi hechos (más rápidos de usar, pero con menos control).

### 3.6 Alternativas descartadas

| Alternativa | Motivo |
|---|---|
| **XL1278-SMT (SX1278, 433 MHz)**, el módulo del cohete V2 | Fuera de X1. Además el SX1278 es un chip de banda baja (cubre hasta ≈ 525 MHz, verificar) y **no se puede sintonizar a 915 MHz**: hay que comprar un módulo basado en SX1276/77/79 |
| RFD900x y radios SiK | USD 199 (≈ $300.000, un tercio de C9); 14–24 g; 1 W y 5 V con picos de ≈ 1 A. Las versiones US saltan por 902–915 MHz, fuera de X1 |
| EBYTE E22-900T30D (SX1262, 30 dBm) | Muy buena sensibilidad (−134 dBm) y LBT (escuchar antes de transmitir), pero a 30 dBm pide 5 V y picos altos. No necesitamos tanta potencia |
| CC1101 | Margen a 3 km ≈ 17 dB; FIFO de 64 B; sin modulación LoRa |
| 2,4 GHz: SX1280, XBee3, nRF24L01+, ESP-NOW | Margen a 3 km (est.): SX1280 ≈ 18 dB; XBee3 PRO ≈ 6 dB; nRF24L01+ con PA ≈ 8 dB; ESP-NOW ≈ −2 dB. La banda está saturada de Wi-Fi, Bluetooth y drones en el campo. El nRF24L01+ solo admite 32 B por paquete y no entra una trama de 52 B |
| LoRaWAN | Necesita una puerta de enlace y está pensado para subidas poco frecuentes, no para 1 Hz |
| Wi-Fi de infraestructura y Bluetooth | Alcance insuficiente y asociación previa |

### 3.7 Normativa argentina que afecta la elección

| Tema | Qué dice | Fuente |
|---|---|---|
| Bandas de uso compartido | 915–928 MHz y 2400–2483,5 MHz, sin autorización previa | Res. 581/2018 [F] |
| Potencia máxima | **30 dBm conducidos y 36 dBm de PIRE** | Anexo de la Res. 581/2018 [F] |
| Antenas con más de 6 dBi | Hay que bajar la potencia del transmisor en tantos dB como excedan los 6 dBi | Anexo de la Res. 581/2018 [F] |
| 902–915 MHz | **No** figura como banda compartida | Res. 1132/2023 [F] |
| Equipos de baja potencia | Deben cumplir ENACOM-Q2-60.14 V20.1 | Res. 551/2021 [F] |

Consecuencias para este diseño:

- Con +20 dBm de potencia y antenas de hasta 6 dBi **cumplimos con holgura** (PIRE ≤ 26 dBm). Con una Yagi de 11 dBi en la estación, la potencia conducida máxima permitida baja a 25 dBm, y usamos ≤ 20 dBm: sigue cumpliendo.
- **Duda sin resolver (TBR):** la norma ENACOM-Q2-60.14 limita el campo a 50.000 µV/m medidos a 3 m, que equivale a unos 0,75 mW de PIRE (−1,25 dBm, cálculo del research 02). No pude determinar qué régimen aplica a un módulo LoRa de venta común ni si CONAE exige homologación a un prototipo universitario. **Hay que preguntarlo** ([sección 11](#11-pendientes-y-consultas)). Si aplicara el régimen más restrictivo, bajaríamos la potencia y el margen (ver [5.4](#54-presupuesto-de-enlace)) lo tolera.

### 3.8 Advertencias de compra y configuración

1. **Comprar "RFM95W-915S2"** (variante de 915 MHz). Las de 868 MHz se pueden sintonizar por software pero con peor eficiencia.
2. **Usar el plan AU915 (915–928 MHz), no el US915 (902–928 MHz).** Muchas librerías y placas traen canales por defecto en 902–915 MHz, que están **fuera** de X1. Programar la frecuencia explícitamente entre 916 y 927 MHz.
3. **Nunca transmitir sin antena conectada:** se puede dañar la etapa de potencia.
4. **Soldar el módulo** y fijar el cable de antena (vibración 15 G, impacto 30 G, montaje fijo).
5. **Condensadores de desacople** (por ejemplo 100 µF + 100 nF) junto al módulo: los picos de ≈ 120 mA pueden bajar la tensión y reiniciar el microcontrolador.
6. **Comprar al menos 3 módulos:** CanSat, estación y repuesto/ensayo destructivo.
7. **Sync word propio**, distinto de 0x34 (reservado para LoRaWAN público).

---

## 4. Trama de telemetría y telecomandos

### 4.1 Criterios de diseño

Seguí la misma idea del documento COMMS V2 del cohete: la trama se divide en **Header** (cabecera de control), **Payload** (carga útil de sensores y estados) y **Trailer** (verificación de errores). Es el formato clásico de telemetría y se traduce directo a una `struct` empaquetada en C. Lo que cambia es el contenido, que ahora lo dictan los requisitos del CanSat.

1. **Binaria y de tamaño fijo.** Menos bytes en el aire significan menos tiempo de transmisión, menos probabilidad de error y más tiempo para escuchar comandos. G5 pide *mostrar* la telemetría en texto, no *transmitirla* en texto: binario en el aire, texto en pantalla y en los CSV.
2. **Enteros escalados en unidades SI, sin `float`.** Cada campo guarda un entero con una resolución fija (por ejemplo, la altitud en decímetros). Ocupa menos que un `float` de 4 bytes, la conversión a unidades SI es una simple división (G6) y no hay ambigüedad de formato entre microcontroladores.
3. **CRC-16 al final** para validar la integridad (G14).
4. **Contador de 16 bits** que sobrevive a reinicios (F1), para calcular paquetes perdidos (G8).
5. **ID de destino y de origen** en cada paquete (X2).
6. **Versión de protocolo** en un byte, para poder cambiar la trama sin confundir a la estación.

> **En criollo:** el CRC es como el dígito verificador del DNI: se calcula con todos los bytes anteriores. Si en el aire se corrompe aunque sea un bit, el dígito no cierra y la estación sabe que ese paquete no sirve.

### 4.2 Alternativas de formato

| Opción | Tamaño de la trama | Aire (SF9/BW250) | Validación | Pros | Contras |
|---|---|---|---|---|---|
| A. Texto tipo CSV con checksum (estilo NMEA) | ≈ 110–121 B | ≈ 300–330 ms | Checksum XOR (débil) | Se lee con cualquier terminal | Paquete grande: más aire y más error; se parte en 3 subpaquetes en los módulos E32 |
| **B. Binario de tamaño fijo + CRC-16 (elegida)** | **52 B** | **164 ms** | CRC-16 | Compacto; simple; reutiliza el diseño del COMMS V2 | Hay que escribir el decodificador; un cambio de campo obliga a actualizar ambos lados (por eso el byte de versión) |
| C. MAVLink 2 (dialecto propio) | ≈ 55–60 B | ≈ 175–185 ms | CRC-16/MCRF4XX | Muchas herramientas existentes | Arrastra infraestructura que no necesitamos; por defecto es "multicast" sin destino, que choca con X2 |
| D. Datagrama RadioHead + payload binario | ≈ 56 B | ≈ 175 ms | CRC propio + CRC del chip | Direccionamiento TO/FROM listo | Su ID de mensaje es de 8 bits y no alcanza como contador F1; la dirección 255 es broadcast |

Tiempos en el aire: cálculo propio, ver [Anexo B](#anexo-b-cálculos-reproducibles).

### 4.3 Trama de telemetría elegida (52 bytes)

Todos los campos en **little-endian** (el byte menos significativo primero, como en ESP32 y la mayoría de los microcontroladores). Estructura: **Header 5 B + Payload 45 B + Trailer 2 B = 52 B.**

| Offset | Bloque | Campo | Tipo | Bytes | Unidad / resolución | Req. |
|---:|---|---|---|---:|---|---|
| 0 | Header | Destino (ID de la estación) | uint8 | 1 | | X2 |
| 1 | Header | Origen (ID del CanSat) | uint8 | 1 | | X2 |
| 2 | Header | Tipo y versión | uint8 | 1 | nibble alto: tipo (1 = TLM); nibble bajo: versión (1). TLM v1 = `0x11` | |
| 3 | Header | Contador de paquetes transmitidos | uint16 | 2 | persiste ante reinicios | F1, G8, G14 |
| 5 | Payload | Tiempo de misión | uint32 | 4 | 1 s | G3, F2 |
| 9 | Payload | Estado de vuelo | uint8 | 1 | ver [4.4](#44-campos-de-estado-y-flags) | G8 |
| 10 | Payload | Flags | uint8 | 1 | ver [4.4](#44-campos-de-estado-y-flags) | X3, F5 |
| 11 | Payload | Altitud barométrica | int16 | 2 | 0,1 m (dm) | X5 |
| 13 | Payload | Presión | uint32 | 4 | 1 Pa | X5 |
| 17 | Payload | Temperatura | int16 | 2 | 0,01 °C | X5 |
| 19 | Payload | Aceleración X, Y, Z | 3 × int16 | 6 | 0,01 m/s² | X5, G7 |
| 25 | Payload | Velocidad angular X, Y, Z | 3 × int16 | 6 | 0,001 rad/s | X5, G7 |
| 31 | Payload | GPS latitud | int32 | 4 | 1e-7 ° | X5 |
| 35 | Payload | GPS longitud | int32 | 4 | 1e-7 ° | X5 |
| 39 | Payload | GPS altitud | int32 | 4 | 1 cm | X5 |
| 43 | Payload | GPS satélites | uint8 | 1 | | X5 |
| 44 | Payload | Tensión de batería | uint16 | 2 | 1 mV | G7 |
| 46 | Payload | Corriente de batería | int16 | 2 | 1 mA | G7 |
| 48 | Payload | Último comando recibido (número de secuencia) | uint8 | 1 | | confirmación por eco |
| 49 | Payload | Resultado del último comando | uint8 | 1 | 0 = OK | confirmación por eco |
| 50 | Trailer | CRC-16/MODBUS de los bytes 0 a 49 | uint16 | 2 | | G14 |
| | | **Total** | | **52** | | |

| Sección | Tamaño |
|---|---:|
| Header (cabecera) | 5 bytes |
| Payload (sensores y estados) | 45 bytes |
| Trailer (CRC-16) | 2 bytes |
| **Total** | **52 bytes (416 bits)** |

**Por qué estas resoluciones y rangos** (a validar con Aviónica según sensores reales):

| Campo | Rango que cubre | Comentario |
|---|---|---|
| Altitud (int16, 0,1 m) | ±3276,7 m | Sobra para 500 m–1 km. Con altitud y tiempo a 1 Hz se verifican las velocidades de C3 (15 ± 3 m/s) y C4 (5 ± 3 m/s) |
| Aceleración (int16, 0,01 m/s²) | ±327,67 m/s² (≈ ±33 g) | Cubre el impacto de 30 G (S5). Si el sensor satura antes, que se sepa |
| Velocidad angular (int16, 0,001 rad/s) | ±32,767 rad/s (≈ ±1877 °/s) | Está en rad/s, que es la unidad SI (G6). Un CanSat que gira puede superar los 327 °/s, por eso no usé 0,01 °/s |
| Temperatura (int16, 0,01 °C) | ±327 °C | |
| Tensión (uint16, mV) | 0–65,5 V | |
| Corriente (int16, mA) | ±32,7 A | |
| GPS lat/lon (int32, 1e-7 °) | ±214° | Resolución de ≈ 1 cm; es el formato nativo de muchos módulos GPS |
| Tiempo de misión (uint32, s) | 136 años | **TBR:** definir con Aviónica si arranca en el encendido, en la separación o es hora UTC. G3 pide resolución de 1 s |

**Por qué no hay "Sync Byte" como en el cohete V2.** El documento COMMS V2 abre cada trama con `0xAA` para detectar el inicio de un paquete. Con LoRa el propio módulo ya detecta el inicio (preámbulo y *sync word*) y descarta lo que no es suyo, y entrega el paquete completo. El byte de destino y el de tipo/versión cumplen el rol de identificar un paquete legítimo. Si en algún momento se usara un módulo UART transparente (plan B), habría que agregar un delimitador (por ejemplo COBS).

### 4.4 Campos de estado y flags

**Estado de vuelo** (uint8), **TBR** con Aviónica: `0` STAND-BY · `1` ASCENSO · `2` DESCENSO_1 · `3` DESCENSO_2 · `4` ATERRIZADO · `5` ERROR.

**Flags** (uint8, un bit por evento):

| Bit | Nombre | Significado |
|---|---|---|
| 0 | INICIO_MISIÓN | Está activo durante los primeros 5 paquetes tras la separación (X3) |
| 1 | ALT_CALIBRADA | Se ejecutó `CAL_ALT_ZERO` (F5: sobrevive a un reinicio) |
| 2 | HORA_CONFIGURADA | Se ejecutó `SET_TIME` (F5) |
| 3 | GPS_FIX | El GPS tiene posición válida |
| 4 | ERROR_SENSOR | Algún sensor no responde o devuelve un valor inválido |
| 5 | REINICIO | Se detectó un reinicio del microcontrolador en este vuelo |
| 6–7 | reservados | |

**Inicio de misión (X3).** Al detectar la separación, el CanSat cambia solo a DESCENSO_1 y activa INICIO_MISIÓN durante **5 paquetes seguidos**. Así, si se pierde el primero, la estación igual lo ve. Esto es una interpretación mía del texto de X3 (dice que "la telemetría enviará un comando" sin decir a quién; lo leo como un mensaje de inicio hacia la estación): hay que confirmarlo con CONAE.

### 4.5 Control de errores: CRC-16

- **Algoritmo:** CRC-16/MODBUS (polinomio `0xA001` reflejado, valor inicial `0xFFFF`). Es la misma función que ya usa el documento COMMS V2 del cohete ([Anexo A.1](#a1-trama-y-crc-compartido-por-cansat-y-estación)), así que no hay que escribirla de nuevo.
- **Qué cubre:** los 50 primeros bytes (todo salvo el propio CRC). Se guarda en los últimos 2 bytes, little-endian.
- **Regla de oro:** un paquete cuenta como **"recibido con éxito" (G14) solo si pasa el CRC en la estación.** El chip LoRa también valida un CRC propio (lo activamos), pero no reemplaza al nuestro: el tramo USB y el firmware puente también pueden fallar.
- **Qué detecta:** todo error de 1 o 2 bits y ráfagas de hasta 16 bits; un error aleatorio pasa sin detectarse con probabilidad de ≈ 1/65.536.

### 4.6 Vector de prueba

Sirve para que los dos extremos (firmware en C y software de la estación en Python) verifiquen que se entienden. Es una trama real generada en C y decodificada en Python (los resultados coinciden).

```
Trama (52 bytes, hex):
010711D204100E000002090B137C7301002E090C00DEFFD503960024FFB80BF8E45FEBC0AC33DD603B00000AE81C5E01030024FF

CRC-16/MODBUS de los bytes 0..49: 0xFF24  (se guarda como 24 FF, little-endian)
```

| Campo | Valor decodificado |
|---|---|
| Destino / origen / tipo-versión | 0x01 / 0x07 / 0x11 (TLM v1) |
| Contador | 1234 |
| Tiempo de misión | 3600 s |
| Estado / flags | 2 (DESCENSO_1) / 0x09 (INICIO_MISIÓN + GPS_FIX) |
| Altitud / presión / temperatura | 487,5 m / 95.100 Pa / 23,5 °C |
| Aceleración X, Y, Z | 0,12 / −0,34 / 9,81 m/s² |
| Velocidad angular X, Y, Z | 0,15 / −0,22 / 3,00 rad/s |
| GPS lat / lon / altitud / satélites | −34,6037° / −58,3816° / 152,0 m / 10 |
| Batería | 7,400 V / 0,350 A |
| Último comando / resultado | 3 / 0 |

### 4.7 Telecomandos

**Formato de la trama de comando** (tipo = 2):

```
[destino][origen][tipo/ver = 0x21][secuencia][opcode][longitud][argumentos ...][CRC-16 (2 B)]
   1 B      1 B        1 B           1 B       1 B      1 B        0..n B           2 B
```

Ejemplo real, `CAL_ALT_ZERO` con secuencia 5, de la estación (`0x01`) al CanSat (`0x07`): `07 01 21 05 01 00 27 C1` (8 bytes).

| Opcode | Nombre | Argumentos | Tamaño | Aire (SF9/BW250) | Cuándo | Req. |
|---|---|---|---:|---:|---|---|
| `0x01` | `CAL_ALT_ZERO` | ninguno | 8 B | 62 ms | En plataforma, antes del despegue. Solo se acepta en STAND-BY | G1, F5 |
| `0x02` | `SET_TIME` | UTC en segundos (uint32) + milisegundos (uint16) | 14 B | 82 ms | Antes del lanzamiento | F3, F2 |
| `0x03` | `REQ_BATT_SOLAR` | ninguno | 8 B | 62 ms | Sin definir (TBR) | X6 |
| `0x04` | `REQ_INFO` | código (1 B) | 9 B | 72 ms | Sin definir; se documenta en el manual de la misión | F4 |
| `0x05` | `SET_CANAL` | índice de canal (1 B) | 9 B | 72 ms | Solo en STAND-BY (propuesta mía, ver [5.1](#51-parámetros-lora-elegidos)) | Bases §6 |

Los opcodes son una propuesta mía (**TBR**). El evento "comienzo de misión" (X3) no es un comando: va como flag en la telemetría ([4.4](#44-campos-de-estado-y-flags)).

**Respuestas (RSP, tipo = 3).** `REQ_BATT_SOLAR` y `REQ_INFO` devuelven datos. El CanSat responde con un paquete corto `[cabecera 5 B][opcode][secuencia][resultado][datos 0–8 B][CRC]` (10–18 B, ≈ 70–95 ms). Para `REQ_BATT_SOLAR` los datos son tensión y corriente de batería y tensión y corriente de panel solar (4 × 2 B). Qué es exactamente "señal de paneles solares" (X6) lo define Aviónica (**TBR**).

**Códigos de resultado:** `0` OK · `1` comando desconocido · `2` argumento inválido · `3` no permitido en el estado actual · `4` error de ejecución.

**Confirmación de comandos (ACK).** Los documentos no piden confirmación, así que lo decidimos nosotros. Evalué tres alternativas:

| Alternativa | Cómo funciona | Pros | Contras |
|---|---|---|---|
| 1. Sin ACK | Se envía y se espera lo mejor | Cero costo | Si se pierde la calibración (G1), nadie se entera |
| 2. ACK de radio por paquete | El receptor responde cada comando con un paquete de confirmación | Estándar | Gasta aire extra y exige que el CanSat esté en recepción en el momento justo; complica el half-duplex |
| **3. Confirmación por eco en la telemetría (elegida)** | La telemetría trae `último comando` y `resultado`; la estación reintenta hasta verlo reflejado | Aprovecha el paquete que ya sale cada segundo; no suma transmisiones | La confirmación llega con ≈ 1 s de latencia |

**Reintentos.** La estación manda el comando justo después de recibir una telemetría y espera ver el eco en los siguientes paquetes. Si no aparece tras 3 telemetrías, reenvía (máximo ≈ 10 intentos y avisa en pantalla). Los comandos de la plataforma (G1 y F3) se pueden repetir las veces que haga falta, porque son idempotentes.

**Comando de hora (F3).** Pide ± 1 s respecto de UTC. La latencia de aire (≈ 0,1–0,3 s) y de USB (decenas de ms) cae dentro del margen. Igual conviene que la estación mande la hora a la que **predice** que llegará (hora actual más latencia estimada) y que el reloj de la notebook esté sincronizado por NTP antes de salir al campo, porque allá no habrá internet.

### 4.8 Contadores, pérdidas y reinicios

**Contador de paquetes transmitidos (F1).** Lo mantiene el CanSat y debe persistir ante reinicios. Aviónica tiene que cuidar un detalle: escribir en la memoria flash a 1 Hz la desgasta en horas. Conviene una memoria FRAM o un esquema de escritura espaciada (por ejemplo cada 10 paquetes), asumiendo que tras un reinicio el contador puede **saltar unas pocas unidades**. Ese salto no es un error de comunicación, pero hay que saber explicarlo.

**Cálculo en la estación** (G8 y G14), manejando el desborde de 16 bits:

```
rx_ok      = paquetes con CRC válido
esperados  = (contador del último válido - contador del primero válido + 1)   [módulo 65536]
perdidos   = esperados - rx_ok
```

La implementación probada está en el [Anexo A.4](#a4-decodificador-y-contador-de-pérdidas-python-estación). El contador se incrementa con **cada paquete que transmite el CanSat**, sea telemetría o respuesta, así que las respuestas no generan falsas pérdidas. Si llega un contador repetido o menor al anterior, se lo trata como duplicado o fuera de secuencia y no se cuenta. Si el CanSat se reinicia, el flag REINICIO queda activo y el contador sigue, así que no hay saltos hacia atrás.

### 4.9 Direccionamiento y X2 ("sin broadcast")

X2 no define "modo de difusión". En RF todo transmisor emite hacia todos, así que solo puede referirse a direccionamiento. Mi lectura es que **cada paquete debe tener un destinatario específico**, para no inundar a los demás equipos del campo.

Diseño que cubre cualquier interpretación (**TBR**, a documentar en el PDR):

1. **ID de destino y de origen en cada paquete**, con filtro en el receptor: lo que no es para mí se descarta. Valores `0x00` y `0xFF` reservados, nunca se asignan, para no confundirlos con difusión.
2. **Canal/frecuencia asignado por CONAE** (la FRR lo verifica, Bases §6).
3. **Sync word propio** del equipo.
4. **Nunca** usar direcciones de difusión del módulo (en los módulos UART: 0x0000 y 0xFFFF).

> **En criollo:** aun con direccionamiento, la señal la reciben físicamente los equipos vecinos: el ID evita que "hagan caso" a nuestros paquetes, pero lo que evita la interferencia es que cada equipo use su propio canal.

---

## 5. Enlace de radio: parámetros, tasa de datos y presupuesto

### 5.1 Parámetros LoRa elegidos

LoRa permite "regular" el enlace con tres perillas. Como no soy especialista, las expliqué con una analogía antes de elegirlas.

| Parámetro | Qué es | Efecto | Elegido |
|---|---|---|---|
| **Ancho de banda (BW)** | Cuánto "espacio" del espectro ocupa la señal | Más ancho = más rápido, pero capta más ruido y tiene menos alcance | **250 kHz** |
| **Spreading Factor (SF)** | Cuánto se "estira" cada símbolo en el tiempo (de 7 a 12) | Más SF = más alcance, pero cada paquete tarda mucho más (casi el doble por cada paso) | **SF9** |
| **Coding Rate (CR)** | Cuántos bits de redundancia se agregan para corregir errores | Más redundancia = más robusto, pero paquete más largo | **4/5** |

> **En criollo:** pensalo como hablar a los gritos con un amigo lejos. Hablar más lento y marcando bien cada sílaba (SF alto) llega más lejos, pero tarda más en decir la frase. Con un paquete por segundo tenemos tiempo de sobra para hablar despacio, así que conviene gastar tiempo en ganar alcance. El cohete V2 usaba SF7 (lo más rápido) porque transmitía 10 veces por segundo; nosotros no lo necesitamos.

**Configuración completa:**

| Parámetro | Valor |
|---|---|
| Frecuencia | Canal asignado por CONAE, dentro de **916–927 MHz**. Con BW de 250 kHz, los centros deben estar entre 916 y 927 MHz para que la señal no se salga de 915–928 MHz. Canal de ensayo hasta que haya asignación: **921,5 MHz** (centro de la banda) |
| Ancho de banda | 250 kHz |
| Spreading Factor | 9 |
| Coding Rate | 4/5 |
| Preámbulo | 8 símbolos |
| Cabecera | Explícita |
| CRC del radio | Activado |
| Sync word | Propio del equipo, distinto de 0x34 (LoRaWAN) |
| Potencia | **+17 dBm** al inicio, hasta **+20 dBm** si el ensayo de alcance lo pide |
| Respaldo de más alcance | BW 125 kHz, SF9 (329 ms) |

### 5.2 Tiempo en el aire (Time on Air)

Es el tiempo que el módulo pasa efectivamente emitiendo un paquete. Se calcula con la fórmula del fabricante (Semtech AN1200.13), implementada en el [Anexo B](#anexo-b-cálculos-reproducibles). Verifiqué mi implementación contra ejemplos publicados (2793,5 ms para 64 B a SF12/BW125 y 354,3 ms para 16 B a SF11/BW250 con preámbulo de 16) y coincidieron [F]. Para el cohete V2 (65 B, SF7, BW250) la misma función da 61,57 ms, igual a lo que figura en ese documento.

**Tiempo en el aire de nuestra trama de 52 bytes (est.)**

| BW \ SF | SF7 | SF8 | SF9 | SF10 | SF11 | SF12 |
|---|---:|---:|---:|---:|---:|---:|
| 125 kHz | 103 ms | 185 ms | 329 ms | 616 ms | 1315 ms | 2466 ms |
| **250 kHz** | 51 ms | 92 ms | **164 ms** | 308 ms | 575 ms | 1233 ms |
| 500 kHz | 26 ms | 46 ms | 82 ms | 154 ms | 288 ms | 535 ms |

**Criterio:** la telemetría debe ocupar ≤ 200–250 ms para dejar ≥ 500 ms de escucha. Entran BW125 con SF7 y SF8, BW250 con SF7 a SF9 y BW500 con SF7 a SF10. **BW125/SF10 o superiores no entran con holgura, y SF11 y SF12 a BW125 ni siquiera entran en 1 s.** Elijo **BW250/SF9 (164 ms)**; como respaldo con más alcance, BW125/SF9 (329 ms), que todavía deja 670 ms libres.

### 5.3 Tasa de datos (kb por segundo)

Éste es el número que define el tamaño del "caño" que necesitamos.

| Magnitud | Valor | Cómo se obtiene |
|---|---|---|
| Tamaño de la trama | **52 B = 416 bits** | [4.3](#43-trama-de-telemetría-elegida-52-bytes) |
| **Tasa de datos de telemetría a 1 Hz** | **416 bit/s = 0,416 kb/s = 0,052 kB/s** | 52 B × 8 bit × 1 paquete/s |
| Comando típico | 8 a 14 B (64 a 112 bits), esporádico | [4.7](#47-telecomandos) |
| Velocidad nominal LoRa (SF9, BW250, CR 4/5) | 3,52 kb/s | SF × (BW / 2^SF) × CR = 9 × 488,3 × 0,8 |
| Velocidad efectiva en el aire | 2,53 kb/s | 416 bits en los 164 ms que dura el paquete (incluye preámbulo, cabecera y CRC del radio) |
| **Ocupación del canal (duty cycle)** | **16,4 %** | 164 ms de emisión por segundo |
| Aire libre para escuchar comandos | 836 ms por segundo | 1000 − 164 |
| Volumen de datos | 187 kB por hora; 468 kB en 2,5 h | 52 B × 3600 s y × 9000 s |
| Paquetes en el descenso (C3/C4) | 56 a 208 | Según las tolerancias de velocidad (est.) |
| Contador de 16 bits | Se desborda a los 65.536 paquetes (≈ 18 h) | Sobra para 2 h de espera más el vuelo |
| UART radio ⇄ notebook | 115.200 baud alcanza de sobra | El puente manda ≈ 130 caracteres por paquete (hex) ≈ 11 ms |

**Comparación con el cohete V2:** 65 B × 10 Hz = 5,2 kb/s. El CanSat necesita **12,5 veces menos** (0,416 kb/s). Por eso podemos bajar la velocidad y ganar alcance.

> **Nota sobre la ocupación.** La ocupación de un canal se calcula con el tiempo que dura la emisión (164 ms de cada 1000 ms), no dividiendo bits por velocidad nominal, porque el preámbulo y la cabecera también ocupan el aire. Con la trama de 65 B a SF7/BW250 del cohete, el mismo cálculo da 61,57 % (61,57 ms de cada 100 ms).

### 5.4 Presupuesto de enlace

**La idea.** Una señal sale del transmisor con cierta potencia, gana algo en las antenas, se debilita con la distancia y llega al receptor. Si llega con **más potencia que la mínima que el receptor puede entender (su sensibilidad)**, el enlace cierra. La diferencia entre las dos es el **margen**, y se mide en dB.

> **En criollo (dB y dBm):** los dB son una escala que suma y resta en lugar de multiplicar. 3 dB más es el doble de potencia y 10 dB más es diez veces. Los dBm son dB con referencia a 1 mW: 20 dBm = 100 mW; −100 dBm es una potencia ínfima, pero los receptores LoRa pueden trabajar con señales de ese orden. Un margen de 10 dB significa que sobra 10 veces la potencia mínima necesaria.

**Fórmula (est.):**

```
Potencia recibida = Ptx + G_CanSat + G_estación − Pérdida en el espacio libre − Pérdidas extra
Margen            = Potencia recibida − Sensibilidad
Pérdida en el espacio libre (dB) = 32,44 + 20·log10(distancia en km) + 20·log10(frecuencia en MHz)
```

**Supuestos** ([1.6](#16-supuestos-de-diseño)): antena del CanSat −3 dBi (peor caso), omni de la estación +3 dBi, pérdidas extra 6 dB (fuselaje, contenedor, polarización, conectores), frecuencia 921,5 MHz. Pérdida en el espacio libre: **91,7 dB a 1 km, 97,8 dB a 2 km, 101,3 dB a 3 km.**

**Sensibilidad del receptor (est.).** Con el modelo estándar de LoRa: `−174 + 10·log10(BW) + 6 dB (figura de ruido) + SNR mínimo del SF`. Para SF9 el SNR mínimo es −12,5 dB. Resultado: **BW125/SF9 = −129,5 dBm; BW250/SF9 = −126,5 dBm; BW500/SF9 = −123,5 dBm.**

> **Corrección a un dato del research.** En el research 01 (§9.1) la sensibilidad de BW250/SF9 figura como −132 dBm: la regla "baja 3 dB por cada duplicación del ancho de banda" se aplicó en el sentido contrario. Más ancho de banda deja entrar más ruido y empeora la sensibilidad. Usando −126,5 dBm, el margen a 3 km con la configuración elegida baja de ≈ 45 dB a ≈ 39 dB. La conclusión no cambia.

**Margen de enlace en el enlace de bajada (CanSat → estación), config. BW250/SF9 (est.)**

| Caso | 1 km | 2 km | 3 km |
|---|---:|---:|---:|
| **Base:** +20 dBm, estación omni de 3 dBi | 48,8 dB | 42,8 dB | **39,2 dB** |
| Potencia inicial +17 dBm | 45,8 dB | 39,8 dB | 36,2 dB |
| Estación con Yagi de 11 dBi, apuntada | 56,8 dB | 50,8 dB | 47,2 dB |
| **Mala orientación:** 15 dB extra de pérdida (CanSat tumbado o en el nulo del patrón) | 33,8 dB | 27,8 dB | **24,2 dB** |
| Respaldo BW125/SF9 (+3 dB de sensibilidad) | 51,8 dB | 45,8 dB | 42,2 dB |

**Lectura:**

- Hasta en el peor caso (24 dB a 3 km) pasamos con holgura el criterio de ≥ 10–15 dB. **Los ≈ 25–30 dB que sobran entre el caso base y ese criterio son nuestra reserva** para lo que el modelo no incluye: vegetación, terreno, reflexión en el suelo, desajuste de la antena y el material del fuselaje.
- **Aterrizaje:** con el CanSat y la estación a nivel del suelo, el modelo de espacio libre es optimista (reflexión en el suelo, obstáculos). Es esperable perder los últimos paquetes a pocos metros de altura. Las velocidades de descenso (C3, C4) se verifican con los paquetes anteriores.
- **Enlace de subida (comandos):** es simétrico. La estación transmite hasta +20 dBm con su antena y llega al CanSat con una potencia equivalente. Con una antena de más de 6 dBi hay que bajar la potencia conducida ([3.7](#37-normativa-argentina-que-afecta-la-elección)).
- **Si la normativa de baja potencia más restrictiva aplicara** (≈ −1 dBm de PIRE), perderíamos ≈ 18 dB (la PIRE del caso base es de ≈ 17 dBm) y quedarían ≈ 21 dB a 3 km: todavía cierra.

### 5.5 Energía del radio

Cuenta para las 2 h de E7 (más 0,5 h de vuelo), con 1 paquete por segundo y el módulo en recepción el resto del tiempo (est.).

| Concepto | Valor |
|---|---|
| Corriente en recepción (RX) | ≈ 12 mA [F] |
| Corriente en transmisión (TX) | 90 mA a +17 dBm; 120 mA a +20 dBm [F] |
| Tiempo de TX por segundo | 164 ms |
| **Corriente promedio** | **≈ 24,8 mA (+17 dBm)**; **≈ 29,8 mA (+20 dBm)** |
| **Energía en 2,5 h** | **≈ 62 mAh (+17 dBm)**; **≈ 74 mAh (+20 dBm)** |

Fórmula: `I_prom = I_RX · (1 − t_aire) + I_TX · t_aire`. **No incluye** microcontrolador, sensores, GPS ni cámara. Una celda 18650 típica tiene entre 2.000 y 3.000 mAh (dato general, verificar la elegida), de modo que la radio pesa del orden del 3 % de la batería. Lo que domina es la **recepción continua**, no la transmisión. Se los paso a Aviónica con el [pico de 120 mA](#38-advertencias-de-compra-y-configuración) y la tensión de 3,3 V.

### 5.6 Convivencia con otros equipos en el campo

Habrá otros CanSats transmitiendo en la misma banda. Medidas, en orden de importancia: (1) **canal asignado por CONAE** y verificado en la FRR; (2) **sync word y IDs propios**; (3) **paquete corto** (164 ms) para reducir la chance de superposición; (4) una **antena de estación direccional** (Yagi) rechazaría parte del ruido de otros equipos, pero hay que apuntarla ([6.3](#63-antena-de-la-estación-terrena)). Los módulos SX127x permiten además detectar actividad en el canal antes de transmitir (CAD): queda como mejora opcional, no la incluyo en la base.

---

## 6. Antenas

La antena transforma la señal eléctrica del módulo en onda de radio y viceversa. Es una de las piezas con más impacto en el alcance y de las más fáciles de arruinar sin darse cuenta.

### 6.1 Fundamentos en criollo

| Concepto | Qué es | Por qué importa acá |
|---|---|---|
| **Longitud de onda λ y antena λ/4** | La onda mide ≈ 325 mm a 921 MHz. Un "monopolo de cuarto de onda" es un alambre de λ/4 ≈ **81 mm** (en la práctica, 77–79 mm) | A 433 MHz, como en el cohete V2, medía 17,3 cm. A 915 MHz es la mitad: **cabe sin problemas** dentro del contenedor de 200–250 mm |
| **Plano de tierra** | Un monopolo necesita una superficie metálica debajo para irradiar bien (la masa de la placa, las baterías) | Dentro de un CanSat de Ø90 mm el plano es chico: esperá 0–2 dBi reales, no los 3 dBi de la etiqueta |
| **dBi** | Cuánto "concentra" la antena respecto de una ideal que irradia igual para todos lados | Más dBi = más alcance en una dirección, pero menos cobertura en las demás |
| **Patrón de radiación** | Una antena vertical irradia como una **dona** alrededor de su eje, con un **nulo** (casi cero) en la dirección del eje | Si el CanSat cuelga con la antena vertical, el nulo apunta al cielo y al suelo |
| **Polarización** | La orientación de la onda (casi todas las antenas baratas son verticales) | Si el CanSat se inclina un ángulo φ, se pierden 20·log10(cos φ) dB: 1,25 dB a 30°, 3 dB a 45°, 6 dB a 60° |
| **VSWR** | Mide cuánto de la señal "rebota" en vez de irradiarse | Meta: < 2 (idealmente < 1,5) en toda la banda 915–928 MHz. Se mide con un analizador de redes (NanoVNA) |

**Efecto del nulo en el descenso (est., modelo idealizado).** Para una estación a distancia horizontal *d* y un CanSat a altura *h*:

| d / h | Pérdida por patrón |
|---|---:|
| 4 | 0,3 dB |
| 2 | 1,0 dB |
| 1 | 3,0 dB |
| 0,6 | 5,8 dB |
| 0,3 | 10,8 dB |
| 0,1 | 20 dB |

**Regla práctica:** ubicar la estación a una distancia horizontal de al menos la mitad de la altura del CanSat (d/h ≥ 0,5), es decir, **no debajo de la zona de caída**. Si el CanSat pasa casi encima de la estación, cae en el nulo. El balanceo en el descenso rellena parcialmente el nulo.

### 6.2 Antena del CanSat

**Restricciones** (de los documentos): banda de X1; entra en Ø90 mm × 200–250 mm incluyendo los sistemas de descenso (S2, S3; las Bases dicen Ø95 × 200 mm, ver [11](#11-pendientes-y-consultas)); no puede sobresalir del contenedor, que debe deslizar libremente (S7); pared impresa en 3D ≥ 2 mm (S6); soporta 15 G y 30 G y va montada fija (S4, S5, S8); el CanSat rota en el descenso (X5).

**Derivadas (mías):** la antena va **dentro** del contenedor; evitar filamentos cargados con fibra de carbono o metal; el GPS puede degradarse cuando la radio transmite, así que se separan las antenas y se prueba el fix.

#### Las cuatro alternativas

| | **A1. Rubber duck λ/4 comercial** | **A2. Monopolo λ/4 de alambre** | **A3. Látigo corto helicoidal ("spring")** | **A5. Chip cerámico / SMD** |
|---|---|---|---|---|
| Qué es | Látigo flexible con conector SMA o RP-SMA, se enrosca | Alambre rígido de cobre o varilla de latón de ≈ 77–79 mm, soldado al módulo o a un conector | Helicoidal corta de 50–100 mm | Componente de 10 × 10 × 11 mm soldado a la placa |
| Ganancia: declarada → real esperada | 2–3 dBi → 1–2 dBi | — → 0–2 dBi (est.) | 2 dBi → 0–2 dBi | ≈ 2 dBi con placa grande; menos con placa chica |
| Largo | 100–110 mm (algunas 195–210 mm) | ≈ 77–79 mm | 50–100 mm | 10 mm |
| Masa | ≈ 9 g | < 1 g | pocos gramos | 0,7 g |
| Conector / montaje | SMA / RP-SMA; **necesita soporte mecánico** | Soldado o U.FL con pigtail; abrazadera impresa | SMA / U.FL con pigtail | Soldada, con red de adaptación |
| Pros | VSWR conocido; se compra local; barata | El más liviano y corto de los que irradian bien; totalmente fijable (S8); se ajusta recortando | Muy compacta; aguanta impacto | Diminuta; aguanta cualquier vibración |
| Contras | Come más de la mitad del largo; es una masa en voladizo: 9 g a 30 G ejercen ≈ 2,6 N (≈ 0,13 N·m sobre el conector); las articuladas violan S8 | **Hay que medirla** con un analizador de redes; la eficiencia depende del plano de tierra y de las baterías | Menos eficiente y más angosta; la ganancia declarada rara vez se cumple | Pide plano de tierra grande y diseño de placa; muy sensible a baterías y carcasa; no se consigue local |
| Precio en AR (est.) | $5.000–70.000 según listado | Prácticamente cero (alambre) | ≈ $16.500 (Whip 50 mm) a $59.000 (importada) | No hay oferta local |

_Precios: capturas de Mercado Libre, Frávega y Desertcart de fechas mezcladas (research 02 §4.8); reverificar el día de comprar._

#### Matriz de decisión (juicio mío, del research 02 §4.6)

| Criterio | Peso | A1 Rubber duck | A2 Alambre λ/4 | A3 Látigo corto | A5 Chip |
|---|---:|---:|---:|---:|---:|
| Cobertura del patrón en el descenso | 25 | 4 | 4 | 3 | 2 |
| Eficiencia / ganancia real | 15 | 4 | 4 | 2 | 1 |
| Robustez mecánica (15 G / 30 G) | 15 | 2 | 4 | 4 | 5 |
| Espacio y masa | 15 | 2 | 4 | 5 | 5 |
| Facilidad de integración y ajuste | 10 | 4 | 3 | 4 | 2 |
| Costo y disponibilidad en Argentina | 10 | 5 | 5 | 4 | 3 |
| Resistencia al desajuste (baterías, carcasa) | 10 | 3 | 3 | 2 | 1 |
| **Total (sobre 100)** | | **68** | **78** | **68** | **55** |

**Descartadas:** dipolo λ/2 de ≈ 160 mm (63 puntos, ocupa demasiado largo), antena impresa en la placa (56, ≈ −1 dBi), parche (45, direccional con orientación incierta) y antena desplegable (agrega un mecanismo que puede no abrir o enredarse con el paracaídas).

#### Elección: A2, monopolo λ/4 de alambre, con A1 como referencia y plan B

- **Principal: A2**, de ≈ 80 mm (empezar en 80 mm y recortar midiendo para centrar el VSWR en ≈ 921 MHz), vertical, a lo largo del eje, dentro del contenedor.
- **Plan B y referencia de ensayo: A1.** Comprar una igual para tener un enlace conocido contra el cual comparar la A2 en las pruebas de alcance. **Si el equipo no consigue acceso a un analizador de redes, la A1 pasa a ser la opción más segura** (el research 02 lo señala explícitamente).
- **Comprar 2 o 3 unidades** de cada una: una queda para ensayos destructivos de vibración e impacto.

> **Sobre los "10 dBi" de los látigos baratos.** En Mercado Libre y Amazon abundan látigos de 13–19 cm que dicen 10 dBi. Para llegar a ≈ 10 dBi con una omnidireccional a 915 MHz hace falta del orden de 1 m de largo; un látigo de 19 cm da 2–3 dBi. **Contá con 2–3 dBi.**

#### Reglas de montaje

1. **Eje vertical y plano de tierra:** la antena sale del borde de la placa, perpendicular a ella, con la placa lo más grande posible como plano de tierra.
2. **Alejada de metales:** al menos ≈ 30 mm de las baterías 18650 y de cualquier masa metálica. Las baterías, en el extremo opuesto.
3. **Extremo del CanSat:** el lado del paracaídas está libre de metales, pero hay que cuidar que las cuerdas no se enreden en la antena. El lado inferior suele tener cámara, baterías y servos, que son metálicos.
4. **Fijación mecánica (S4, S5, S8):** abrazadera impresa o soporte sobre la placa; no depender solo del cordón de soldadura ni del conector; dejar un alivio de tensión.
5. **Conector:** U.FL a SMA o pigtail corto. Cada conector suma ≈ 0,1–0,2 dB y los cables finos pierden del orden de 1 dB por metro a 900 MHz: usar el más corto posible.
6. **Material del contenedor:** PLA o PETG simples. Avisar a Estructuras.
7. **GPS:** antena de radio y antena de GPS separadas; probar el fix con la radio transmitiendo.
8. **Mantener polarización vertical** con el CanSat colgado.

### 6.3 Antena de la estación terrena

**Restricciones:** de mesa o portátil (G11); portátil, a lo largo de la línea de vuelo y sin corriente alterna (G10), así que solo antenas pasivas; incluida en el equipo con la notebook y la radio (G9); banda de X1; no cuenta para C9.

**Derivadas (mías):** la estación transmite poco (solo comandos), así que **el enlace crítico es el de bajada**. El CanSat se mueve: baja de 500 m–1 km en 1–3 minutos y deriva con el viento, así que una antena que hay que apuntar le suma una tarea operativa a alguien del equipo.

#### Las cuatro alternativas

| | **S1. Omni de baja ganancia** | **S2. Omni colineal de fibra de vidrio** | **S3. Yagi de 3 elementos** | **S4. Yagi de 6–10 elementos** |
|---|---|---|---|---|
| Ganancia | 2–3 dBi (se puede subir a 5) | 5–8 dBi (declarado) | 7–8 dBi | 10–15 dBi |
| Cobertura | 360° horizontal; ≈ 78° vertical | 360° horizontal; vertical ≈ 30° (5 dBi) o 15–20° (8 dBi) | Haz ancho (≈ 100–130°) | Haz de 40–55°: **hay que apuntarla** |
| Largo | 9–21 cm | 0,5–1 m | ≈ 0,3–0,4 m (est.) | 0,7–1,4 m |
| Masa | 10–30 g | ≈ 100–400 g (est.) | < 300 g | 0,5–0,8 kg |
| Conector / montaje | SMA / RP-SMA; mástil o trípode con extensión | N o SMA; mástil | N / SMA; trípode | N / SMA / FME; trípode con rótula |
| Alcance (est.) | 1–3 km con margen a 915 MHz | 3–5 km | 3–5 km | 5 km o más, apuntada |
| Pros | Sin apuntar; simple; barata | Más ganancia sin apuntar; robusta | Ganancia útil sin apuntar fino | +8 a +13 dB sobre una omni; rechaza otros equipos |
| Contras | No rechaza interferencia; nulo hacia el cenit | Mayor ganancia = haz vertical más angosto: el CanSat puede quedar fuera a mucha elevación | Se vende poco en Argentina; DIY requiere medir | Incómoda de portar; oferta pobre a 900 MHz en Argentina |
| Precio visto (AR$, est.) | ≈ $5.000–70.000 | $25.000 (captura vieja) a $398.000 | n/d | Yagi 10 dBi sin precio legible; TerraWave 15 dBi $460.418 sin stock |

#### Matriz de decisión (juicio mío, del research 02 §5.6)

| Criterio | Peso | S1 | S2 | S3 | S4 |
|---|---:|---:|---:|---:|---:|
| Cobertura con CanSat móvil y rotando | 25 | 4 | 3 | 3 | 2 |
| Ganancia / margen de enlace | 20 | 2 | 3 | 3 | 5 |
| Simplicidad operativa (sin apuntar) | 15 | 5 | 5 | 3 | 2 |
| Portabilidad (G10, G11) | 10 | 5 | 3 | 4 | 2 |
| Rechazo de interferencia de otros equipos | 10 | 1 | 1 | 3 | 5 |
| Costo y disponibilidad en Argentina | 10 | 5 | 3 | 3 | 3 |
| Cumplimiento (límite de PIRE) | 10 | 5 | 5 | 5 | 4 |
| **Total (sobre 100)** | | **75** | **66** | **66** | **64** |

La combinación omni + Yagi (S1 + S4) puntúa 80, pero depende de que G9 lo permita (dice "una antena", en singular).

#### Elección: S1, omnidireccional vertical de 2–5 dBi

- **Principal: S1**, en mástil o trípode de **2–3 m**, con el coaxil más corto posible (RG58 pierde ≈ 0,5–0,6 dB por metro a 900 MHz) y alejada de la notebook y de las personas (al menos 1 m). **No pasar de ≈ 5 dBi** para no angostar demasiado el haz vertical (ver S2).
- **Por qué:** a 915 MHz el margen alcanza sin apuntar (24 dB a 3 km en el peor caso, [5.4](#54-presupuesto-de-enlace)), no suma una tarea operativa y cumple con holgura el límite de PIRE.
- **Opcional: S4** (Yagi de 10–11 dBi) operada por un integrante, para ganar ≈ 8 dB y filtrar interferencia. **Consulta pendiente a CONAE por G9:** ¿se pueden llevar dos antenas? Alternativa sin riesgo: llevar la Yagi como repuesto o para ensayos y declarar solo la omni. Si se usa, la potencia conducida máxima permitida es 25 dBm y trabajamos con ≤ 20 dBm: cumple.
- **Ubicación:** a cierta distancia lateral de la zona de caída (ver [6.1](#61-fundamentos-en-criollo)) y, si es posible, sobre un mástil para reducir pérdidas por el suelo.

### 6.4 El gran desconocido: el fuselaje del cohete

G1 pide calibrar la altitud en la plataforma de lanzamiento, y ahí el CanSat ya está **dentro del cohete** (C1). El enlace debe funcionar a través del fuselaje, cuyo material no conocemos. Fibra de vidrio y cartón son casi transparentes a 915 MHz. **Fibra de carbono y piezas metálicas pueden bloquear la señal** y volver imposible el comando de calibración aunque todo lo demás esté bien. Es la consulta a CONAE más importante de este documento ([sección 11](#11-pendientes-y-consultas)). Mientras no haya respuesta, hay que medir cuántos dB se pierden con un tubo de fibra de vidrio y uno de cartón ([sección 9](#9-plan-de-ensayos)).

### 6.5 Qué comprar (antenas y cables)

| Ítem | Cantidad | Uso |
|---|---:|---|
| Antena rubber duck 915 MHz, 2–3 dBi, SMA o RP-SMA (el mismo conector que el módulo) | 3 | Referencia del CanSat, antena de la estación, repuesto/ensayo destructivo |
| Alambre de cobre rígido o varilla de latón (Ø 1–1,5 mm) | ≈ 20 cm | Monopolo λ/4 (A2), con material de sobra para recortar |
| Pigtail U.FL a SMA | 3 | Conexión módulo ⇄ antena |
| Mástil o trípode de 2–3 m + coaxil corto (1–2 m) + adaptadores | 1 juego | Antena de la estación |
| Analizador de redes (NanoVNA) o acceso a laboratorio | — | Medir VSWR; universidades con laboratorio de RF y radioaficionados (Radio Club Argentino) suelen ayudar |
| Yagi 10–11 dBi (opcional) | 1 | Segunda antena de estación, si G9 lo permite |

Antenas y conectores del CanSat cuentan para C9. El conector (SMA vs RP-SMA) hay que preguntarlo siempre: **no son intercambiables** y las antenas "Wi-Fi" suelen ser RP-SMA.

---

## 7. Estación terrena

### 7.1 Qué tiene que hacer la estación

| Req. | Qué implica |
|---|---|
| G4 | Cada equipo desarrolla su propia estación terrena |
| G5 | Mostrar la telemetría en formato texto, en ascenso y descenso |
| G6 | Unidades SI indicadas en pantalla |
| G7 | Gráficos en tiempo real de altitud, voltaje, corriente, acelerómetro y tasas de rotación |
| G8 | Mostrar tiempo de misión, temperatura, posición GPS, paquetes recibidos, paquetes perdidos y estado del software de vuelo |
| G12 | Fuentes de al menos 14 pt, trazos y ejes de gráficos en negrita, texto oscuro sobre fondo claro |
| G13 | Todo en una sola interfaz, simultáneo, sin pestañas |
| G2 | CSV con todos los datos de sensores, guardados en la notebook y en un disco de respaldo |
| G14 | Contar los paquetes recibidos con éxito (distinto del contador de transmitidos) |
| G15 | Enviar telecomandos desde la estación |
| G9, G10, G11 | Notebook con ≥ 2 h de batería, radio y antena incluidas; portátil; sin corriente alterna |
| Bases §6 | La FRR hace una prueba de estación: telemetría en vivo, gráficos y despliegue funcionando |

### 7.2 Hardware de la estación: interfaz radio ⇄ notebook

Hay que resolver cómo llegan los bytes de la radio a la notebook y cómo salen los comandos.

| Opción | Cómo se conecta | Pros | Contras | Decisión |
|---|---|---|---|---|
| A. Módulo UART + adaptador USB-UART | Módulo (E32, XBee, RFD900x) → chip CP2102N/CH340 → USB | La más simple; poco firmware | No preserva los límites de paquete (hay que enmarcar); límites propios del módulo (58 B en E32) | Solo si se usa el plan B de radio |
| **B. Módulo SPI + microcontrolador puente con USB** | RFM95W → ESP32-C3 (u otro) → USB (puerto serie virtual) | Control total; entrega RSSI/SNR por paquete; reutiliza firmware; mismo módulo que el CanSat | Hay que escribir y depurar el firmware puente (≈ 100 líneas) | **Elegida** |
| C. Placa integrada MCU + LoRa con USB | Heltec LoRa32 V3 (≈ $63.000), LilyGo T-Beam 915 (≈ $83.000) | Hay stock local; ya trae USB; sin soldar | Potencia y antena limitadas (≈ 100 mW); el Heltec V3 usa SX1262 (compatible en el aire, pero otra librería) | Plan B de hardware |
| D. 2,4 GHz con ESP32 (ESP-NOW) | ESP32 → USB | Unicast por MAC nativo | Banda descartada ([3.6](#36-alternativas-descartadas)) | Descartada |

**Elegida: B.** El equipo ya usa **ESP32-C3 SuperMini** en el cohete V2; si Aviónica no los necesita, sirven como puente. El puente:

1. Recibe cada paquete LoRa y lo manda a la notebook como una **línea de texto** con RSSI, SNR y los bytes en hexadecimal (fácil de depurar con un monitor serie, sin necesidad de marcar los límites de paquete).
2. Recibe de la notebook un comando y lo **retiene hasta recibir la siguiente telemetría válida; entonces lo transmite de inmediato**. Esto resuelve la regla de oro de la [sección 2.5](#25-ciclo-de-1-segundo-half-duplex): el puente reacciona en milisegundos y no depende de la latencia del sistema operativo.

**Contrato de la interfaz USB** (115.200 baud, 8N1, una línea por mensaje, **TBR**):

| Sentido | Mensaje | Significado |
|---|---|---|
| Puente → notebook | `RX,<rssi_dBm>,<snr_dB>,<hex>` | Paquete recibido (sin validar: el CRC lo valida el software de la estación) |
| Puente → notebook | `SENT,<ms>` | Se transmitió el comando retenido |
| Puente → notebook | `INFO,<versión>,<frec_Hz>,<BW>,<SF>` | Al arrancar: versión de firmware y parámetros de radio |
| Notebook → puente | `CMD,<hex>` | Encolar un comando (se reemplaza el pendiente) |
| Notebook → puente | `CFG,<frec_Hz>` | Cambiar de canal sin reprogramar |

### 7.3 Energía de la estación

No hay corriente alterna (G10). Con un módulo de ≤ 100 mW la energía **no es un problema**:

| Elemento | Consumo |
|---|---|
| Radio RFM95W en RX | ≈ 12 mA [F] |
| Radio RFM95W en TX a +20 dBm (solo comandos esporádicos) | 120 mA [F] |
| Microcontrolador puente (ESP32-C3) | sin dato verificado: **medir** con un tester USB |

- Un puerto **USB 2.0** entrega 500 mA por norma (100 mA antes de configurar el dispositivo) y uno USB 3.x 900 mA [F]. Con el módulo de 100 mW **sobra margen** y no hace falta fuente externa ni hub con alimentación propia: **la radio sale de la batería de la notebook**, que es lo más seguro frente a la lectura de G9 y G10.
- El problema de picos de corriente aparece con radios de 1 W (RFD900x, E32-T30D, ≈ 0,8–1 A): otra razón para no elegirlas.
- Lo que más consume la batería de la notebook es la **pantalla a brillo máximo** (G12 exige leerla con sol), el procesador y el disco de respaldo. **Hay que medir la autonomía real con el software corriendo 2 h a brillo máximo.**

**Buenas prácticas del lado de la notebook (verificar con pruebas):**

- Cable USB corto y de buena calidad: la caída de tensión en un cable largo o fino baja los 5 V y reinicia el módulo.
- **Deshabilitar la suspensión selectiva de USB** en Windows para que no corte el puerto durante el vuelo.
- Que el software busque el dispositivo por **VID/PID o número de serie**, no por "COM3" (el número cambia al cambiar de puerto).
- Preferir un puerto USB 2.0 o un alargue (el USB 3.x emite ruido que degrada receptores sobre todo en 2,4 GHz; a 915 MHz es un problema menor, pero se evita con el coaxil y la distancia).
- **Sincronizar el reloj de la notebook** por NTP antes de salir al campo: el comando de hora (F3) sale de ahí y en el campo no hay internet.
- Para el disco de respaldo (G2) preferir un **pendrive o SSD** en vez de un disco rígido mecánico, que consume corriente del mismo bus.

### 7.4 Software de la estación: cuatro alternativas

| | **E1. Reutilizar y adaptar la estación existente (Python/FastAPI + interfaz web)** | **E2. Estación propia nueva (Python + PySide6 + pyqtgraph + pyserial)** | **E3. Dashboard genérico (Serial Studio)** | **E4. Navegador puro con Web Serial API** |
|---|---|---|---|---|
| Idea | Backend FastAPI que lee el puerto serie y una interfaz web que consume sus datos | App de escritorio con una ventana propia | Herramienta configurable, nacida en competencias CanSat | Una página web que lee el puerto serie directo, sin servidor |
| Una sola pantalla sin pestañas (G13) | ⚠️ posible en web; **verificar** que hoy no use pestañas | ✅ `QMainWindow` con grilla, sin `QTabWidget` | ✅ | ✅ |
| Fuentes y estilos (G12) | ✅ con CSS (`font-size: 14pt`, trazos y ejes en negrita); a adaptar | ✅ control total | ⚠️ limitado a sus widgets | ✅ con CSS |
| Gráficos en tiempo real (G7) | ✅ (verificar la librería usada) | ✅ pyqtgraph es rápido | ✅ | ✅ |
| Lectura del puerto serie | ✅ en el backend (pyserial); verificar | ✅ pyserial | ✅ | ⚠️ solo Chrome/Edge 89+; Safari no |
| CSV a dos destinos (G2) | ⚠️ a agregar en el backend | ✅ manual | ⚠️ guarda CSV, un solo destino | ⚠️ difícil de escribir a dos destinos |
| Comandos desde la misma pantalla (G15) | ⚠️ a agregar (botones + endpoint) | ✅ botones propios | ⚠️ limitado | ✅ |
| Funciona sin internet | ⚠️ **verificar** que no cargue librerías desde un CDN | ✅ | ✅ | ✅ si las librerías van locales |
| Consumo de energía | ⚠️ servidor + navegador: más que una app nativa (verificar con la prueba de 2 h) | Bajo a medio | Medio | Medio |
| Esfuerzo | Bajo a medio | Alto | Bajo | Medio |
| **Cumple G4** (desarrollo propio) | ⚠️ **riesgo medio-alto: consultar a CONAE** | ✅ | ❌ riesgo alto (herramienta de terceros; build open-core) | ✅ si es código propio |
| Riesgo principal | G4 y calidad desconocida del código | Tiempo de desarrollo | G4, licencia y G12 | Compatibilidad de navegador y CSV doble |

Referencias: research 03 §5.2–5.3. Los ✅ y ⚠️ de E1 son una **hipótesis**: todavía no revisé el código de la estación existente.

#### Elección: E1 con un núcleo propio, y E2 como plan de respaldo

La decisión del equipo es reutilizar la estación que ya está hecha (Python/FastAPI con interfaz web) porque ahorra mucho tiempo. **El riesgo es G4: "Cada equipo deberá desarrollar su propia estación terrena".** Una estación hecha por otro equipo y reutilizada puede no contar como desarrollo propio. Por eso propongo reutilizar **solo la estructura** y dejar el corazón como desarrollo nuestro:

| Parte | Quién la desarrolla | Qué es |
|---|---|---|
| Firmware del CanSat y del puente | **Propio** | [Anexo A](#anexo-a-código-de-referencia) |
| Protocolo, trama y comandos | **Propio** | [Sección 4](#4-trama-de-telemetría-y-telecomandos) |
| **Núcleo `cansat_link` (Python): lector del puente, decodificador, CRC, contador de pérdidas, escritor de CSV, codificador de comandos** | **Propio** | [Anexo A.4](#a4-decodificador-y-contador-de-pérdidas-python-estación) |
| Adaptación de la interfaz (G6, G7, G8, G12, G13, G15) | **Propio** | Cambios sobre la interfaz existente |
| Simulador de telemetría | **Propio** | Reproduce un CSV de descenso con ruido, pérdidas y errores de CRC |
| Servidor y componentes web base | Reutilizado del otro equipo | Con permiso escrito |

```
 RFM95W -> ESP32-C3 puente --USB--> [ notebook ]
                                      |
         lector serie (cansat_link) ->|-> decodificador + CRC ->  estado compartido
                                      |        |-> CSV principal (notebook)
                                      |        |-> CSV de respaldo (pendrive o SSD)
                                      |        |-> registro crudo (hex) y tramas con CRC inválido
                                      |        '-> WebSocket -> interfaz web (una sola pantalla)
 Interfaz web: botones de comandos -> POST /api/comando -> cansat_link -> serie -> puente -> LoRa
```

**Condiciones para que esta decisión sea segura:**

1. **Consultar a CONAE por G4** antes del PDR: "¿se puede reutilizar y adaptar la estación terrena desarrollada por otro equipo si el decodificador, los comandos y la interfaz de misión son desarrollo propio?".
2. **Permiso escrito o licencia** del otro equipo para reutilizar su código.
3. **El núcleo `cansat_link` es independiente de la interfaz.** Si CONAE dijera que no, cambiar la interfaz por una propia (E2) cuesta poco: lo difícil (decodificar, contar, guardar y mandar comandos) ya está hecho.
4. **Documentar en el PDR** qué se reutiliza y qué se desarrolló.

#### Datos a completar sobre la estación existente

| Dato | Valor |
|---|---|
| Equipo / autor y año | _(completar)_ |
| Repositorio y licencia, o permiso de uso | _(completar)_ |
| ¿Cómo recibe hoy los datos? (puerto serie, archivo, MQTT, otro) | _(completar)_ |
| Formato de datos que espera (texto, JSON, binario) | _(completar)_ |
| Librería de gráficos y si carga recursos de internet (CDN) | _(completar)_ |
| ¿Ya guarda CSV? ¿A cuántos destinos? | _(completar)_ |
| ¿Ya permite enviar comandos? | _(completar)_ |
| ¿Usa pestañas o paneles ocultos? (G13) | _(completar)_ |
| Navegador y resolución con que se probó | _(completar)_ |

#### Qué hay que adaptar, requisito por requisito

| Req. | Qué hacer |
|---|---|
| **G13** | Una sola página; todos los gráficos y los indicadores visibles a la vez; **sin pestañas ni paneles que se oculten** |
| **G12** | `font-size` de al menos **14 pt en todo** (14 pt ≈ 18,7 px), incluidos ejes, leyendas y botones; trazos gruesos y ejes en negrita; fondo claro y texto oscuro. Probar a la resolución real de la notebook (por ejemplo 1366 × 768): con 14 pt, 5 gráficos y 12 indicadores **el espacio es justo** |
| **G7** | 5 gráficos: altitud, tensión, corriente, aceleración (3 curvas X/Y/Z) y velocidad angular (3 curvas) |
| **G8** | Panel de texto grande: tiempo de misión, temperatura, GPS (latitud, longitud, altitud, satélites), paquetes recibidos, paquetes perdidos y estado del software de vuelo |
| **G6** | Unidad SI **junto a cada valor y en cada eje**: m, Pa, °C, V, A, m/s², rad/s. Consultar a CONAE si se acepta °C (el SI usa K) |
| **G5** | Tabla de texto con los campos decodificados en tiempo real, más un registro crudo en hexadecimal para depurar |
| **G2** | El **backend** escribe dos archivos CSV (notebook y disco de respaldo) con `flush()` y `os.fsync()` por línea. Si el disco de respaldo se desconecta, sigue escribiendo en el principal y avisa en pantalla |
| **G15** | Botones en la misma pantalla (calibrar altitud, configurar hora, pedir batería y paneles, pedir información), con el estado de cada comando (enviado, confirmado, error) |
| **G14 / G8** | Contar `rx_ok` solo con CRC válido; perdidos con el contador de la trama ([4.8](#48-contadores-pérdidas-y-reinicios)) |
| **Sin internet** | **Todas las librerías JavaScript y CSS deben ir locales** (nada de CDN): en el campo no hay red |
| **Calidad del enlace** | Mostrar RSSI y SNR del último paquete; alarma de "SIN SEÑAL" si pasan 3 s sin paquetes válidos (**TBR**) |
| **Reconexión** | Detectar el puente por VID/PID y reconectar solo cuando se desenchufa el USB |
| **Arranque** | Entorno virtual armado y una copia offline de las dependencias, probado en una notebook "limpia"; versiones fijadas en `requirements.txt` |

---

## 8. Gestión de fallos y calidad de datos

La estructura de esta sección sigue la del documento COMMS V2, adaptada al CanSat.

| # | Fallo | Cómo se detecta | Qué hace el sistema | Cómo se prueba |
|---|---|---|---|---|
| 1 | **Sensor sin respuesta o lectura inválida** | Aviónica evalúa los códigos de retorno y rangos | Flag ERROR_SENSOR en la trama; mantiene el último valor seguro | Desconectar físicamente el bus del sensor en banco |
| 2 | **GPS sin posición válida** | Sin fix o sin satélites | GPS_FIX = 0 y satélites = 0 (o último valor válido); alerta visible en la estación | Tapar la antena GPS / probar en interior |
| 3 | **Trama con CRC inválido o tamaño distinto de 52 B** | La estación recalcula el CRC sobre los 50 primeros bytes | Se descarta, **no suma a `rx_ok`**, y se guarda en un archivo aparte (`crc_fail`) para análisis | Inyectar errores de bits en un simulador |
| 4 | **Paquetes perdidos, duplicados o fuera de secuencia** | Salto en el contador de 16 bits | Se cuentan los perdidos; los duplicados y los fuera de secuencia se ignoran | Simular pérdida del 10 % |
| 5 | **Reinicio del microcontrolador o del módulo de radio** | El contador y el estado continúan (F1, F5); el flag REINICIO se activa | La estación registra el evento; el contador puede saltar unas unidades por el esquema de escritura espaciada | Reset manual durante una simulación de vuelo |
| 6 | **Radio del CanSat colgada** (bus SPI) | Aviónica: watchdog | Reinicio del microcontrolador y reconfiguración del módulo | Provocar la falla en banco |
| 7 | **Pérdida de enlace** | 3 s sin paquetes válidos (**TBR**) | Alarma visual "SIN SEÑAL"; al volver un paquete válido, la recepción se reanuda sola | Alejar los módulos o tapar la antena hasta perder y recuperar el enlace |
| 8 | **Comando perdido o no ejecutado** | El eco en la telemetría no refleja la secuencia enviada | Reintento hasta 10 veces y aviso en pantalla | Enviar comandos a distancia y bloquear algunos |
| 9 | **Puente USB desconectado** | El puerto serie desaparece | La estación sigue grabando lo que tiene, avisa y reconecta por VID/PID | Desenchufar y enchufar el cable en caliente |
| 10 | **Disco de respaldo desconectado** (G2) | Falla de escritura | Sigue grabando en el disco principal y avisa | Desenchufar el pendrive durante la prueba |
| 11 | **Interferencia de otros equipos** | RSSI alto sin paquetes válidos; paquetes con ID ajeno | Se descartan por ID, sync word y CRC; el canal asignado lo evita de origen | Prueba con dos transmisores en el campo |
| 12 | **Notebook sin batería** | Autonomía medida < 2 h | Cargar al 100 %, brillo máximo medido, modo de energía que no baje el brillo | Prueba de autonomía de 2 h |
| 13 | **Desacuerdo entre datos recibidos y registro a bordo** (si Aviónica graba una SD) | Comparación post-vuelo de logs | Se toma el registro de a bordo como referencia y se documenta el % de pérdida telemétrica | Ensayo dinámico con ambos logs |

---

## 9. Plan de ensayos

Los criterios de aceptación son **propuestas mías (TBR)**.

| # | Ensayo | Qué se mide | Criterio propuesto | Req. |
|---|---|---|---|---|
| 1 | **Tiempo en el aire real** del paquete (analizador lógico sobre DIO0 o AUX) | Duración de la emisión | 164 ms ± 5 % | X4, C5 |
| 2 | **VSWR / S11** con NanoVNA, con el CanSat armado (baterías, cámara, contenedor cerrado). Repetir con la mano cerca y con el paracaídas guardado | Adaptación de la antena | < 2 (ideal < 1,5) en 915–928 MHz | X1 |
| 3 | **Patrón de radiación**: CanSat girando sobre un soporte no metálico a 20–50 m, registrando RSSI cada 15° en dos planos | Nulos y cobertura | Verificar el nulo del eje ([6.1](#61-fundamentos-en-criollo)) | X5 |
| 4 | **Alcance en línea de vista** a 500 m, 1 km y 2–3 km, con el CanSat en alto | % de paquetes recibidos, RSSI, SNR | ≥ 95 % recibidos a 2 km | X4 |
| 5 | **Atravesar el fuselaje**: CanSat dentro de un tubo de fibra de vidrio, de cartón y, cuando CONAE lo informe, del material del cohete | dB perdidos y si pasan los comandos G1 y F3 | Comando de calibración confirmado a través del tubo | G1, F3 |
| 6 | **Comandos a distancia**: `CAL_ALT_ZERO` y `SET_TIME` | Eco en la telemetría; error de hora respecto de UTC | Confirmación en ≤ 3 telemetrías; error de hora < 1 s | G1, F3, F5 |
| 7 | **Vibración 15 G y choque 30 G** con revisión visual y de VSWR antes y después | Integridad mecánica | VSWR sin cambio apreciable; enlace sin degradación | S4, S5, S8 |
| 8 | **GPS con la radio transmitiendo a 1 Hz** | Mantenimiento del fix | Fix sostenido | X5 |
| 9 | **Antena A2 vs. A1**: misma prueba de alcance | Diferencia de RSSI | A2 no peor que A1 menos 3 dB | |
| 10 | **Consumo del módulo con la batería elegida** durante 2 h más el vuelo | Corriente y energía | Dentro de las estimaciones de [5.5](#55-energía-del-radio) | E7 |
| 11 | **Simulador**: inyectar errores y pérdidas, reiniciar el CanSat en medio de la prueba | `rx_ok`, perdidos, continuidad del contador | Dos corridas de descenso de 200 s con 10 % de pérdida, contadores correctos | F1, G8, G14 |
| 12 | **Estación**: autonomía de 2 h a brillo máximo; lectura a pleno sol; fuentes ≥ 14 pt; sin pestañas; desconexión de USB y del disco de respaldo | Autonomía y recuperación | Se cumple y se recupera sola | G9, G12, G13, G2 |
| 13 | **Corriente del puente USB** con un tester; probar puertos USB 2.0, 3.x y USB-C | Corriente de pico | Ningún puerto corta | G9, G10 |
| 14 | **Filtrado por ID**: la estación ignora paquetes de otro equipo | Descarte correcto | 100 % descartados | X2 |
| 15 | **Prueba de estación de la FRR en seco**: telemetría en vivo, gráficos y despliegue, con el simulador y luego con la radio real, en la notebook real de la competencia | Funcionamiento completo y canal asignado | Todo funciona | Bases §6 |

---

## 10. Riesgos

| Riesgo | Mitigación y ensayo |
|---|---|
| **El cohete o el contenedor atenúan la señal en la plataforma** (fibra de carbono o metal) | Consultar a CONAE; ensayo 5; si hace falta, mayor potencia o plan alternativo para G1 |
| Canales por defecto en 902–915 MHz **fuera de X1** | Fijar la frecuencia en 916–927 MHz por código y verificarla con un SDR o analizador de espectro |
| **Modo broadcast activo por defecto** (módulos UART: dirección 0x0000/0xFFFF) | Con el módulo elegido no aplica; ID de destino y de origen en cada paquete; documentar en el PDR |
| Antena mal ajustada (VSWR alto) | NanoVNA; ensayo 2; comprar A1 de referencia |
| Rotación del CanSat y nulo del patrón | Ensayo 3; ubicar la estación a distancia lateral |
| **G4: estación reutilizada puede no contar como propia** | Núcleo propio; consulta a CONAE; plan de respaldo E2 |
| **G9: "una antena"** y la Yagi opcional | Consulta a CONAE; declarar solo la omni si no hay respuesta |
| Pérdida de paquetes al aterrizar | Registrar RSSI; asumir la pérdida de los últimos metros |
| Colisión con otros equipos | Canal asignado; sync word e IDs propios; paquete corto |
| Picos de corriente reinician el microcontrolador | Condensadores de desacople y regulador con capacidad de pico |
| Plazos de entrega | Comprar ya: módulos de revendedores internacionales demoran semanas; XBee y RFD900x, meses |
| Vibración y golpe | Soldar el módulo, fijar el cable de antena, alivio de tensión |
| Mensaje de inicio de misión perdido (X3) | Flag repetido en 5 paquetes seguidos |
| Interfaz web que no entra en pantalla con 14 pt | Probar temprano a la resolución real de la notebook |
| Interfaz web que carga recursos de internet | Librerías locales; probar con la red apagada |
| Contador F1 y desgaste de memoria flash | FRAM o escritura espaciada; asumir saltos tras un reinicio |
| Régimen regulatorio de baja potencia más restrictivo | Consulta a ENACOM/CONAE; el margen ([5.4](#54-presupuesto-de-enlace)) tolera bajar la potencia |

---

## 11. Pendientes y consultas

Las consultas a CONAE se hacen por mail a **cansat.universitario@conae.gob.ar** durante el período de inscripción: **verificar si todavía estamos a tiempo.**

### 11.1 Consultas a CONAE

| # | Consulta | Impacta en |
|---|---|---|
| 1 | **Material y diámetro interior del fuselaje del cohete** | Enlace en plataforma (G1) y ascenso |
| 2 | **G4:** ¿se puede reutilizar y adaptar una estación terrena de otro equipo si decodificador, comandos e interfaz son propios? | Sección 7 |
| 3 | **G9:** ¿se admiten dos antenas en la estación (omni + Yagi)? | 6.3 |
| 4 | ¿Hay que transmitir antes de la separación (G5, X3, E7)? ¿Se puede transmitir durante las 2 h en plataforma? | 2.4 |
| 5 | **X3:** ¿qué es el "comando" que la telemetría envía al separarse, y a quién? | 4.4 |
| 6 | **X2:** ¿qué entienden por "modo de difusión"? ¿Alcanza con unicast y canal asignado? | 4.9 |
| 7 | **Asignación de frecuencias:** cuántos equipos vuelan en simultáneo y cómo se asigna el canal | 5.1, 5.6 |
| 8 | **Normativa:** ¿rige la Res. 581/2018 (30 dBm conducidos / 36 dBm de PIRE) o el régimen de dispositivos de baja potencia (50.000 µV/m a 3 m)? ¿Se exige homologación (ENACOM-Q2-60.14) a un prototipo? | 3.7 |
| 9 | **Ubicación de las estaciones** respecto de la plataforma y de la línea de vuelo; distancia máxima a la zona de caída en Falda del Cañete (Bases §7) | 1.6, 6.1 |
| 10 | **Altitud máxima real:** ¿500 m o 1 km? | 1.6 |
| 11 | **Dimensiones:** Ø95 × 200 mm (Bases) vs. Ø90 × 200–250 mm (Anexo). Diseño para Ø90 × 200 mm, el único valor que cumple ambos | 6.2 |
| 12 | **G6:** ¿se aceptan °C (en lugar de K)? | 7.4 |
| 13 | **G13:** ¿una barra de desplazamiento o un diálogo aparte para configuración cuenta como "pestaña"? | 7.4 |
| 14 | **G2:** ¿qué se considera "disco de respaldo"? ¿Vale un pendrive? | 7.3 |
| 15 | **Voltaje y corriente:** G7 los grafica en tiempo real, X6 los pide por telecomando. Los dos: en cada paquete y por `REQ_BATT_SOLAR`. ¿Se acepta? | 4.3, 4.7 |
| 16 | **C4:** "80 % de la altitud máxima": ¿80 % de la altura o del recorrido? Solo cambia la cantidad de paquetes | 5.3 |

### 11.2 Para los demás subsistemas

| Subsistema | Qué necesito |
|---|---|
| **Aviónica** | Modelo de microcontrolador y pines SPI libres; tensión de 3,3 V con pico de 120 mA y capacidad para 2,5 h; **detección de la separación** y máquina de estados ([2.4](#24-estados-del-cansat-y-cuándo-transmite)); **contador persistente** (F1, F5); origen del tiempo de misión; rango y escalado reales de los sensores ([4.3](#43-trama-de-telemetría-elegida-52-bytes)); qué es la "señal de paneles solares" (X6); posición de las baterías y del GPS |
| **Estructuras** | Material del contenedor (PLA o PETG simples); espacio y soporte de la antena; ubicación de baterías metálicas; si el CanSat se separa en dos cuerpos (la radio debe quedar en el que sigue transmitiendo) |
| **Software de estación** | Los datos de la tabla de [7.4](#datos-a-completar-sobre-la-estación-existente) y permiso de uso |

### 11.3 A verificar antes de comprar

1. Precio y stock reales en Argentina: las capturas de este documento son de fechas mezcladas.
2. Tipo de conector (SMA vs. RP-SMA) de cada antena y del módulo, y adaptadores.
3. Que el módulo sea **RFM95W-915S2** y no el de 868 MHz ni uno basado en SX1278.
4. VSWR medido de las antenas compradas en 915–928 MHz.
5. Datos que no pude confirmar: nombre exacto del paso de canales del E32, manuales completos del E22 y del RYLR998, y si el modo LoRa del SX127x filtra direcciones por hardware (creo que no).

---

## 12. Conclusiones

El diseño del subsistema COMMS para el CanSat queda definido en sus puntos esenciales:

- **Enlace:** LoRa a 915–928 MHz con el módulo **RFM95W-915 (SX1276)**, configuración **BW 250 kHz, SF9, CR 4/5**. Cada paquete ocupa **164 ms** de cada segundo (**16,4 %**).
- **Trama:** binaria de **52 bytes** (Header 5, Payload 45, Trailer 2), con CRC-16, ID de destino y de origen, contador de 16 bits y confirmación de comandos por eco. A 1 Hz son **416 bit/s = 0,416 kb/s = 0,052 kB/s**, unas 12 veces menos que el cohete V2.
- **Margen de enlace:** **≈ 39 dB a 3 km** en el caso base y **≈ 24 dB** en el peor caso de orientación, contra los 10–15 dB buscados. Hay holgura suficiente para compensar lo que el modelo no incluye.
- **Antenas:** monopolo λ/4 de alambre de ≈ 80 mm dentro del contenedor (con una rubber duck como referencia) y omnidireccional de 2–5 dBi en mástil en la estación (con Yagi opcional, sujeta a G9).
- **Estación:** radio y puente USB alimentados por la notebook; software reutilizando la estación existente (FastAPI y web) con un núcleo propio, **condicionado a la consulta por G4**, con plan de respaldo en PySide6.
- **Lo que más puede salir mal** no es la potencia ni el alcance sino tres incógnitas: el **material del fuselaje**, la **interpretación de G4** y las **ambigüedades de X2 y X3**. Las tres están en la lista de consultas.

Todo lo que dependía de un supuesto quedó marcado **TBR** para resolverse antes de la CDR, que debe cerrar todos los TBD y TBR del PDR.
