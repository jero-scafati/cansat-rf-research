# Investigación SOTA — Puntos 4, 5 y 6
**CanSat Argentina Universitario 2026 · Subsistema de comunicaciones / estación terrena**

- **Punto 4:** Interfaz radio ⇄ notebook y energía de la estación
- **Punto 5:** Protocolo del enlace (paquetes, validación, direccionamiento, comandos)
- **Punto 6:** Software de la estación terrena

Fecha de relevamiento: 26/10/2026. Todo lo que dice "precio" es una foto del momento (ARS) y cambia seguido.

---

## 0. Cómo leer este documento

Cada afirmación importante lleva una etiqueta para que sepan cuánto pesa:

| Etiqueta | Significa |
|---|---|
| **[V]** | Verificado en una fuente que consulté en esta investigación (URL en la sección 9). |
| **[I]** | Inferencia o cálculo mío a partir de datos [V] o de los requisitos. Hay que validarlo. |
| **[C]** | Conocimiento general de ingeniería que **no** verifiqué en esta búsqueda. Confirmar antes de decidir con esto. |

Referencias a requisitos (G1…G15, X1…X6, F1…F5, E7, C5…) son las del Anexo A v2 (Apéndice A) y del archivo `investigacion-sota.md`.

**Alcance y dependencias.** El punto 1 (la radio en sí) lo investiga otra persona, pero los puntos 4, 5 y 6 dependen de qué radio se elija. Por eso en la sección 6 hay una matriz "si la radio es X, entonces interfaz / protocolo / software son Y". Donde necesité datos de radios (consumo, tamaño máximo de paquete, direccionamiento), los tomé de datasheets o páginas de fabricante y los marco.

**Importante sobre G4.** "Cada equipo deberá desarrollar su propia estación terrena". Esto condiciona el punto 6 más de lo que parece: usar un dashboard genérico ya hecho puede no cumplir el espíritu del requisito (ver 5.3).

---

## 1. Resumen ejecutivo (lo que yo haría)

1. **Interfaz (punto 4):** que la radio de la estación llegue a la notebook como **puerto serie virtual por USB** (USB-UART con CP2102N, o USB nativo CDC si usan una placa con microcontrolador). Es lo más simple, tiene drivers en Windows/Linux/macOS y funciona con `pyserial`. **Ojo con la energía:** una radio de 1 W puede pedir cerca de 1 A en transmisión, más que lo que un puerto USB 2.0 entrega por norma (500 mA). Soluciones: bajar la potencia en la estación (solo manda comandos esporádicos), alimentar por un hub USB con alimentación propia o power bank, o elegir una radio de menor consumo. En recepción el consumo es de decenas de mA, así que las 2 h de G9 no son un problema de energía sino de **corriente pico**.
2. **Protocolo (punto 5):** paquete **binario de tamaño fijo (~50 bytes)** con ID de equipo, tipo, contador de transmitidos (F1), tiempo de misión, estado y **CRC-16**; la estación decodifica y muestra en texto (G5). A 1 Hz entra holgado en LoRa con SF7–SF9 a 125 kHz (~100–330 ms en el aire) **[I]**. Los comandos van con opcode + argumentos + CRC, y la **confirmación viaja en la telemetría** (campo "último comando recibido / resultado"), sin ACK de radio, para no gastar aire ni complicar el half-duplex.
3. **Software (punto 6):** **Python + PySide6 + pyqtgraph + pyserial**. Es la opción que mejor equilibra: una sola ventana sin pestañas (G13), control total de fuentes y trazos (G12), gráficos en tiempo real livianos, CSV a dos destinos (G2), comandos desde la misma pantalla (G15), licencia limpia (PySide6 es LGPL) y es código propio (G4). Construyan **un simulador de telemetría desde el día 1**.
4. **Lo que más puede salir mal:** (a) corriente pico del radio vs. puerto USB; (b) tamaño de paquete vs. límites del módulo (ej. 58 bytes en la familia E-byte UART, 32 bytes en nRF24); (c) SF alto que no entra en 1 s; (d) pantalla a 14 pt con 6+ gráficos en una notebook chica; (e) licencia/G4 si se usa software ajeno.

---

## 2. Trazabilidad: qué requisito toca cada punto

| Punto | Requisitos que aplican |
|---|---|
| **4. Interfaz y energía** | G9 (notebook ≥ 2 h, radio, antena), G10 (portátil, **sin CA**), G15 (enviar telecomandos), G1/F3 (comandos antes del despegue), X1 (bandas), X2 (sin broadcast) |
| **5. Protocolo** | F1 (contador persistente), F2/G3 (tiempo de misión, 1 s), F3 (hora ±1 s UTC), F4 (comandos de info), F5 (estados persisten), G14 (recibidos con éxito ≠ transmitidos), G8 (recibidos y perdidos), X3 (comienzo de misión), X4/C5 (1 Hz), X5 (contenido), X6 (batería y paneles por telecomando), G7 (V e I en tiempo real), X2 (no broadcast) |
| **6. Software** | G2 (CSV en notebook y disco de respaldo), G4 (propia), G5 (texto en tiempo real), G6 (unidades SI en pantalla), G7 (gráficos), G8 (indicadores), G9 (autonomía), G12 (≥14 pt, negrita, texto oscuro sobre fondo claro), G13 (todo simultáneo, sin pestañas), G14, G15, y la prueba de estación en la FRR |

---

## 3. PUNTO 4 — Interfaz radio ⇄ notebook y energía de la estación

### 3.1 Qué hay que resolver

1. Cómo llegan los bytes de la radio a la notebook y cómo salen los comandos.
2. Quién alimenta la radio de la estación (sin CA en el sitio, G10) y con qué corriente.
3. Que todo siga funcionando ≥ 2 h (G9) con la pantalla visible bajo sol.
4. Que no se pierdan datos ni se cuelgue el puerto durante el vuelo.

### 3.2 Arquitecturas de interfaz (comparativa)

| Opción | Cómo se conecta | Ejemplos | Qué entrega a la notebook | Cómo se mandan comandos | Quién alimenta | Pros | Contras / riesgos |
|---|---|---|---|---|---|---|---|
| **A. Radio con UART nativo + adaptador USB-UART** | Módulo UART (3,3 V TTL) → chip USB-UART → USB | Familia E-byte E32 (LoRa 915 MHz), RFD900x (SiK), XBee-PRO 900HP | Flujo de bytes por puerto serie virtual (COMx / `/dev/ttyUSBx`) | Se escriben bytes al mismo puerto (transparente) o con modo de comandos/API del módulo | USB del adaptador (5 V) o fuente externa | La más simple; casi no hay firmware en la estación; librería `pyserial` | El módulo UART **no preserva límites de paquete** (hay que enmarcar); límites propios (58 B por paquete en E-byte); corriente de TX puede superar el puerto USB |
| **B. Módulo SPI + microcontrolador puente con USB** | Radio SPI (RFM95/SX1276, SX1262, SX1280) → MCU (ESP32, RP2040, STM32) → USB (CDC o USB-UART) | RFM95W + ESP32/Arduino; Heltec LoRa32 V3; Adafruit Feather RFM95 | Lo que defina su firmware (p. ej. paquetes ya validados, con RSSI/SNR) | Comando por el mismo puerto; el firmware lo arma y lo transmite | Puerto USB de la notebook (la placa regula a 3,3 V) | Control total del enlace, RSSI/SNR, direccionamiento propio, reutilizan el firmware del CanSat | Hay que escribir y depurar el firmware puente; más piezas que fallan |
| **C. Placa integrada MCU + radio ya con USB** | Todo en una placa (USB directo) | Heltec LoRa32 V3, LilyGo T-Beam (ESP32 + LoRa + GPS) | Igual que B | Igual que B | USB (o batería 18650 en T-Beam) | Barata, hay stock en Argentina, ya trae USB-UART | Antena y potencia limitadas (hasta ~100 mW en estos módulos); calidad de USB-UART (CP2104/CH9102) varía por lote |
| **D. 2,4 GHz con ESP32 (ESP-NOW) como puente** | ESP32 → USB | Cualquier ESP32 DevKit | Paquetes ≤ 250 B por unicast con MAC | Se envían por `esp_now_send` | USB | Direccionamiento por MAC (unicast) nativo, ACK de capa MAC, barato y local | Banda 2,4 GHz (hay que evitar canal 14); alcance a validar en el punto 1; interferencia de Wi-Fi del campo |
| **E. nRF24L01+ (2,4 GHz) + MCU** | SPI → MCU → USB | Módulos nRF24L01+ | Payload de 1–32 bytes | Igual que B | USB | Muy barato, ACK y direccionamiento por hardware (3–5 bytes) | **Payload ≤ 32 B**: el paquete de ~50 B no entra sin recortar o fragmentar; canales por encima de 2483,5 MHz quedan fuera de X1 |
| **F. Enlace inalámbrico a la notebook (BT/Wi-Fi)** | Radio → ESP32 → Wi-Fi/BLE → notebook | — | — | — | Batería propia | Sin cable | Descartada [I]: suma latencia, otro enlace que puede fallar y más consumo sin ganar nada |

**Recomendación preliminar [I]:** A o C según la radio que salga del punto 1. B si necesitan RSSI/SNR y direccionamiento por software.

### 3.3 Chips USB-UART (si van por la opción A o usan una placa con USB-UART)

| Chip | Drivers | Velocidad máx. | Notas | Riesgo | Precio AR (módulo suelto, ML, ARS) |
|---|---|---|---|---|---|
| **CP2102N / CP2104 (Silicon Labs)** | Linux: incluido en el kernel. Windows: lo baja Windows Update. macOS: driver de Silabs | 3 Mbps (CP2102N), 2 Mbps (CP2104) | Mejor soporte de drivers que el CH340; el CP2102N es el sucesor del CP2104 **[V]** | Bajo | ~$2.000–$7.000 (módulos CP2102); con cable ~$10.600–$12.500; marca HiLetgo ~$60.600 |
| **FT232RL (FTDI)** | Muy buen soporte en Windows/macOS/Linux; el original es "el estándar de oro" de confiabilidad **[V]** | 3 Mbps | Hay muchísimas copias truchas en el mercado **[C]** | **Medio** (falsificaciones) | ~$4.300–$13.000 (módulos "FT232RL" de ML; probablemente clones, sin garantía de chip original) **[I]** |
| **CH340 / CH341 (WCH)** | Windows 10/11 trae driver; Linux módulo `ch341`; macOS requiere instalar el driver de WCH **[V]** | ~2 Mbps | Muy barato; algunos reportan problemas ocasionales de estabilidad **[V]** | Medio (macOS) | ~$1.200–$5.800 |
| **CH9102** | Similar a CH340 (driver WCH) | — | Viene en varias placas ESP32 recientes **[V]** | Medio | (incluido en placas) |
| **PL2303 (Prolific)** | Los drivers modernos de Windows **rechazan los chips clonados** (error "Code 10") **[V]** | — | Evitar si no se puede garantizar chip original | **Alto** | ~$2.500 |

**Decisión sugerida [I]:** CP2102N (o CP2104) de un vendedor con buena reputación (ej. los de ~50 calificaciones ≥ 4,6 en ML). Comprar **dos** (uno de repuesto) y probar ambos con la notebook de la competencia **antes** de la FRR. Si la placa que usan trae su propio USB-UART, igual lleven un adaptador de reserva.

### 3.4 Energía: presupuesto de corriente y de energía

**Qué entrega un puerto USB por norma [V]:**

| Tipo de puerto | Corriente a 5 V |
|---|---|
| USB 2.0 estándar | 500 mA (100 mA antes de configurar el dispositivo) |
| USB 3.x estándar | 900 mA |
| Puerto de carga (BC 1.2, CDP/DCP) | hasta 1,5 A (solo si la notebook lo implementa) |
| USB-C | 1,5 A o 3 A si el puerto lo anuncia; con PD, más |

Un dispositivo certificado no debe pedir más de lo que declaró al enumerarse **[V]**. En la práctica muchas notebooks cortan el puerto por sobrecorriente **[V: caso en foro de Arch Linux; C en general]**.

**Consumo de las radios candidatas (de datasheets y fichas de venta):**

| Radio / módulo | Tensión de alimentación | Consumo en TX | Consumo en RX | Estado |
|---|---|---|---|---|
| RFD900x (SiK, 1 W) | 5 V nominal (3,5–5,5 V) | ~0,8 A pico, hasta ~1 A a +30 dBm | ~60 mA típico | [V] |
| XBee-PRO 900HP | 2,1–3,6 V | 215 mA | 29 mA | [V] (datasheet de 2012; revisar la versión vigente) |
| E-byte E32-915T20D (100 mW) | 3,3–5 V | ~120 mA | ~14 mA | [V] |
| E-byte E32 clase 1 W (433 MHz como referencia) | 3,3–5,5 V | ~550 mA instantáneo (la variante 33 dBm llega a 1,2 A) | ~15 mA | [V] para 433 MHz; **verificar la ficha de la variante 915 MHz** |
| RFM95W (SX1276, SPI) | 3,3 V (1,8–3,7 V) | 120 mA a +20 dBm; 90 mA a +17 dBm; 29 mA a +13 dBm | 10–14 mA | [V] |
| ESP32 / MCU puente | 5 V (USB) → 3,3 V | sin dato verificado | sin dato verificado | **Medir** con un tester USB |
| nRF24L01+ | 1,9–3,6 V | sin dato verificado en esta búsqueda | — | [C] decenas de mA, verificar datasheet |

**Energía total en 2 h (cotas de peor caso) [I]:**

| Escenario | Potencia | Energía en 2 h |
|---|---|---|
| XBee-PRO 900HP recibiendo | ~0,10 W | ~0,19 Wh |
| E32 100 mW recibiendo | ~0,08 W | ~0,15 Wh |
| RFD900x recibiendo | ~0,30 W | ~0,6 Wh |
| E32 1 W transmitiendo **continuamente** (cota absurda) | ~2,75 W | ~5,5 Wh |
| RFD900x transmitiendo **continuamente** a 1 W (cota absurda) | ~5 W | ~10 Wh |

**Lectura [I]:** la estación pasa casi todo el tiempo en recepción y solo transmite comandos esporádicos (calibración, hora, pedidos de estado). La energía del radio es despreciable frente a una batería de notebook de 40–60 Wh **[C]**. Lo que consume la batería es la **pantalla a máximo brillo** (G12 exige verla con sol intenso), el procesador y el disco de respaldo. **Midan la autonomía real con brillo máximo y el simulador corriendo 2 h**, no con la radio en reposo.

**El problema real es el pico de corriente.** Una radio de 1 W puede superar los 500 mA de un USB 2.0 en el instante de transmitir. Opciones:

1. **Bajar la potencia de TX de la estación** (los comandos viajan pocos bytes y poco seguido; la estación está cerca o en línea de vista). Es lo más barato y lo más usual **[I]**.
2. **Hub USB con alimentación propia** o **cable en Y / power bank** que provea los 5 V y use la notebook solo para datos. Es "otra fuente propia", compatible con lo que dice la investigación del punto 4, pero **confirmen con CONAE** que G9 no exige que la radio salga de la batería de la notebook.
3. **Condensador de reserva** cerca del PA del módulo (cientos de µF) para absorber el pico sin tirar la tensión del bus **[C]**.
4. Elegir radios de ≤ 100 mW en la estación si el alcance lo permite.

**Buenas prácticas del lado de la notebook [C]:**
- Cable USB corto y de buena calidad (la caída de tensión en un cable largo o fino baja los 5 V y reinicia el módulo).
- **Deshabilitar la suspensión selectiva de USB** y el ahorro de energía del puerto en Windows para que no corte el puerto durante el vuelo.
- Fijar el nombre del puerto (el número COM cambia al cambiar de puerto físico): que el software **busque el dispositivo por VID/PID o número de serie**, no por "COM3".
- **Interferencia de USB 3.0 en 2,4 GHz:** los puertos y cables USB 3.x emiten ruido que degrada receptores en 2,4 GHz. Si usan esa banda, conecten el receptor a un puerto USB 2.0 o con un alargue lejos del puerto. [C: problema muy conocido, verificar con mediciones en su notebook]
- Sincronizar el reloj de la notebook **antes de salir al campo** (el comando de hora F3 sale de acá y se pide ±1 s respecto de UTC; en el campo no habrá internet).
- Si el "disco de respaldo" (G2) es un disco rígido USB mecánico, consume corriente del mismo bus. Prefieran un **pendrive o SSD** [I].

### 3.5 Formato de datos y comandos por la interfaz

| Interfaz / radio | Formato que entrega a la notebook | Cómo se mandan comandos | Notas para el software |
|---|---|---|---|
| **UART transparente (E32 en modo normal, RFD900x/SiK)** | Flujo de bytes sin límites de paquete | Se escriben bytes; la estación los envía tal cual | Hay que enmarcar (delimitador, COBS) y validar con CRC en la aplicación |
| **E32 en modo "fixed"** | Flujo de bytes | Los 3 primeros bytes del envío son dirección alta, dirección baja y canal **[V]** | Ojo: dirección 0xFFFF equivale a broadcast (evitar por X2) |
| **XBee en modo API** | Tramas con delimitador 0x7E, longitud, tipo, datos y checksum (0xFF − suma) **[V]** | Trama "transmit request" con dirección de 64 bits | La librería parsea tramas; hay modo punto a punto sin ACK y con ACK |
| **Firmware puente propio (opción B/C)** | Lo definen ustedes: por ejemplo trama COBS + metadatos (RSSI, SNR) | Por el mismo puerto, con un opcode propio | Permite mostrar RSSI/SNR en pantalla (útil para la FRR) |
| **ESP-NOW (puente ESP32)** | Callbacks con MAC de origen + payload ≤ 250 B **[V]** | `esp_now_send(MAC, datos)` | El ESP32 puede empaquetar a USB con COBS |

### 3.6 Dónde conseguir lo del punto 4 en Argentina

Foto de precios del 26/10/2026 (ARS, IVA incluido salvo aclaración). **Verifiquen stock y precio antes de comprar.**

| Ítem | Dónde | Precio relevado | Notas |
|---|---|---|---|
| Módulos CP2102 USB-TTL | Mercado Libre AR (varios vendedores) | $2.079–$6.878; con cable ~$10.600–$12.500 | Hay vendedores con 50+ calificaciones |
| Módulos "FT232RL" | Mercado Libre AR | $4.337–$13.154 | Probablemente clones; no confiar sin probar |
| CH340 USB-TTL | Mercado Libre AR / Unoelectro | $1.156 (Unoelectro) – ~$5.800 | El más barato; más trabajo de driver en macOS |
| PL2303 | Mercado Libre AR | ~$2.499 | Evitar (clones rechazados por drivers modernos) |
| Heltec LoRa 32 V3.1 (868/915 MHz) | Unoelectro (AR) | $63.224 | La ficha muestra "sin stock" y a la vez unidades; **confirmar** |
| LilyGo T-Beam v1.2 (915 MHz, GPS) | Mercado Libre AR | ~$83.439 | Con soporte 18650; envío local según vendedor |
| RFM95W (SX1276) 915 MHz, pack x2 | Mercado Libre AR | ~$39.990 | Solo el módulo SPI, necesita MCU |
| LilyGo T-Echo (SX1262) 915 MHz | Mercado Libre AR (compra internacional desde EE. UU.) | ~$290.679 | Caro y por courier |
| RFD900x-US (referencia de precio) | irlock.com (EE. UU.) | USD 199 | **No encontré oferta local verificada**; importación por courier |
| XBee-PRO 900HP, E32-915T30D | — | — | **No encontré oferta local verificada** en esta búsqueda; asumir importación |
| Power banks USB-C PD 10.000–20.000 mAh | Mercado Libre AR y retail | — | Hay mucha oferta; **no relevé precios argentinos confiables** en esta búsqueda |

**Plazo y aduana [I]:** todo lo que no esté en stock local implica courier (semanas) y posible retención aduanera. Si deciden importar una radio, háganlo con colchón de al menos 4–6 semanas y encarguen una unidad de repuesto.

**Costo C9 [I]:** el costo de la estación terrena queda afuera de C9 ("herramientas de análisis y soporte en tierra"). El gasto de estos adaptadores y de los power banks no suma al tope de $1.000.000 del CanSat.

### 3.7 Pruebas y checklist del punto 4

- [ ] Medir con un tester USB la corriente del radio de la estación en RX y en TX a la potencia elegida.
- [ ] Probar con la notebook **real** de la competencia en sus puertos USB 2.0, 3.x y USB-C; verificar que ningún puerto corte.
- [ ] Desenchufar y volver a enchufar el cable en caliente: ¿el software se reconecta solo?
- [ ] Autonomía: 2 h con brillo máximo, software corriendo con el simulador, disco de respaldo conectado.
- [ ] Probar con el adaptador de repuesto.
- [ ] Confirmar con CONAE que alimentar la radio desde un hub/power bank propio no viola G9 (ver sección 8).

---

## 4. PUNTO 5 — Protocolo del enlace

### 4.1 Qué hay que resolver

El protocolo tiene tres capas conceptuales. Conviene separarlas:

1. **Capa de radio (PHY/MAC):** la resuelve el módulo (LoRa, FSK, GFSK, Wi-Fi…). Algunos ya traen CRC, direccionamiento y ACK.
2. **Capa de enlace propia:** cómo se enmarca y valida un paquete en el aire y en el cable UART/USB.
3. **Capa de aplicación:** qué campos lleva la telemetría y cómo son los comandos.

### 4.2 Formatos de paquete (comparativa)

Tomo como ejemplo la telemetría mínima de la acción 1A (X5, G3, F1, G7, G8). Un paquete binario compacto tiene **49 bytes** (cálculo en 4.3).

| Opción | Overhead aprox. | Tamaño del ejemplo | Validación de errores | Identificación de destinatario | Contador / pérdidas | Depuración | Esfuerzo | Riesgos |
|---|---|---|---|---|---|---|---|---|
| **A. ASCII tipo CSV con checksum (estilo NMEA)** | Separadores y texto; checksum XOR de 1–2 caracteres | **~121 B** (ejemplo medido) | Checksum XOR/suma (débil) o CRC en hexa | Campo de texto "ID equipo" | Campo de texto | Excelente: se lee con cualquier terminal | Bajo | Paquete grande (más aire, más probabilidad de error); parseo lento; en E32 se parte en 3 paquetes de ≤58 B |
| **B. Binario de tamaño fijo + CRC-16** | 2–3 B de cabecera + 2 B de CRC | **49 B** | CRC-16 (ver 4.5) | Byte de ID de equipo + dirección de capa de radio | Contador de 16 bits | Medio: hace falta decodificador y modo "volcado hexa" | Medio | Cambiar un campo exige actualizar ambos lados (usen un campo de versión) |
| **B+. Binario fijo + CRC-16 + COBS** (para enlaces UART transparentes) | + 1 B de COBS + 1 B delimitador 0x00 | ~51 B | CRC-16 | Igual que B | Igual que B | Medio | Medio | COBS suma como máximo 1 byte cada 254 más el delimitador **[V]**; resincroniza solo en el próximo 0x00 |
| **C. MAVLink 2 (dialecto propio)** | Cabecera 10 B + CRC 2 B (+13 B si se firma) **[V]** | ~55–60 B | CRC-16/MCRF4XX con CRC_EXTRA que valida la definición **[V]** | `sysid`/`compid` y `target_system` para comandos **[V]** | Número de secuencia de 1 byte **[V]** | Muy bueno (herramientas existentes) | Medio-alto: definir mensajes en XML y generar código | Por defecto es "flujo multicast" sin destino **[V]**; el contador de 8 bits da vueltas rápido; arrastra mucha infraestructura que no necesitan |
| **D. CCSDS Space Packet** | Cabecera primaria de 6 B [C] | ~55 B | CRC opcional [C] | APID [C] | Sequence count de 14 bits [C] | Medio | Alto | Pensado para misiones espaciales; sobrediseño para este caso; **no lo verifiqué** en esta búsqueda |
| **E. Datagrama de RadioHead (RH_RF95) + payload binario** | 4 B de cabecera (TO, FROM, ID, FLAGS) **[V]** | 53 B | CRC de 16 bits del propio radio LoRa + el propio de ustedes si lo agregan **[V]** | Campos TO/FROM de 1 byte **[V]** | ID de 1 byte (por mensaje) **[V]** | Bueno | Bajo si usan SX127x + Arduino | El ID de 8 bits no alcanza como contador F1; la dirección 255 es broadcast y no se confirma **[V]** |
| **F. API del módulo (XBee API / E32 fixed)** | Cabecera del módulo (6–14 B en XBee) **[V]** | ~60 B | Checksum de trama (XBee) + CRC propio | Dirección de 64 bits (XBee), 2 B + canal (E32) **[V]** | Propio | Medio | Bajo (depende de la radio) | Queda acoplado a esa familia de radios |

**Recomendación preliminar [I]:** **B (o B+ si el enlace es UART transparente)**, con ID de equipo, versión, tipo, contador de 16 bits, tiempo de misión, estado, campos X5/G7 y CRC-16. Si tienen LoRa SPI con RadioHead, la opción E es válida usando los 4 bytes de cabecera para TO/FROM/tipo/flags y poniendo **dentro del payload** su propio contador F1 de 16 bits.

### 4.3 Tamaño del paquete (propuesta de layout)

Layout binario little-endian. Las resoluciones son **propuestas**: las definen con Aviónica según los sensores reales.

| # | Campo | Tipo | Bytes | Resolución / unidad sugerida | Requisito |
|---|---|---|---|---|---|
| 1 | Destino | uint8 | 1 | ID de la estación | X2 |
| 2 | Origen (ID equipo) | uint8 | 1 | ID del CanSat | X2 |
| 3 | Tipo y versión | uint8 | 1 | TLM, CMD, RSP, EVT | — |
| 4 | Contador de paquetes transmitidos | uint16 | 2 | Persiste ante reinicios | F1 |
| 5 | Tiempo de misión | uint32 | 4 | s (resolución 1 s) | G3, F2 |
| 6 | Altitud (barométrica) | int16 | 2 | dm | X5, SN1 |
| 7 | Presión | uint32 | 4 | Pa | X5 |
| 8 | Temperatura | int16 | 2 | 0,01 °C | X5, SN2 |
| 9 | Aceleración X, Y, Z | 3 × int16 | 6 | mg o 0,01 m/s² | X5, SN5 |
| 10 | Tasa de rotación X, Y, Z | 3 × int16 | 6 | 0,01 °/s (se muestra en rad/s o con la conversión) | X5, SN5 |
| 11 | GPS latitud | int32 | 4 | 1e-7° | X5, SN4 |
| 12 | GPS longitud | int32 | 4 | 1e-7° | X5 |
| 13 | GPS altitud | int32 | 4 | cm | X5 |
| 14 | Satélites rastreados | uint8 | 1 | — | X5 |
| 15 | Tensión de batería | uint16 | 2 | mV | G7, SN3 |
| 16 | Corriente de batería | int16 | 2 | mA | G7, SN8 |
| 17 | Estado del software de vuelo / flags | uint8 | 1 | STAND-BY, ASCENSO, MISIÓN…; un bit de evento "inicio de misión" | G8, X3, F5 |
| 18 | CRC-16 | uint16 | 2 | Sobre campos 1–17 | G14 |
| | **Total** | | **49** | | |

Campo opcional (+2 B → **51 B**): `ultimo_cmd_id` y `resultado` (para confirmar comandos por la telemetría, ver 4.7).

**Referencia en ASCII:** una fila equivalente en texto plano con 18 campos mide ~121 bytes **[I: ejemplo armado y contado]**.

### 4.4 Tiempo en el aire y presupuesto de 1 s (X4/C5)

#### LoRa (SX1276/SX1262)

Calculé con la fórmula de tiempo en el aire del datasheet SX127x (cabecera explícita, CRC de payload activo, CR 4/5, preámbulo de 8 símbolos, optimización de baja tasa automática cuando el símbolo dura más de 16 ms). **[I: cálculo mío, validen con un calculador oficial o con mediciones]**

| SF | 49 B · BW 125 kHz | 49 B · BW 250 kHz | 49 B · BW 500 kHz | 53 B (49+4 de RadioHead) · BW 125 kHz | 121 B (ASCII) · BW 125 kHz |
|---|---|---|---|---|---|
| 7 | 98 ms | 49 ms | 24 ms | 103 ms | 205 ms |
| 8 | 175 ms | 87 ms | 44 ms | 185 ms | 359 ms |
| 9 | 329 ms | 164 ms | 82 ms | 329 ms | 656 ms |
| 10 | 576 ms | 288 ms | 144 ms | 616 ms | 1.190 ms |
| 11 | **1.233 ms** | 535 ms | 267 ms | 1.315 ms | 2.544 ms |
| 12 | **2.302 ms** | 1.151 ms | 535 ms | 2.466 ms | 4.760 ms |

Conclusiones **[I]**:
- Con **BW 125 kHz, SF11 y SF12 no entran en 1 s** (ni con 49 bytes). SF10 apenas entra y deja menos de 0,4 s para escuchar comandos. Para 1 Hz con margen conviene **SF7–SF9 a 125 kHz** o subir el ancho de banda (250/500 kHz) si necesitan SF más alto.
- Dos de los cuatro presets de RadioHead (`Bw31_25Cr48Sf512` y `Bw125Cr48Sf4096`) rinden del orden de 2–3 s por paquete de 49 B y **no sirven para 1 Hz** **[I]**. Los otros dos son `Bw125Cr45Sf128` (SF7, por defecto) y `Bw500Cr45Sf128` **[V]**.
- El paquete en ASCII de ~121 B pone SF10 fuera de 1 s y SF9 a dos tercios del segundo. **Binario gana.**

#### UART entre radio y notebook

A 9.600 baud (8N1) el paquete de 49 B tarda ~51 ms y el de 121 B ~126 ms; a 57.600 baud, ~8,5 ms y ~21 ms **[I]**. Usen **57.600 o 115.200 baud** salvo que el módulo lo limite.

#### Módulos UART de E-byte (E32)

Tienen búfer de 512 B y **parten automáticamente en paquetes de 58 bytes** **[V]**. Tasas de aire de 0,3 a 19,2 kbps según el modelo **[V]**:

| Tasa de aire | 49 B | 58 B | 121 B (2 paquetes) |
|---|---|---|---|
| 2,4 kbps | ~163 ms | ~193 ms | ~403 ms |
| 4,8 kbps | ~82 ms | ~97 ms | ~202 ms |
| 9,6 kbps | ~41 ms | ~48 ms | ~101 ms |

*(Cálculo simple bytes×8/tasa, sin preámbulo ni overhead propio del módulo [I].)*

#### Límites de tamaño por tecnología

| Tecnología | Límite de payload | Estado |
|---|---|---|
| LoRa con RadioHead (RH_RF95) | 251 B de datos (255 menos 4 de cabecera) | [V] |
| ESP-NOW v1.0 | 250 B (v2.0 hasta 1.470 B) | [V] |
| E-byte UART | 58 B por paquete (parte automático) | [V] |
| nRF24L01+ | 32 B | [V] |
| XBee-PRO 900HP | Depende de la configuración | Verificar en el manual |

**Conclusión [I]:** el paquete binario de 49–51 B entra en todas las tecnologías **salvo nRF24** (32 B). En E32 queda debajo del corte de 58 B.

### 4.5 Validación de errores

| Método | Qué detecta | Overhead | Comentario |
|---|---|---|---|
| Checksum XOR o suma de 8 bits | Errores simples; falla con muchos patrones | 1 B | Insuficiente para un enlace con ruido |
| CRC-8 | Ráfagas cortas | 1 B | Aceptable en paquetes de ≤ 15 B [C] |
| **CRC-16 (CCITT / MCRF4XX)** | Todo error de un bit, dobles, ráfagas ≤ 16 bits; la probabilidad de no detectar un error aleatorio es ~1/65.536 [C] | **2 B** | **Recomendado.** MAVLink usa CRC-16/MCRF4XX **[V]** |
| CRC-32 | Más robusto | 4 B | Sobra para ~50 B a 1 Hz [C] |
| CRC del propio radio LoRa (payload) | El chip descarta paquetes con CRC malo y RadioHead cuenta los malos **[V]** | 0 B (lo pone el radio) | **No reemplaza** el CRC de aplicación, porque el tramo UART/USB hacia la notebook y el firmware puente también pueden fallar [I] |

**Regla de oro [I]:** un paquete cuenta como "recibido con éxito" (G14) **solo si pasa el CRC de la aplicación en la estación**, y recién entonces incrementa `rx_ok`.

### 4.6 Identificación del destinatario y X2 (sin broadcast)

"X2: las radios no deben utilizar el modo de difusión (broadcast)" no define el término. Mi lectura **[I]** es "cada paquete va dirigido a un destinatario concreto".

| Tecnología | Mecanismo de direccionamiento | Ojo con el broadcast |
|---|---|---|
| RadioHead | Campos TO y FROM de 1 byte | La dirección **255 es broadcast** y no se confirma **[V]**; no la usen |
| E-byte E32 | Modo "fixed": dirección alta + dirección baja + canal en los 3 primeros bytes **[V]** | La dirección 0xFFFF o el modo transparente reparten a todos los que estén en el mismo canal **[V]** |
| XBee 900HP | Dirección de 64 bits del destino **[V]** | Existe dirección de broadcast; no usarla **[V]** |
| ESP-NOW | Unicast por MAC de 6 bytes (o broadcast si se pide) **[V]** | No añadir la MAC de broadcast a la lista de pares **[I]** |
| nRF24L01+ | Direcciones de 3–5 bytes por "pipe" **[V]** | Es unicast por dirección del receptor |
| RFD900x (SiK) | Punto a punto o multipunto con ID de red **[V]** | Usar punto a punto |

**Diseño que cubre cualquier interpretación [I]:**
1. Dirección propia (unicast) a nivel de radio donde exista.
2. **ID de equipo y de estación dentro del paquete**, y filtro en el receptor (descartar lo que no es para mí).
3. **Frecuencia/canal asignado** por CONAE (la FRR lo verifica).
4. Documentar todo en el PDR como supuesto TBR.

Aun con unicast, la señal la reciben físicamente otros equipos: el direccionamiento no impide la interferencia, la evita el plan de canales.

### 4.7 Telecomandos

#### Estructura propuesta de trama de comando

`[destino][origen][tipo=CMD][seq_cmd][opcode][longitud][argumentos...][CRC-16]` — y la respuesta es otro tipo (`RSP`) o un campo en la telemetría.

#### Tabla de comandos (a partir del roadmap)

| Opcode | Nombre | Argumentos | Sentido | Cuándo | Requisito |
|---|---|---|---|---|---|
| 0x01 | `CAL_ALT_ZERO` | — | Estación → CanSat | En plataforma, antes del despegue | G1, F5 |
| 0x02 | `SET_TIME` | UTC en segundos (uint32) + milisegundos (uint16) | Estación → CanSat | Antes del lanzamiento | F3, F2 |
| 0x03 | `REQ_BATT_SOLAR` | — | Estación → CanSat; responde por telemetría | Sin definir | X6 |
| 0x04+ | `REQ_INFO_x` | Según el comando | Estación → CanSat | Sin definir; se documenta en el manual de la misión | F4 |
| — | Evento "inicio de misión" | — | CanSat → Estación, automático | Al separarse del cohete | X3 |

*Los opcodes son propuestos [I].*

#### Confirmación de comandos (ACK): tres alternativas

| Alternativa | Cómo funciona | Pros | Contras |
|---|---|---|---|
| **1. Sin ACK** | Se envía y se espera lo mejor | Cero costo | Si se pierde la calibración (G1) no se enteran |
| **2. ACK de capa de radio** (RadioHead `RHReliableDatagram`) | El receptor responde con un paquete con bit ACK y mismo ID; reintenta 3 veces con timeout aleatorio de 200–400 ms por defecto **[V]** | Ya implementado | Usa aire extra y exige que el CanSat esté en RX en el momento justo; con half-duplex complica los 1 Hz |
| **3. Confirmación en la telemetría (recomendada) [I]** | La telemetría lleva `ultimo_cmd_id` y `resultado`; la estación reintenta hasta verlo reflejado | Aprovecha el paquete que ya sale cada segundo; no suma TX | Solo se confirma cuando llega el siguiente paquete (~1 s de latencia) |

**Regla de ventana de escucha [I]:** como el CanSat transmite a 1 Hz, la estación envía el comando **inmediatamente después de recibir un paquete de telemetría**. Así sabe que el CanSat acaba de transmitir y está por escuchar. Solo hace falta una ventana de RX corta del CanSat después de cada TX (ver tiempos de 4.4).

**Comando de hora (F3):** F3 pide ±1 s respecto de UTC. La latencia de aire (~0,1–0,3 s) y de UART (decenas de ms) está dentro del margen. Aun así conviene que el comando lleve el tiempo con el que **la estación predice** que llegará (hora actual + latencia estimada) **[I]**. El reloj de la notebook debe estar sincronizado (NTP) antes de salir.

### 4.8 Contadores, pérdidas y estados (F1, F5, G8, G14, X3)

**Contador de paquetes transmitidos (F1):** persistente ante reinicios. Cuidado con la memoria: escribir en EEPROM/flash a 1 Hz la desgasta en horas **[C]**; usen FRAM o un esquema de escritura espaciada y asuman que tras un reinicio el contador salta algunas unidades **[I]**.

**Cálculo en la estación [I]:**
```
rx_ok       = paquetes con CRC válido recibidos
primero     = contador del primer paquete válido
ultimo      = contador del último paquete válido
esperados   = ultimo - primero + 1      (considerar el desborde de uint16)
perdidos    = esperados - rx_ok
```
Si el CanSat se reinicia, el contador sigue (F1), así que no hay saltos hacia atrás; un salto grande indica reinicio o ráfaga larga perdida.

**Inicio de misión (X3) [I]:** agreguen un bit de evento "inicio de misión" y **repítanlo durante los primeros N paquetes** del estado MISIÓN. Si se pierde el primero, la estación igual lo ve.

**Estado persistente (F5):** el campo de estado en el paquete sirve para que la estación vea si el CanSat se reinició.

### 4.9 Datos en el aire vs. en pantalla (G5)

G5 pide mostrar en texto la telemetría. No obliga a que sea texto en el aire **[I]**. Estrategia: **binario en el aire, texto en pantalla y en los CSV**, y un registro adicional "crudo" en hexa para depurar.

### 4.10 Recomendación del punto 5

- Paquete binario fijo de ~50 B con CRC-16, contadores y estado; COBS si el enlace es UART transparente.
- Direccionamiento unicast + IDs en el paquete + canal asignado.
- Comandos con opcode + args + CRC, confirmados por eco en la telemetría.
- Documentar el protocolo (tabla de campos y de comandos) en el manual de la misión (F4) y en el PDR.

### 4.11 Pruebas del punto 5

- [ ] Inyectar errores de bits y paquetes perdidos en un simulador: verificar `rx_ok`, `perdidos` y que no cuente paquetes corruptos.
- [ ] Reiniciar el CanSat en medio de la prueba: el contador debe continuar (F1) y el estado sobrevivir (F5).
- [ ] Medir el tiempo en el aire real con el SF elegido y confirmar que cabe en 1 s con la ventana de escucha.
- [ ] Enviar `CAL_ALT_ZERO` y `SET_TIME` con la radio a distancia y comprobar la confirmación por telemetría.
- [ ] Verificar que la estación ignora paquetes de otro equipo (ID distinto).

---

## 5. PUNTO 6 — Software de la estación terrena

### 5.1 Qué hay que cumplir

| Requisito | Qué implica en el software |
|---|---|
| G4 | Software propio del equipo |
| G5 | Telemetría en texto en tiempo real, ascenso y descenso |
| G6 | Unidades SI indicadas en pantalla |
| G7 | Gráficos en tiempo real de altitud, tensión, corriente, aceleración y tasas de rotación |
| G8 | Mostrar tiempo de misión, temperatura, posición GPS, paquetes recibidos, perdidos y estado del software de vuelo |
| G12 | Fuentes ≥ 14 pt, trazos y ejes en negrita, texto oscuro sobre fondo claro |
| G13 | Todo simultáneo en una GUI, sin pestañas |
| G2 | CSV con todos los datos de sensores, en la notebook y en un disco de respaldo |
| G15 | Enviar telecomandos desde la estación |
| G9 / G10 | Autonomía ≥ 2 h con la notebook, sin CA |
| FRR | Prueba de estación: telemetría en vivo, gráficos y despliegue funcionando |

**Observación sobre G6 [I]:** el SI usa rad/s para velocidad angular, m/s² para aceleración, Pa para presión y K para temperatura (el °C es una unidad aceptada con el SI). Para evitar discusiones en la FRR, muestren la unidad **junto a cada valor** y consideren mostrar las rotaciones en rad/s (con °/s entre paréntesis si hace falta). Pueden consultarlo con CONAE.

### 5.2 Alternativas (comparativa)

Leyenda: ✅ cumple bien · ⚠️ cumple con trabajo o con restricción · ❌ problema serio.

| Opción | Una ventana sin pestañas (G13) | Fuentes/estilos (G12) | Gráficos en tiempo real (G7) | Lectura del puerto serie | CSV a 2 destinos (G2) | Comandos desde la misma pantalla (G15) | Consumo | Licencia / costo | Curva de aprendizaje | Cumple G4 |
|---|---|---|---|---|---|---|---|---|---|---|
| **1. Python + PySide6 (o PyQt) + pyqtgraph + pyserial** | ✅ | ✅ control total | ✅ rápido; pyqtgraph se apoya en NumPy y Qt GraphicsView **[V]** | ✅ `pyserial` | ✅ manual | ✅ botones propios | Bajo-medio con actualización limitada [I] | PySide6: LGPL; PyQt6: GPL o comercial **[V]** | Media | ✅ |
| **2. Python + Tkinter + matplotlib** | ✅ | ⚠️ posible pero más manual | ⚠️ lento con 6 gráficos y animación [C] | ✅ | ✅ | ✅ | Medio | Libre | Baja-media | ✅ |
| **3. Python + web local (Dash/Plotly, Streamlit, Flask + JS)** | ⚠️ una sola página es posible; hay que evitar pestañas | ✅ con CSS | ✅ | Backend en Python | ✅ | ✅ | Más alto: servidor + navegador [C] | Libre | Media | ✅ |
| **4. Navegador puro con Web Serial API + librería de gráficos** | ✅ | ✅ | ✅ | ⚠️ solo Chrome/Edge 89+; Safari no; Firefox: una fuente dice sí desde la 151 y otra no **[V, contradictorias]** | ⚠️ difícil de escribir a dos destinos | ✅ | Medio | Libre | Media | ✅ |
| **5. C# (.NET WPF/WinForms) + ScottPlot/OxyPlot** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | Bajo | Libre | Media (si ya lo conocen) | ✅ [C] |
| **6. Java/JavaFX** | ⚠️ el ejemplo público de CanSat que encontré usa pestañas **[V]** (viola G13 si se copia) | ✅ | ✅ | ✅ | ✅ | ✅ | Medio | Libre | Media | ✅ |
| **7. Qt C++/QML** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | Muy bajo | LGPL o comercial | Alta | ✅ [C] |
| **8. LabVIEW** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | Medio | La edición Community es gratuita pero **no** para uso comercial ni para enseñanza/investigación en instituciones que otorgan títulos **[V]**; zona gris para una competencia universitaria | Media | ✅ |
| **9. MATLAB App Designer** | ✅ | ✅ | ⚠️ puede ser pesado [C] | ✅ | ✅ | ✅ | Alto [C] | Licencia (puede haber acceso por la universidad) [C] | Media | ✅ |
| **10. Serial Studio (dashboard genérico)** | ✅ | ⚠️ limitado a lo que ofrecen sus widgets | ✅ (hasta 60 FPS **[V]**) | ✅ | ✅ guarda CSV **[V]** | ⚠️ limitado | Medio | Open-core: la build GPL excluye funciones Pro (XY, 3D, MQTT…); los binarios oficiales son de prueba o Pro **[V]**; hay versiones viejas bajo MIT **[V]** | Baja | **❌ Riesgo alto** (ver 5.3) |
| **11. Grafana + InfluxDB / Node-RED Dashboard** | ⚠️ paneles múltiples en un tablero, pero cuesta cumplir G12 | ⚠️ | ✅ | Necesita puente propio | ⚠️ | ⚠️ | Alto (varios servicios) [C] | Libre | Media-alta | ⚠️ |

### 5.3 Sobre usar un dashboard ya hecho (Serial Studio) y G4

Serial Studio nació justamente de competencias CanSat: su autor armó software de estación para cada proyecto y terminó haciendo una herramienta genérica configurable **[V]**. Eso lo hace muy tentador, pero:

- **G4:** "Cada equipo deberá desarrollar su propia estación terrena". Una herramienta de terceros configurada **puede no contar** como desarrollo propio. **Consulten a CONAE** antes de depender de ella. Mientras tanto es una gran herramienta de **depuración** durante el desarrollo **[I]**.
- **Licencia:** la build GPL se compila uno mismo y no incluye funciones Pro; los binarios oficiales son de prueba o requieren licencia comercial **[V]**. Una de las bifurcaciones menciona límites de sesión de 30 minutos en la versión comunitaria **[V, fuente secundaria; versión posiblemente vieja]**.
- **G12/G13:** el control fino de tipografía, trazos y layout está limitado a sus widgets.

**Ejemplos públicos de estaciones de CanSat que encontré [V]:** un proyecto con PyQt5 + pyqtgraph + pyserial (muestra datos en tiempo real), uno con JavaFX con pestañas, y uno de 2018 con React + Plotly. Sirven como referencia de UI, **no los copien** (G4).

### 5.4 Recomendación: Python + PySide6 + pyqtgraph + pyserial

**Por qué [I]:**
- Una sola ventana con layout propio y control total del estilo (G12, G13).
- Muchos equipos de CanSat y competencias similares lo usan con éxito **[V]**.
- Licencia sin trabas: PySide6 (LGPL) y PyQt6 (GPL) funcionan igual para este uso; **si en algún momento van a distribuir el programa sin código, PySide6 evita problemas** **[V]**.
- pyqtgraph apunta a PyQt5/6 y PySide2/6 y es rápido para gráficos en tiempo real **[V]**.
- Python facilita el simulador, el análisis posterior del CSV y los scripts de prueba.

**Arquitectura sugerida [I]:**

```
[radio/USB] -> hilo lector (pyserial) -> cola -> decodificador (COBS/CRC) -> estado compartido
                                                                   |-> escritor CSV principal (notebook)
                                                                   |-> escritor CSV de respaldo (disco)
                                                                   |-> registro crudo (hexa)
 GUI (timer a 5–10 Hz) lee el estado -> actualiza texto y gráficos
 Botones -> cola de comandos -> hilo lector/escritor serie -> CAL_ALT, SET_TIME, REQ_BATT...
```

**Detalles de implementación:**

| Tema | Recomendación |
|---|---|
| **Una ventana (G13)** | `QMainWindow` con una grilla (`QGridLayout`) o `GraphicsLayoutWidget` de pyqtgraph; **no usar `QTabWidget`** ni ventanas flotantes |
| **Gráficos (G7)** | 5 gráficos: altitud, tensión, corriente, aceleración (3 curvas X/Y/Z), tasa de rotación (3 curvas) |
| **Indicadores (G8)** | Panel de texto grande: tiempo de misión, temperatura, GPS (lat, lon, alt, satélites), paquetes recibidos, perdidos, estado |
| **Fuentes y estilo (G12)** | Fuente ≥ 14 pt en **todo**, incluyendo ejes y leyendas de pyqtgraph; trazos gruesos (`pg.mkPen(width=3)`), ejes en negrita, fondo claro y texto oscuro |
| **Unidades SI (G6)** | Etiquetar cada valor y cada eje (m, Pa, °C, V, A, m/s², rad/s) |
| **Pantalla** | Diseñar para la resolución real de la notebook (ej. 1366×768). Con 14 pt, 5 gráficos y un panel de 12 indicadores **el espacio es justo**: prueben temprano **[I]** |
| **Actualización** | Limitar el redibujado a 5–10 Hz (datos a 1 Hz); evitar OpenGL; ventana deslizante de datos para no acumular memoria |
| **CSV (G2)** | Dos manejadores de archivo (disco interno y respaldo); `flush()` + `os.fsync()` por línea; si el disco de respaldo se desconecta, **seguir escribiendo en el principal y avisar en pantalla** |
| **Reconexión** | Detectar por VID/PID, reintentar solo cuando se desconecta el USB |
| **Modo simulador** | Reproduce un CSV de descenso (con ruido, pérdidas y errores de CRC) para probar sin radio ni CanSat |
| **Registro de eventos** | Registrar comandos enviados y confirmaciones (con hora) |
| **Empaquetado** | Dejar el entorno virtual armado y una copia offline de las dependencias (sin internet en el campo); probar en una notebook "limpia" [C] |
| **Dependencias** | Fijar versiones (`requirements.txt`) y tener un respaldo del instalador |

### 5.5 Consumo de energía del software

Datos a 1 Hz y un redibujado de 5–10 Hz son una carga muy chica para el procesador; el consumo dominante es la **pantalla a máximo brillo** **[I]**. Para validarlo:
1. Cargar la notebook al 100 %.
2. Brillo máximo, modo de energía "equilibrado" o "mejor rendimiento" (sin que baje el brillo), simulador + CSV a dos discos + puerto USB con la radio.
3. Cronometrar hasta quedar con, por ejemplo, 20 %. Extrapolar a 2 h con un margen amplio **[C]**.

### 5.6 Pruebas del punto 6

- [ ] Prueba de la FRR en seco: telemetría en vivo, gráficos y despliegue funcionando (usando el simulador y luego la radio real).
- [ ] Leer la pantalla **afuera, a pleno sol** (G12).
- [ ] Verificar que ningún texto baje de 14 pt y que ningún elemento esté en una pestaña.
- [ ] Matar el cable USB, el disco de respaldo y el simulador en medio de la prueba: ¿se recupera?
- [ ] Autonomía de 2 h (ver 5.5).
- [ ] Dos corridas completas simulando un descenso de 200 s con pérdida del 10 % de paquetes y comprobar `perdidos`.
- [ ] Probar en la notebook real de la competencia (versión de Windows, drivers, resolución).

---

## 6. Matriz de decisión conjunta: según la radio que salga del punto 1

| Si la radio es… | Interfaz (punto 4) | Protocolo (punto 5) | Notas |
|---|---|---|---|
| **LoRa SPI (RFM95W/SX1276, SX1262)** con ESP32/RP2040 como puente (opciones B/C) | USB-CDC de la placa o USB-UART CP2102N | RadioHead (opción E) o paquete propio binario + CRC-16; payload ≤ 251 B | Máx. ~100 mW (+20 dBm): baja corriente. SF7–SF9 a 125 kHz para 1 Hz. Hay stock local de módulos y placas (3.6) |
| **E-byte E32-915 (UART)** | Opción A con CP2102N | Binario ≤ 58 B + CRC-16; con COBS si es transparente; modo "fixed" con dirección y canal | Datos del módulo [V]; **verificar la variante 915 MHz y su corriente**; no se encontró oferta local verificada |
| **RFD900x (SiK)** | Opción A con FTDI/USB-UART 3,3 V; alimentar con 5 V | MAVLink o paquete propio; es un túnel serie transparente | El modelo bloqueado para Australia cubre 915–928 MHz y el de EE. UU. cubre 902–915 MHz según el datasheet **[V]**: **ojo que solo el rango 915–928 coincide con X1**. Pico de ~1 A; importación (≈ USD 199) |
| **XBee-PRO 900HP** | Opción A (puerto serie), modo API | API frames de Digi (0x7E, checksum, 64 bits) + CRC propio | TX 215 mA, RX 29 mA, 2,1–3,6 V **[V]**; sin oferta local verificada; revisar variante regional |
| **ESP-NOW (2,4 GHz)** | Opción D: ESP32 como puente USB | Unicast por MAC; payload ≤ 250 B; ACK de capa MAC | Cuidar canal (evitar 14) y la coexistencia con Wi-Fi del campo; alcance a validar en el punto 1 |
| **nRF24L01+ (2,4 GHz)** | Opción E con MCU | Payload ≤ 32 B: reducir el paquete o fragmentar | Canales ≤ 83 (2483 MHz) para respetar X1 [I]; alcance a validar en el punto 1 |

---

## 7. Plan de acción sugerido (orden)

1. Con el equipo de radio: decidir la radio (punto 1) y confirmar su **corriente pico** y su **límite de payload**.
2. Definir el **layout del paquete** con Aviónica (campos, resoluciones, rangos de sensores).
3. Escribir el **simulador de telemetría** (genera CSV y flujo serie con pérdida y errores).
4. Armar el **software de la estación con el simulador** (layout, fuentes, CSV, comandos).
5. Probar la interfaz USB-UART con la notebook real y medir corriente.
6. Integrar con la radio real y medir tiempos de aire.
7. Pruebas de autonomía y a pleno sol.
8. Documentar protocolo y comandos (manual de la misión, F4) y los supuestos TBR en el PDR.

---

## 8. Preguntas para consultar a CONAE (relevantes a los puntos 4, 5 y 6)

1. **G4:** ¿se puede usar una herramienta de terceros (por ejemplo, un dashboard genérico) como base de la estación terrena, o debe ser código propio?
2. **G9/G10:** ¿la radio de la estación puede alimentarse de un hub USB con alimentación propia o un power bank, o debe salir de la batería de la notebook?
3. **G13:** ¿Cuenta como "pestaña" una barra de desplazamiento o un panel plegable? ¿Se acepta que algunos controles estén en un diálogo aparte (por ejemplo, el de configuración de comandos)?
4. **G6:** ¿Se acepta °/s y °C, o debe mostrarse rad/s y K? (O ambas.)
5. **G2:** ¿Qué se considera "disco de respaldo"? ¿Vale un pendrive USB?
6. **X2:** ¿Qué entienden exactamente por "modo de difusión"? ¿Alcanza con direccionamiento unicast y canal asignado?
7. **X3/G5:** ¿Hay que transmitir telemetría antes de la separación (en el cohete)? Esto define la ventana de enlace dentro del cohete (ya estaba en la lista de ambigüedades).
8. **Banda/potencia:** ¿límite de potencia de ENACOM y asignación de canales por equipo? (compartido con el punto 1).

*Recuerden que las consultas por mail se hacen durante el período de inscripción: verifiquen si todavía están a tiempo.*

---

## 9. Fuentes consultadas

**Radios y módulos**
- RFD900x (hoja de datos y fichas): https://www.scribd.com/document/791617197/RFD900x · https://irlock.com/products/rfd-900x-modem-bare · https://irlock.com/products/rfd900x-us-telemetry-modem-fcc-approved
- XBee-PRO 900HP (datasheet): https://lv.mouser.com/pdfdocs/Digi_International_XbeePro_900HP_Datasheet.pdf
- E-byte E32 (modos, corriente, buffer): https://mischianti.org/lora-e32-device-for-arduino-esp32-or-esp8266-specs-and-basic-usage-part-1/ · https://www.globalsources.com/product/lora-module_1219470283f.htm · https://makersportal.com/shop/lora-915mhz-module-ebyte · https://radiokot.ru/forum/download/file.php?id=331188
- RFM95 / T-Beam: https://www.adafruit.com/products/5684 · https://produto.mercadolivre.com.br/MLB-1521545945-ttgo-tbeam-esp32-gps-lora-915-mhz-e-suporte-de-bateria-_JM
- ESP-NOW (Espressif): https://docs.espressif.com/projects/esp-idf/en/latest/esp32c61/api-reference/network/esp_now.html
- nRF24L01+: https://people.ece.cornell.edu/land/courses/ece4760/StudentWork/Jaehyun_Kim_jk726/Transceiver_nRF24L01_plus_Analysis.pdf · https://msalamon.pl/en/radio-communication-using-nrf24l01-modules-part-1/

**Protocolos**
- RadioHead (documentación y código): https://www.airspayce.com/mikem/arduino/RadioHead/classRHReliableDatagram.html · https://os.mbed.com/users/davidr99/code/RadioHead-148/docs/tip/classRH__RF95.html
- MAVLink 2: https://mavlink.io/en/guide/serialization.html · https://mavlink.io/en/about/overview.html
- COBS: https://docs.rs/cobs_codec_rs · https://pub.dev/packages/cobs_codec

**USB y energía**
- Límites de corriente USB: https://developerhelp.microchip.com/xwiki/bin/view/applications/usb/power-delivery/ · https://www.ti.com/document-viewer/lit/html/SLUAA94/GUID-B0D36C0B-E26F-4684-8163-777D2BD2BF11
- Chips USB-UART: https://learn.adafruit.com/adafruit-cp2102n-cp2104-friend-usb-to-serial-converter/overview · https://microcontrollerslab.com/best-usb-to-serial-adapters-buying-guide/ · https://zbotic.in/top-arduino-nano-clones-in-india-ch340-vs-cp2102-compared/

**Software de estación**
- Serial Studio: https://github.com/fork200/Serial-Studio · https://github.com/yangfrchi/Serial-Studio · https://www.hackster.io/alex-spataru/serial-studio-a-dashboard-software-for-serial-devices-0866b7
- PySide6/PyQt6 (licencias): https://www.pythonguis.com/faq/licensing-differences-between-pyqt6-and-pyside6/
- pyqtgraph: https://github.com/pyqtgraph/pyqtgraph
- Web Serial API: https://caniuse.com/web-serial · https://web-platform-dx.github.io/web-features-explorer/features/serial/
- LabVIEW Community: https://www.ni.com/labviewse · https://forums.ni.com/t5/LabVIEW/Watermarks-have-started-to-appear/m-p/4478049
- Estaciones de CanSat públicas (referencia de UI): https://github.com/darods/CanSat-Ground-station · https://github.com/Jatin7385/CANSAT_GCS · https://github.com/MStou/CanSat2018-Ground-Station

**Precios y disponibilidad en Argentina**
- Mercado Libre AR: https://listado.mercadolibre.com.ar/ttgo-lora · https://listado.mercadolibre.com.ar/usb-ttl-cp2102 · https://listado.mercadolibre.com.ar/usb-ttl-ft232 · https://listado.mercadolibre.com.ar/conversor-usb-ttl-ch340
- Unoelectro: https://www.unoelectro.com.ar/productos/heltec-wifi-lora-32-v3-1-868-915-mhz-oled-unoelectro/
- Oriel: https://oriel.com.ar/producto/convertidor-usb-a-ttl-cp2102/

---

## 10. Lo que NO verifiqué y conviene confirmar

- Corriente de TX real de la variante **E32-915T30D** (usé la de la familia/433 MHz como referencia).
- Si el **XBee-PRO 900HP** actual tiene una variante apta para la banda 915–928 MHz de Argentina y quién lo vende localmente.
- Disponibilidad y precio **argentino** de power banks, de radios E32/RFD900x/XBee y de módulos nRF24/SX1280 (no aparecieron ofertas locales verificables).
- Estructura del **CCSDS Space Packet** y detalles de **LoRa/SX1262** más allá del cálculo de tiempo en el aire: están marcados [C]/[I].
- Consumo real de los puentes ESP32/RP2040: hay que medirlo.
- Normativa de **ENACOM** (potencia, canales): es del punto 1 y quedó fuera del alcance.
- Soporte de Web Serial en Firefox: las dos fuentes que encontré se contradicen.
