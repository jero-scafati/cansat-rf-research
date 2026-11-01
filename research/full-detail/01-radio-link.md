# Enlace de radio CanSat ⇄ estación terrena: alternativas para 1A (telemetría) y 1B (telecomandos)

Documento de investigación para el PDR. Fecha de elaboración: 26 de octubre de 2026.

**Alcance:** acciones 1A (CanSat → estación, telemetría a 1 Hz) y 1B (estación → CanSat, telecomandos) de `investigacion-sota.md`. Antenas (2 y 3), interfaz con la notebook (4), protocolo (5) y software de estación (6) aparecen solo donde condicionan la elección de la radio.

## 0. Cómo leer este documento

Marqué el origen de cada dato para que sepas cuánto confiar en él.

| Marca | Significado |
|---|---|
| **[F]** | Dato tomado de una fuente consultada (datasheet, manual, ficha de tienda, resolución). Las fuentes están al final. |
| **(est.)** | Cálculo o estimación mía a partir de fuentes. Está explicado cómo se hizo. |
| **(verificar)** | Dato que conozco de memoria o que una fuente da de forma ambigua. Confirmarlo en el datasheet antes de ponerlo en el PDR. |
| **(inferido)** | Conclusión mía a partir de varios datos. |

Los precios son de listados internacionales de 2025–2026. **No pude confirmar precios ni stock en comercios argentinos** con las búsquedas que hice. Las conversiones a pesos usan el dólar oficial del Banco Nación (≈ $1.540 el 20 de octubre de 2026 **[F]**) y son solo una orden de magnitud: no incluyen envío ni impuestos de importación.

---

## 1. Resumen ejecutivo

### Recomendación

1. **Opción principal: módulo LoRa de 915 MHz con chip SX1276 o SX1262, manejado por vuestro propio firmware.** Ejemplo: HopeRF RFM95W‑915 (SX1276, ≈ USD 8–11 **[F]**). Configuración de partida: BW 250 kHz, SF9, CR 4/5. Con ella un paquete de 49 bytes ocupa ≈ 164 ms en el aire (est.) y el margen de enlace a 3 km es ≈ 45 dB (est.). Consume ≈ 30 mA promedio (est.), pesa unos 3 g **[F]** y cuesta una fracción de C9. El costo es desarrollo: hay que escribir el direccionamiento y la validación de paquetes.
2. **Opción de integración rápida: módulo LoRa con interfaz UART** (EBYTE E32‑915T20D o E22‑900T30D, o REYAX RYLR998). Se manejan con comandos serie y traen direccionamiento. Pierden control fino del tiempo en el aire, y los EBYTE necesitan fuente robusta si se usan a 30 dBm.
3. **Opción robusta y probada, pero cara o difícil de conseguir: XBee‑PRO 900HP (variante Australia, 915–927 MHz) o radios SiK/RFD900x.** Tienen salto de frecuencia, direccionamiento y reintentos incorporados. A 3 km quedan con ≈ 25 dB de margen (est.), suficiente. El problema es precio, plazo de entrega y la banda (ver hallazgo 1).
4. **2,4 GHz no lo recomiendo como enlace principal.** Con antena dentro del contenedor y 2–3 km de distancia, nRF24L01+ sin PA, XBee3 estándar y ESP‑NOW no tienen margen (est.). SX1280 y XBee3 PRO tienen margen justo (6–18 dB a 3 km, est.).

### Tres hallazgos que cambian la elección

1. **La banda de 902–915 MHz no está declarada de uso compartido en Argentina.** La Res. 581/2018 declara de uso compartido solo 915–928 y 2400–2483,5 MHz **[F]**. Muchas radios "900 MHz" de EE. UU. (XBee‑PRO 900HP versión US, SiK, RFD900x) saltan por 902–928 MHz por defecto. Hay que usar la variante australiana (915–927 MHz, XBee‑PRO 900HP **[F]**) o limitar el rango por configuración (SiK/RFD900x: parámetros de frecuencia mínima y máxima, verificar). Los módulos LoRa se sintonizan a cualquier canal, así que acá no hay problema si se elige bien el canal.
2. **El "modo broadcast" de X2 puede estar prendido por defecto.** En los EBYTE E32, la dirección 0x0000 o 0xFFFF es dirección de difusión **[F]**, y 0x0000 es el valor de fábrica. Hay que cambiarla a una dirección propia y usar el modo de transmisión fija. En los XBee, 0x000000000000FFFF es la dirección de difusión **[F]**: usar unicast con la dirección de 64 bits del destino.
3. **El presupuesto C9 es chico en dólares.** $1.000.000 equivalen a ≈ USD 650 (est.). Una RFD900x (USD 199 **[F]**) se come un tercio. Un módulo LoRa de USD 10 es irrelevante en el presupuesto.

---

## 2. Requisitos que condicionan la elección

| Req. | Qué exige | Efecto sobre la radio |
|---|---|---|
| X1 | Radios en 915–928 MHz o 2400–2483,5 MHz | Descarta 433 MHz, 868 MHz y 902–915 MHz |
| X2 | Sin modo broadcast | Direccionamiento punto a punto en cada paquete y comando |
| X3 | Comando automático al separarse del cohete | La radio debe poder transmitir un mensaje de inicio de misión |
| X4, C5 | Telemetría una vez por segundo | Tiempo en el aire bien por debajo de 1 s |
| X5 | Altitud, presión, temperatura, aceleraciones, rotaciones, GPS (lat, lon, alt, satélites) | Define el tamaño del paquete |
| X6, F4, G1, F3, G15 | Telecomandos (calibrar altitud, configurar hora, reporte de batería y paneles, información adicional) | La radio debe recibir mientras transmite 1 Hz: ventana de escucha |
| G14, G8, F1 | Paquetes recibidos con éxito y perdidos; contador de transmitidos que sobrevive reinicios | El paquete necesita contador y validación de integridad (CRC) |
| E7 | Funcionar al menos 2 h integrado en el cohete | Consumo en espera y recepción continua |
| E1, E2, M4 | Sin LiPo; litio con empaque metálico tipo 18650; sin contactos de resorte | Condiciona la tensión de alimentación disponible |
| S1, S2, S3 | 500 ± 10 g; Ø 90 mm; 200–250 mm | Masa y volumen del módulo y su antena |
| S4, S5, S8 | Vibración 15 G; impacto 30 G; montaje fijo | Soldar o fijar el módulo; evitar conectores de pin sin sujeción |
| C9 | CanSat < $1.000.000 (la estación terrena no cuenta) | Costo de la radio del CanSat |
| Bases §6 | La FRR verifica que la frecuencia sea la asignada | Poder cambiar de canal sin reprogramar de cero |

---

## 3. Supuestos de diseño

Todos son supuestos míos, a marcar como TBR en el PDR.

| Supuesto | Valor | Justificación |
|---|---|---|
| Altitud máxima | 500 m (documentos) y 1 km (reunión) | Las Bases dicen solo "superior a 500 m". Diseñar el margen con 1 km |
| Deriva horizontal | ≈ 1,7 km con viento de 10 m/s y descenso de ≈ 170 s | Cálculo de `investigacion-sota.md` |
| Distancia en línea recta | ≈ 2 km (con 1 km de altura y 1,7 km de deriva, √(1² + 1,7²) = 1,97 km) | Cálculo mío |
| Distancia de diseño del enlace | **3 km** | Margen sobre los 2 km |
| Tiempo de enlace requerido | 56–208 paquetes en el descenso (C3/C4) más el tiempo en plataforma y ascenso | `investigacion-sota.md` |
| Antena del CanSat | −3 dBi efectivos (dentro del contenedor, rotando) | Peor caso, est. |
| Antena de la estación | +3 dBi (omnidireccional) o +9 dBi (direccional) | Dos casos |
| Pérdidas extra | 6 dB (fuselaje del cohete, contenedor, polarización, conectores) | Estimación conservadora |
| Margen mínimo deseado | ≥ 10 dB, idealmente ≥ 15 dB | Práctica habitual para enlaces con desvanecimiento |

---

## 4. Normativa argentina que importa

| Tema | Qué dice | Fuente |
|---|---|---|
| Bandas de uso compartido | 915–928 MHz y 2400–2483,5 MHz (entre otras) se declaran de uso compartido, sin autorización para su uso | Res. 581/2018 **[F]** |
| Potencia máxima | En 915–928 MHz y en 2400–2483,5 MHz: **30 dBm conducidos y 36 dBm de PIRE** | Anexo de la Res. 581/2018 **[F]** |
| Antenas con ganancia direccional > 6 dBi | La potencia del transmisor se reduce en tantos dB como la ganancia exceda 6 dBi | Anexo Res. 581/2018 **[F]** |
| Punto a punto fijo en 2,4 GHz | Se permite ganancia > 6 dBi reduciendo 1 dB de potencia por cada 3 dB de exceso | Anexo Res. 581/2018 **[F]** |
| Interferencia | Las emisiones no pueden interferir a servicios primarios o secundarios; si interferís, tenés que cesar | Res. 581/2018 **[F]** |
| Homologación | Los equipos de baja potencia en 915–928, 2400–2483,5 y 5725–5850 MHz deben cumplir ENACOM‑Q2‑60.14 V20.1 | Res. 551/2021 **[F]** |
| 902–915 MHz | La Res. 4653/2019 deja sin efecto resoluciones anteriores en todas las bandas salvo el rango 902–915 MHz **[F]**; ese rango no figura como compartido | Res. 1132/2023 **[F]** |
| Cambios recientes | Hay una Res. 57/2026 sobre el marco de certificación **[F]** (solo vi el título) | ib‑lenhardt **[F]** |

**Consecuencias prácticas:**

- Un módulo de 30 dBm (E32‑915T30D, RFD900x a plena potencia) con antena de ≤ 6 dBi cumple (PIRE ≤ 36 dBm). Con una antena direccional de 9 dBi en la estación habría que bajar 3 dB la potencia **del transmisor de la estación** (el de los comandos). La antena del CanSat es pequeña, así que no hay problema.
- La homologación comercial (ENACOM‑Q2‑60.14) afecta a la venta de equipos en Argentina. **No sé si CONAE la exige a los equipos del concurso.** Preguntarlo junto con los demás puntos. Ver sección 15.
- Mantener todos los canales dentro de 915–928 MHz con un margen de ancho de banda en los bordes. Por ejemplo, centros entre 916 y 927 MHz.

---

## 5. Paquete de telemetría (1A)

Tamaño del paquete: lo calculé campo por campo. Define el tiempo en el aire.

| Campo | Bytes | Req. |
|---|---:|---|
| ID de equipo / destinatario | 1 | X2 |
| Estado (stand‑by / ascenso / descenso / aterrizado) + estado del software de vuelo + bandera de inicio de misión | 1 | G8, X3 |
| Tiempo de misión (segundos, uint32) | 4 | G3, F2 |
| Contador de paquetes transmitidos (uint16) | 2 | F1 |
| Altitud barométrica (float32) | 4 | X5, SN1 |
| Presión atmosférica (uint32, Pa) | 4 | X5 |
| Temperatura (int16, 0,01 °C) | 2 | X5, SN2 |
| Aceleración 3 ejes (3 × int16) | 6 | X5, G7 |
| Velocidad angular 3 ejes (3 × int16) | 6 | X5, G7 |
| GPS latitud + longitud (2 × int32) | 8 | X5 |
| GPS altitud (int32) | 4 | X5 |
| GPS satélites rastreados (uint8) | 1 | X5 |
| Tensión de batería (uint16, mV) | 2 | G7, SN3 |
| Corriente de batería (uint16, mA) | 2 | G7, SN8 |
| CRC‑16 | 2 | G14 |
| **Total** | **49** | |

- Con un byte más para "último comando recibido" (confirmación implícita, ver sección 6) son **50 bytes**. Planificar con **50 a 64 bytes** para dejar margen.
- Un paquete equivalente en texto ASCII/CSV mide ≈ 110 bytes (ejemplo armado por mí). Mucho más largo en el aire. G5 pide mostrar la telemetría en texto **en la pantalla**, no transmitirla en texto.
- Si el contador de paquetes es uint16 se desborda a los 65.535 paquetes (≈ 18 h a 1 Hz), más que suficiente para 2 h + vuelo.

---

## 6. Telecomandos (1B) y ventana de escucha

### 6.1 Comandos mínimos (de `investigacion-sota.md`)

| Comando | Argumentos | Bytes aprox. (con ID, secuencia y CRC) | Req. |
|---|---|---:|---|
| Calibrar altitud a cero | ninguno | 8 | G1, F5 |
| Configurar hora | hora UTC (uint32) | 12 | F3, F2 |
| Pedir reporte de batería y paneles | ninguno | 8 | X6 |
| Pedir información adicional | código (1–2 B) | 9 | F4 |
| (propio) Cambiar de canal / modo | canal (1 B) | 9 | Bases §6 |

Estructura sugerida: `[destino][origen][nº de secuencia][código][argumentos…][CRC‑16]`. Un comando mide **8–14 bytes**.

### 6.2 Cómo recibir mientras se transmite a 1 Hz

Casi todas las radios candidatas son **half‑duplex** (no transmiten y reciben a la vez). La solución es temporal:

```
|<-- 1 s ---------------------------------------------------->|
| TX telemetría (≤ 200 ms) | RX ventana de comando (≥ 300 ms) | reposo/RX |
```

- Después de cada paquete de telemetría el CanSat escucha. La **estación manda el comando justo después de recibir un paquete**, así que no necesita adivinar el momento y no hay colisiones.
- La confirmación del comando (ACK) viaja en el siguiente paquete de telemetría, en un byte "último comando ejecutado". Los documentos no piden ACK; es una decisión vuestra, y esta es la más barata.
- Presupuesto con LoRa BW250/SF9: telemetría 164 ms + conmutación + comando de 12 bytes 72 ms (est.) ≈ 250–300 ms de 1000 ms.
- En la plataforma (comandos G1 y F3), la estación puede reintentar hasta que el estado confirme la ejecución.

### 6.3 Particularidades por familia

| Familia | Cómo hace el sentido inverso |
|---|---|
| LoRa SPI (RFM95W, SX1262) | Vuestro firmware conmuta TX → RX. Se puede quedar en RX continuo y pasar a TX solo 164 ms por segundo |
| LoRa UART (E32, E22, RYLR998) | El módulo está en recepción cuando no transmite; recibir un comando entrega los bytes por la UART. El módulo E32 demora ≈ 5 ms entre recibir y sacar los datos por la UART **[F]** |
| XBee‑PRO 900HP | El módulo maneja el medio (FHSS, reintentos). Se envían comandos con la misma API que la telemetría |
| SiK / RFD900x | Enlace serie transparente con ventanas de tiempo alternadas (TDM). Parece full duplex para el usuario |
| nRF24L01+ | **ACK con carga útil**: el receptor (estación) adjunta un comando de hasta 32 bytes al ACK del paquete de telemetría, sin conmutar de modo. Muy elegante, pero la banda es 2,4 GHz |
| ESP‑NOW | Ambos lados pueden enviar; half‑duplex en el aire |

---

## 7. Fichas de las alternativas

### 7.1 Familia sub‑GHz (915–928 MHz)

#### A. HopeRF RFM95W‑915 (SX1276), LoRa SPI

- **Banda:** 868/915 MHz; sintonizable por software a cualquier canal **[F]**. Sensibilidad hasta −148 dBm (a velocidades de bits muy bajas). Potencia +20 dBm (100 mW). Velocidad: FSK hasta 300 kbps, LoRa hasta 37,5 kbps **[F]**.
- **Consumo:** RX 10,3–12,1 mA; TX 20–120 mA a +20 dBm; reposo ≈ 1 µA; alimentación 1,8–3,7 V **[F]**.
- **Tamaño y masa:** 16 × 16 × 2 mm; 3 g **[F]** (ficha de un revendedor, verificar).
- **Interfaz:** SPI. Librerías abiertas (Sandeep Mistry `LoRa`, RadioHead, RadioLib).
- **Alcance reportado:** ≈ 2 km en línea de vista con alambre simple; 20 km con antenas direccionales y configuración óptima (Adafruit **[F]**).
- **Dúplex:** half‑duplex. Se conmuta TX/RX por registro.
- **Direccionamiento:** el modo LoRa de la SX127x no filtra direcciones por hardware (verificar). Se arma en software (byte de destino) y se aísla con palabra de sincronía (*sync word*) y canal.
- **Cambio de canal:** por firmware (frecuencia en registro).
- **Precio:** USD 6,5–11 (SOS electronic ≈ € 6,5 sin IVA; Adafruit USD 10,95; LCSC ≈ USD 7,7) **[F]**. En pesos: ≈ $12.000–17.000 (est., sin envío ni impuestos). El módulo suelto se vende en tiendas latinoamericanas (uelectronics.com lo lista; **no verifiqué que envíe a Argentina**).
- **Pros:** bajísimo costo y masa; mejor sensibilidad por dólar; control total del tiempo en el aire; consumo bajo; mucha documentación.
- **Contras:** hay que escribir el protocolo (dirección, CRC propio o el del modem, reintentos); sin certificación incluida; cuidado con la antena (no transmitir sin antena).

#### B. EBYTE E32‑915T20D / E32‑915T30D (SX1276, UART)

- **Banda:** módulo diseñado para 900–931 MHz, 32 canales; el valor por defecto es el canal 0x0F, 915 MHz **[F]**. El paso entre canales es 1 MHz (inferido por analogía con la fórmula 410 + CHAN para la versión de 433 MHz). **Dentro de la banda argentina quedan los canales 15 a 28** (915–928 MHz); recomiendo 16 a 27 (inferido, verificar en el manual).
- **Potencia:** T20D 20 dBm (19,3–20,6); T30D 30 dBm (29,4–30,8) **[F]**.
- **Sensibilidad:** −144 a −148 dBm a 0,3 kbps con SF12 **[F]**. A 2,4 kbps (valor por defecto) es peor; la sensibilidad baja al subir la velocidad.
- **Velocidades en el aire:** 0,3 / 1,2 / **2,4** (por defecto) / 4,8 / 9,6 / 19,2 kbps. Subpaquete de 58 bytes; un paquete de 49–50 bytes cabe en uno **[F]**.
- **Consumo:** T20D TX 110–130 mA, RX 13–15 mA, apagado 3–5 µA, alimentación 2,3–5,2 V. T30D TX 650–770 mA, RX 20–22 mA, alimentación 3,3–5,2 V (5 V recomendado) **[F]**. La ficha pide fuente de más de 1 A y ondulación < 100 mV para la versión de 30 dBm **[F]**.
- **Masa y tamaño:** T20D 21 × 36 mm, 6,7 g; T30D 24 × 43 mm, 8,2 g **[F]**. Existen versiones SMD (T20S 16 × 26 mm, 2,1 g; T30S 25 × 40,5 mm, 5,3 g) **[F]**.
- **Alcance reportado:** T20D ≈ 3 km, T30D ≈ 8 km, con antena de 5 dBi a 2,5 m de altura, paquetes de 30 bytes cada 2 s, pérdidas < 5 % **[F]**.
- **Dúplex:** half‑duplex, automático. Modo 0 (normal): transmite cuando recibe bytes por la UART y recibe el resto del tiempo **[F]**.
- **Direccionamiento:** modo de transmisión fija. Los primeros 3 bytes de cada trama son dirección alta, dirección baja y canal del destino **[F]**. **La dirección 0x0000 y la 0xFFFF son de difusión (broadcast) y monitoreo. Usar una dirección propia distinta de esas dos** (X2).
- **Cambio de canal:** comando serie C0/C2 en modo 3 (sleep), a 9600 8N1 **[F]**. Requiere acceso a la UART del módulo (dejar un conector o hacerlo desde el microcontrolador).
- **Corrección de errores:** FEC activable **[F]**. Cifrado y compresión incluidos **[F]**.
- **Precio:** ≈ USD 9–11 (RobotShop, T30D USD 10,57, T30S USD 10,02) **[F]**; en pesos ≈ $15.000 (est.).
- **Pros:** el más fácil de integrar (UART transparente); direccionamiento incluido; cambio de canal por comando; FEC.
- **Contras:** el tiempo en el aire no se controla con precisión (buffer interno, espera de 3 bytes para cerrar el paquete); el T30D exige 5 V y picos de 0,7–0,8 A (problema con baterías de 18650 sin boost); ficha del módulo de 2018 (versión V8 actual puede diferir).

#### C. EBYTE E22‑900T30D (SX1262, UART) y E220‑900T30D (LLCC68)

- **E22‑900T30D:** SX1262, banda 850–930 MHz, hasta 30 dBm, velocidad en el aire 2,4–62,5 kbps, sensibilidad −134 dBm a 2,4 kbps (SF11), alcance "hasta 10 km en condiciones ideales", alimentación 2,6–5,5 V (potencia plena a ≥ 5 V). Modos: transparente, **transmisión dirigida a dirección y canal**, WOR, **LBT (escuchar antes de transmitir)**, RSSI, relé automático, clave de cifrado, sueño profundo de 3 µA **[F]**.
- **E220‑900T30D:** chip LLCC68, 915 MHz, hasta 10 km declarados **[F]**. El LLCC68 es una versión de LoRa con factores de dispersión limitados (verificar).
- **Precio:** E22‑900T30D € 12,29 (Rotorama); E220‑900T30D USD 9,66 (RobotShop) **[F]**.
- **Consumo:** el SX1262 consume mucho menos en RX (4,2–10,1 mA) y TX (32–118 mA según potencia) que el SX1276 **[F]**. El módulo E22 de 30 dBm sigue necesitando 5 V y picos altos (verificar en su manual).
- **Pros:** mejor sensibilidad que E32; LBT ayuda a la convivencia en el campo con otros equipos; direccionamiento dirigido por hardware; mismo esquema UART.
- **Contras:** no vi el manual completo (verificar corriente, masa, subpaquete, canales exactos en 915 MHz); mismo problema de alimentación a 30 dBm.

#### D. REYAX RYLR998 (LoRa UART con comandos AT)

- **Banda:** 820–960 MHz (versión 868/915); potencia −20 a +22 dBm; sensibilidad −129 dBm; alimentación 1,8–3,6 V **[F]**.
- **Consumo:** TX 140 mA a +22 dBm; RX 17,5 mA; sueño 15 µA; modo de recepción inteligente ≈ 2,7 mA promedio **[F]**.
- **Tamaño y masa:** 32 × 14 × 2,2 mm; 2 g, con antena integrada (versión M4: conector I‑PEX) **[F]**.
- **Direccionamiento:** `AT+ADDRESS` (0–65535), `AT+NETWORKID`, `AT+SEND=<dirección>,…`: **unicast por dirección** **[F]**. Parámetros de modulación con `AT+PARAMETER`. Certificaciones FCC, NCC y CE declaradas **[F]**.
- **Alcance reportado:** hasta 20 km en buen entorno (fabricante) **[F]**.
- **Precio:** USD 11,99 (Lemos International) **[F]** ≈ $18.500 (est.).
- **Pros:** el más liviano y pequeño; direccionamiento y red incluidos; fácil con Arduino y ESP32.
- **Contras:** sensibilidad peor que los SX1276 crudos en la ficha (−129 dBm); no controlás el paquete (los comandos AT agregan encabezado); menos documentación sobre el tiempo en el aire.

#### E. Digi XBee‑PRO 900HP (S3B)

- **Banda:** 902–928 MHz con salto de frecuencia, máscara de canales configurable **[F]**. **Variante Australia (sufijo ‑022, 915–927 MHz)**: XBP9B‑DMWT‑022, ‑DMST‑022, ‑DMUT‑022 (DigiMesh 200 kbps) y ‑DPxT‑021 (punto a multipunto 10 kbps) **[F]**. La versión US (‑002) cubre 902–928 MHz y **sale de la banda argentina**. En Brasil hay una versión 905/920 MHz (‑012) **[F]**.
- **Potencia:** hasta 24 dBm (250 mW), seleccionable. Sensibilidad −101 dBm a 200 kbps y −110 dBm a 10 kbps **[F]**.
- **Consumo:** TX 215 mA, RX 29 mA, sueño 2,5 µA; alimentación 2,1–3,6 V **[F]**.
- **Interfaz:** UART 3 V o SPI; modo AT o API (tramas con dirección de destino de 64 bits) **[F]**. Carga útil máxima 255 bytes (consultar NP) **[F]**.
- **Direccionamiento:** **unicast con dirección de 64 bits y ACK con reintentos**; la dirección 0x000000000000FFFF es difusión (no usar) **[F]**.
- **Alcance reportado:** hasta 14 km (9 millas) en línea de vista con la antena dipolo; 28 millas con antenas de alta ganancia **[F]**. En un ensayo de campo, LoRa y XBee dieron alcances similares en un entorno urbano, aunque LoRa es más configurable **[F]**.
- **Precio y plazo:** lanzamiento USD 39–42 (2012) **[F]**; Mouser lista la XBP9B‑DMWTB002 (programable, antena de alambre, versión US) a USD 61,11 **[F]** y en el sitio argentino de Mouser aparece en pesos (≈ $100.000, sin confirmar variante ni impuestos). DigiKey indica plazos estándar de 17 a 25 semanas en kits de 900HP y de 12 a 17 semanas en XBee3 **[F]**.
- **Pros:** probado, robusto; salto de frecuencia ayuda a convivir con otros equipos; unicast y reintentos incorporados.
- **Contras:** la variante AU es difícil de conseguir (encontré una ficha "retirada" en Core Electronics y el listado de Digi **[F]**); plazo de entrega largo; en un ensayo con modo API y tramas pequeñas se vio degradación de rendimiento con intervalos cortos entre tramas (foro Digi **[F]**); más caro por mucho.

#### F. RFDesign RFD900x / RFD900ux y radios SiK (3DR, Holybro)

- **RFD900x:** 902–928 MHz (US) / 915–928 MHz (Australia); 1 W (+30 dBm) en pasos de 1 dB; velocidades en el aire de 12 a 750 kbps (500 kbps en versiones nuevas); sensibilidad −111 dBm a 12 kbps, −105 a 64 kbps, −98 a 200 kbps; 5 V (3,5–5,5 V), ≈ 800 mA de pico a máxima potencia, RX/espera ≈ 60 mA; 14,5 g; dos conectores RP‑SMA con diversidad; firmware SiK de código abierto, AES‑128 **[F]**.
- **RFD900ux:** 10 g, dos conectores u.FL, USD 83 **[F]**.
- **SiK 3DR / Holybro V3 (100 mW):** −121 dBm a 2 kbps y −105 dBm a 64 kbps; 100 mA en TX y 25 mA en RX; 14,3 g (versión Air) a 23,5 g con antena; corrige hasta 25 % de errores de bit **[F]**. Existe una versión Holybro de largo alcance con identificación FCC para **915,06–927,04 MHz** **[F]**: es compatible con la banda argentina.
- **Dúplex y direccionamiento:** enlace serie transparente punto a punto con identificación de red (NETID, verificar).
- **Precio:** RFD900x USD 199 (IRLock) o AUD 169,50 **[F]** ≈ $300.000 (est., sin envío ni impuestos); Holybro SiK V3 AUD 255,68 en compras de 5 o más (Core Electronics) **[F]**.
- **Pros:** muy probadas en telemetría; plug and play; FHSS y corrección de errores.
- **Contras:** costo alto frente a C9; las radios US saltan por 902–915 MHz (hay que limitar la frecuencia mínima/máxima, verificar el nombre del parámetro en el manual); la RFD900x consume mucho y exige 5 V; masa de 14–24 g (afecta S1).

#### G. Texas Instruments CC1101 (SPI, FSK)

- **Banda:** 300–348, 387–464 y 779–928 MHz **[F]**. Hasta +12 dBm; sensibilidad −112 dBm a 1,2 kbaud y 868 MHz; 14,7 mA en RX a 868 MHz; 0,6–600 kbps; FIFO de 64 bytes **[F]**. Módulo "RF‑CC1101‑915": RX 20 mA / TX 38 mA, −10 a +12 dBm **[F]**.
- **Pros:** consumo bajo, muy barato; filtro de dirección en hardware (verificar).
- **Contras:** baja sensibilidad a velocidades de varios kbps (el margen a 3 km es ≈ 17 dB, est.); FIFO de 64 bytes obliga a cuidar el tamaño del paquete; sin modulación LoRa.

### 7.2 Familia 2,4 GHz

#### H. Semtech SX1280 (LoRa 2,4 GHz)

- **Características:** sensibilidad hasta −132 dBm (a 0,476 kbps), +12,5 dBm a 24 mA en módulos comerciales, LoRa 0,476–202 kbps, FLRC hasta 1,3 Mbps, medición de distancia por tiempo de vuelo **[F]**. Módulos: LoRa1280/1281 (NiceRF), LAMBDA80 (RF Solutions, ≈ 192 SEK sin IVA) **[F]**.
- **Contras:** potencia baja; a 3 km el margen estimado es ≈ 18 dB con BW 406 kHz, SF9 (est.); poca experiencia en CanSat.

#### I. Digi XBee3 (802.15.4 / DigiMesh / Zigbee)

- **XBee3 estándar:** 8 dBm; **XBee3 PRO:** hasta 19 dBm; sensibilidad −103 dBm a 250 kbps; TX 135 mA, RX 15–17 mA; 2,1–3,6 V **[F]**.
- **Precio:** USD 19–38 (DigiKey) **[F]**; plazos de 12 a 17 semanas **[F]**.
- **Contras:** margen a 3 km de solo 6 dB con PRO y −5 dB con el estándar (est.).

#### J. Nordic nRF24L01+ (con y sin PA/LNA)

- **Sin PA:** 0 dBm, −94 dBm a 250 kbps, 11,3 mA TX, 13,5 mA RX **[F]**.
- **Con PA/LNA:** +20 dBm; sensibilidad −104 dBm a 250 kbps; TX 115 mA, RX 45 mA; 5 g **[F]**. Alcance "≈ 1000 m en campo abierto" (vendedor) **[F]**.
- **Canales:** 125 canales de 2400 a 2525 MHz **[F]**. **Solo valen los canales 0 a 82** (hasta 2482 MHz, dentro de 2400–2483,5 MHz); con 2 Mbps usar canales separados por 2 MHz (inferido).
- **Ventaja única:** ACK con carga útil de hasta 32 bytes para devolver comandos sin conmutar de modo **[F]**. Payload máximo 32 bytes por paquete: el paquete de 49–50 bytes se parte en dos.
- **Contras:** margen negativo sin PA (−22 dB a 3 km, est.) y de 8 dB con PA/LNA (est.).

#### K. ESP32 con ESP‑NOW

- Payload máximo de 250 bytes (v1.0) o 1470 bytes (v2.0) **[F]**; dirección por MAC (unicast), cifrado AES‑128 opcional para hasta 6 pares **[F]**.
- Alcance reportado: ≈ 200–250 m en línea de vista con la antena de placa; 500–650 m en pruebas con modo largo alcance; "hasta 1 km" según el fabricante **[F]**. Consumo: ≈ 100 mA en RX y 400 mA en TX (ESP32) **[F]**.
- **Contras:** consumo alto (≈ 250 mAh en 2,5 h, est.); alcance probado muy por debajo de 2–3 km.

### 7.3 Descartadas

- **HC‑12, APC220, E32‑433, E34‑433 y similares (433 MHz), y módulos 868 MHz:** fuera de X1.
- **Bluetooth/BLE y Wi‑Fi (infraestructura):** alcance insuficiente y asociación previa; no cumplen los 2–3 km.
- **LoRaWAN:** necesita puerta de enlace y está pensado para enlaces ascendentes poco frecuentes. No sirve para 1 Hz.

---

## 8. Tablas comparativas

### 8.1 Radio y banda

| Alternativa | Banda usada | Potencia máx. | Sensibilidad (dBm) | Velocidad en el aire | Modulación |
|---|---|---|---|---|---|
| A. RFM95W (SX1276) | 915 MHz, canal elegible | +20 dBm | −148 (mín.); −132 a BW250/SF9 (est.) | LoRa hasta 37,5 kbps; FSK hasta 300 kbps | LoRa / FSK |
| B. E32‑915T20D | 900–931 MHz, 32 canales | +20 dBm | −144 a −147 a 0,3 kbps; peor a 2,4 kbps | 0,3–19,2 kbps | LoRa |
| B'. E32‑915T30D | 900–931 MHz, 32 canales | +30 dBm | −145 a −148 a 0,3 kbps | 0,3–19,2 kbps | LoRa |
| C. E22‑900T30D (SX1262) | 850–930 MHz | +30 dBm | −134 a 2,4 kbps | 2,4–62,5 kbps | LoRa |
| D. RYLR998 | 820–960 MHz | +22 dBm | −129 | según `AT+PARAMETER` | LoRa |
| E. XBee‑PRO 900HP (AU) | 915–927 MHz, FHSS | +24 dBm | −110 (10 kbps); −101 (200 kbps) | 10 o 200 kbps | FSK/FHSS |
| F. RFD900x | 902–928 (limitar a 915–928) | +30 dBm | −111 (12 kbps); −105 (64 kbps); −98 (200 kbps) | 12–750 kbps | FSK/FHSS |
| G. SiK V3 100 mW | 902–928 (hay versión 915–927) | +20 dBm | −121 (2 kbps); −105 (64 kbps) | hasta 250 kbps | FSK/FHSS |
| H. CC1101 (módulo) | 915 MHz | +12 dBm | −112 (1,2 kbps) | 0,6–600 kbps | FSK/GFSK |
| I. SX1280 (módulo) | 2400–2483,5 MHz | +12,5 dBm | −132 (0,476 kbps) | 0,476–202 kbps (LoRa) | LoRa / FLRC |
| J. XBee3 / XBee3 PRO | 2,4 GHz | +8 / +19 dBm | −103 | 250 kbps | DSSS 802.15.4 |
| K. nRF24L01+ (PA/LNA) | 2400–2482 MHz | 0 / +20 dBm | −94 / −104 (250 kbps) | 0,25–2 Mbps | GFSK |
| L. ESP32 ESP‑NOW | 2,4 GHz, canal Wi‑Fi | ≈ +20 dBm | no verificado (≈ −94, est.) | 1–150 Mbps / LR | 802.11 |

### 8.2 Funciones para 1A y 1B

| Alternativa | Dúplex | Cómo recibe comandos mientras transmite | Direccionamiento punto a punto (X2) | Cambio de canal sin reprogramar | Paquete máx. |
|---|---|---|---|---|---|
| A. RFM95W | Half | Firmware: TX 164 ms y luego RX | Por software (byte de destino + sync word) | Sí, por firmware | 255 B (LoRa) |
| B. E32 | Half, automático | Queda en RX cuando no transmite | **Modo fijo** (dirección + canal); no usar 0x0000/0xFFFF | Sí, comando C0/C2 por UART | 58 B por subpaquete, buffer 512 B |
| C. E22 | Half, automático | Idem | Transmisión dirigida; LBT | Sí, por comando | según manual (verificar) |
| D. RYLR998 | Half | Idem | `AT+ADDRESS` + `AT+NETWORKID` | Sí, por AT | verificar (≈ 240 B) |
| E. XBee‑PRO 900HP | Half (el módulo gestiona) | Misma API para ambos sentidos | **Unicast 64 bits con ACK** | Sí, por AT/XCTU | 255 B |
| F. RFD900x / SiK | TDM (parece full duplex) | Transparente | NETID (verificar) | Sí, por AT | serie transparente |
| G. CC1101 | Half | Firmware | Filtro de dirección en hardware (verificar) | Sí, por firmware | 64 B de FIFO |
| H. SX1280 | Half | Firmware | Por software | Sí | 255 B |
| I. XBee3 | Half | API | Unicast 64 bits | Sí | ≈ 80–256 B según protocolo |
| J. nRF24L01+ | Half | **ACK con carga útil (32 B)** | Direcciones de 5 bytes por tubería | Sí, por firmware | 32 B |
| K. ESP‑NOW | Half | Ambos lados envían | Unicast por MAC | Sí, por firmware | 250 B / 1470 B |

### 8.3 Eléctrico, mecánico y económico

| Alternativa | Tensión | Corriente RX | Corriente TX | Reposo | Masa | Tamaño (mm) | Precio unitario | En pesos (est.) | Disponibilidad |
|---|---|---|---|---|---|---|---|---|---|
| A. RFM95W‑915S2 | 1,8–3,7 V | 10,3–12,1 mA | 20–120 mA | 1 µA | ≈ 3 g | 16 × 16 × 2 | USD 6,5–11 | $10–17 mil | Muchos revendedores; sin confirmar local |
| B. E32‑915T20D | 2,3–5,2 V | 13–15 mA | 110–130 mA | 3–5 µA | 6,7 g | 21 × 36 | ≈ USD 10 | ≈ $15 mil | Internacional (RobotShop y similares) |
| B'. E32‑915T30D | 3,3–5,2 V | 20–22 mA | 650–770 mA | 4–6 µA | 8,2 g | 24 × 43 | USD 10,57 | ≈ $16 mil | Idem |
| C. E22‑900T30D | 2,6–5,5 V | no verificado | no verificado | 3 µA | no verificado | no verificado | € 12,29 | ≈ $20 mil | Rotorama |
| D. RYLR998 | 1,8–3,6 V | 17,5 mA | 140 mA (22 dBm) | 15 µA | 2 g | 32 × 14 × 2,2 | USD 11,99 | ≈ $18,5 mil | Lemos y otros |
| E. XBee‑PRO 900HP | 2,1–3,6 V | 29 mA | 215 mA | 2,5 µA | no verificado | 20 pines | ≈ USD 61 | ≈ $94 mil | Variante AU difícil; plazo de 17–25 semanas en kits |
| F. RFD900x | 3,5–5,5 V (5 V) | ≈ 60 mA | hasta 1 A a 30 dBm | no verificado | 14,5 g | no verificado | USD 199 | ≈ $306 mil | IRLock, RFDesign |
| G. SiK V3 100 mW | 5 V | 25 mA | 100 mA | no verificado | 14,3–23,5 g | 55 × 26 × 13 | AUD 255,68 (5+) | no calculado | Core Electronics, Holybro |
| H. CC1101 (módulo) | 1,8–3,6 V | 20 mA | 38 mA | 1 µA | ≈ 1–2 g | pequeño | USD 2–5 (no verificado) | $3–8 mil | Muy común |
| I. SX1280 (módulo) | 1,8–3,7 V | no verificado | 24 mA (12,5 dBm) | no verificado | no verificado | 20 × 27 | ≈ 192 SEK sin IVA | ≈ $27 mil | RS Components; stock variable |
| J. XBee3 PRO | 2,1–3,6 V | 15–17 mA | 135 mA | no verificado | no verificado | no verificado | USD 19–38 | $29–59 mil | Plazos de 12–17 semanas |
| K. nRF24L01+ PA/LNA | 1,9–3,6 V | 45 mA | 115 mA | 4,2 µA | 5 g | 29 × 15 × 12 | USD 2–4 | $3–6 mil | Muy común |
| L. ESP32 | 3,3 V | ≈ 100 mA | ≈ 400 mA pico | µA (modos de sueño) | 2–10 g | variable | USD 3–8 | $5–12 mil | Muy común |

---

## 9. Tiempo en el aire de un paquete de 49 bytes

Calculado con la fórmula de Semtech AN1200.13 (encabezado explícito, CRC activado, CR 4/5, preámbulo de 8 símbolos). Verifiqué mi implementación contra ejemplos publicados: 2793,5 ms para 64 bytes a SF12/BW125 y 354,3 ms para 16 bytes a SF11/BW250 con preámbulo de 16, y coincidieron con las referencias **[F]**.

### 9.1 LoRa SX127x / SX126x (est.)

| BW | SF7 | SF8 | SF9 | SF10 | SF11 | SF12 |
|---|---:|---:|---:|---:|---:|---:|
| 125 kHz | 98 ms | 175 ms | 329 ms | 575 ms | 1233 ms | 2302 ms |
| 250 kHz | 49 ms | 87 ms | **164 ms** | 288 ms | 535 ms | 1151 ms |
| 500 kHz | 24 ms | 44 ms | 82 ms | 144 ms | 267 ms | 535 ms |

Sensibilidad aproximada (est., SX1276; baja 3 dB por cada duplicación del ancho de banda): BW125/SF9 −129 dBm; BW250/SF9 −132; BW250/SF8 −129; BW500/SF9 −135.

**Criterio:** telemetría ≤ 200–250 ms para dejar ≥ 500 ms de ventana de escucha. Entran BW250/SF8 a SF10, BW500/SF7 a SF10 y BW125/SF7 a SF8. Elegiría **BW250 SF9 (164 ms)** como punto de partida y **BW125 SF9 (329 ms)** como respaldo de más alcance. BW125/SF10 y superiores no entran con holgura.

Un comando de 12 bytes tarda 36 ms (BW500/SF9), 72 ms (BW250/SF9) o 144 ms (BW125/SF9) (est.).

### 9.2 Otras familias (est.)

| Alternativa | Velocidad | Tiempo en el aire de 49 B (+ encabezado) |
|---|---|---|
| XBee‑PRO 900HP 200 kbps | 200 kbps | ≈ 3 ms (con 20 B de encabezado supuestos) |
| XBee‑PRO 900HP 10 kbps | 10 kbps | ≈ 55 ms |
| RFD900x / SiK 64 kbps | 64 kbps | ≈ 7 ms; a 12 kbps ≈ 39 ms |
| E32 a 2,4 kbps | 2,4 kbps | ≈ 230 ms (49 B más preámbulo y FEC; medir con la señal AUX) |
| E32 a 4,8 kbps | 4,8 kbps | ≈ 115 ms |
| CC1101 a 1,2 kbps | 1,2 kbps | ≈ 390 ms (justo) |
| SX1280 LoRa BW406/SF9 | — | ≈ 101 ms |
| XBee3 802.15.4 | 250 kbps | ≈ 2–3 ms |
| nRF24L01+ | 250 kbps | ≈ 1–3 ms por paquete de 32 B; dos paquetes |

Los valores del E32 son una estimación gruesa; el fabricante no publica el tiempo en el aire. Hay que medirlo con el pin AUX en un osciloscopio o analizador lógico.

---

## 10. Presupuesto de enlace

**Fórmula (est.):** margen = P_tx + G_CanSat + G_estación − pérdida en espacio libre − pérdidas extra − sensibilidad.
Con G_CanSat = −3 dBi, G_estación = +3 dBi (omni), pérdidas extra = 6 dB. Pérdida en espacio libre: 915 MHz → 91,7 / 97,7 / 101,2 dB a 1 / 2 / 3 km; 2440 MHz → 100,2 / 106,2 / 109,7 dB.

### 10.1 Margen en el enlace de bajada (CanSat → estación)

| Configuración | P_tx (dBm) | Sens. (dBm) | 1 km | 2 km | 3 km | 3 km con Yagi de 9 dBi |
|---|---:|---:|---:|---:|---:|---:|
| RFM95W, BW250/SF9 | 20 | −132 | 54 | 48 | **45** | 51 |
| RFM95W, BW125/SF9 | 20 | −129 | 51 | 45 | 42 | 48 |
| E32‑915T20D, 2,4 kbps (sens. est. −129) | 20 | −129 | 51 | 45 | 42 | 48 |
| E32‑915T30D, 2,4 kbps (sens. est. −129) | 30 | −129 | 61 | 55 | 52 | 58 |
| RYLR998 (22 dBm, sens. de ficha) | 22 | −129 | 53 | 47 | 44 | 50 |
| E22‑900T30D (30 dBm) | 30 | −134 | 66 | 60 | 57 | 63 |
| XBee‑PRO 900HP 10 kbps | 24 | −110 | 36 | 30 | **27** | 33 |
| XBee‑PRO 900HP 200 kbps | 24 | −101 | 27 | 21 | 18 | 24 |
| RFD900x a 27 dBm, 64 kbps | 27 | −105 | 34 | 28 | 25 | 31 |
| SiK V3 100 mW, 64 kbps | 20 | −105 | 27 | 21 | 18 | 24 |
| CC1101 +12 dBm, 1,2 kbps | 12 | −112 | 26 | 20 | 17 | 23 |
| SX1280 LoRa +12,5 dBm, BW406/SF9 | 12,5 | −121,5 (est.) | 28 | 22 | 18 | 24 |
| XBee3 PRO 19 dBm | 19 | −103 | 16 | 10 | **6** | 12 |
| XBee3 estándar 8 dBm | 8 | −103 | 5 | −1 | **−5** | 1 |
| nRF24L01+ PA/LNA 20 dBm, 250 kbps | 20 | −104 | 18 | 12 | **8** | 14 |
| ESP‑NOW 20 dBm (sens. ≈ −94, est.) | 20 | −94 | 8 | 2 | **−2** | 4 |
| nRF24L01+ sin PA, 250 kbps | 0 | −94 | −12 | −18 | **−22** | −16 |

Lectura:

- Las configuraciones LoRa tienen **≥ 40 dB de margen a 3 km**: sobran 25–30 dB para desvanecimiento, rotación del CanSat y mala orientación de antena.
- XBee‑PRO 900HP y SiK/RFD900x quedan en 18–27 dB: aceptable.
- En 2,4 GHz solo SX1280 llega a ≈ 18 dB. Los demás no cumplen el criterio de ≥ 10 dB.
- **Aterrizaje:** con el CanSat y la estación a nivel del suelo, el modelo de espacio libre es optimista (reflexión en el suelo, obstáculos). Es esperable perder los últimos paquetes a pocos metros de altura. Los datos con altitud y velocidad de descenso (C3, C4) se verifican con los paquetes anteriores.
- **Enlace de subida (comandos):** la estación puede transmitir con más potencia que el CanSat (módulos de 30 dBm), pero con antena de > 6 dBi hay que bajar la potencia (sección 4). A 1–2 km desde la plataforma el margen es amplio en todas las alternativas sub‑GHz.

---

## 11. Energía (radio sola, 2,5 h: 2 h de E7 más 0,5 h de vuelo, 1 paquete por segundo) (est.)

Corriente promedio = I_RX · (1 − t_aire) + I_TX · t_aire. No incluye el microcontrolador, los sensores ni el GPS.

| Alternativa | Tensión | I promedio | Energía en 2,5 h | Comentario |
|---|---:|---:|---:|---|
| nRF24L01+ 0 dBm | 3,3 V | 13,5 mA | 34 mAh | Sin alcance |
| SX1280 12,5 dBm | 3,3 V | ≈ 11 mA (RX supuesto 10 mA) | ≈ 29 mAh | RX estimado, verificar |
| XBee3 PRO | 3,3 V | 17 mA | 43 mAh | |
| SiK V3 100 mW | 5 V | 25,5 mA | 64 mAh | |
| CC1101 | 3,3 V | 27 mA | 68 mAh | |
| **RFM95W (BW250/SF9, 20 dBm)** | 3,3 V | **29,7 mA** | **74 mAh** | RX continuo 12 mA + 164 ms de TX por segundo |
| XBee‑PRO 900HP 200 kbps | 3,3 V | 29,6 mA | 74 mAh | |
| E32‑915T20D | 3,3 V | 38 mA | 96 mAh | |
| XBee‑PRO 900HP 10 kbps | 3,3 V | 39 mA | 98 mAh | |
| nRF24L01+ PA/LNA | 3,3 V | 45 mA | 113 mAh | |
| RYLR998 22 dBm | 3,3 V | 58 mA | 145 mAh | |
| RFD900x 27 dBm | 5 V | 63 mA (TX supuesto 500 mA) | 158 mAh | TX estimado; pico de 1 A a 30 dBm |
| E32‑915T30D | 5 V | 177 mA | 443 mAh | Es el consumo más alto de las UART |
| ESP32 ESP‑NOW | 3,3 V | ≈ 100 mA | ≈ 250 mAh | |

Una celda 18650 típica tiene 2.000–3.000 mAh (dato general, verificar la elegida). Para el balance eléctrico: pasales a aviónica el consumo del radio en **RX continuo durante 2 h**, que es lo que domina. Si la radio puede entrar en recepción de bajo consumo en plataforma (ventanas de escucha en vez de RX continuo), el consumo baja, pero hay que cuidar que los comandos G1 y F3 sigan llegando.

---

## 12. Matriz de decisión (subjetiva)

Puntajes de 1 a 5 según lo expuesto en este documento. Los pesos y los puntajes son un juicio mío: ajustalos con el equipo.

| Criterio | Peso |
|---|---:|
| Margen de enlace a 3 km | 20 |
| Costo y disponibilidad en Argentina | 20 |
| Facilidad de integración y de desarrollo | 15 |
| Consumo y masa | 10 |
| Cumplir X2 con poco trabajo | 10 |
| Recepción de comandos mientras transmite | 10 |
| Convivencia e inmunidad a interferencia en el campo | 10 |
| Evidencia de uso en telemetría de este tipo | 5 |

| Alternativa | Margen | Costo/disp. | Integración | Consumo/masa | X2 | Comandos | Convivencia | Evidencia | **Total /100** |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| **A. RFM95W (LoRa SPI)** | 5 | 5 | 3 | 5 | 3 | 3 | 4 | 4 | **83** |
| D. RYLR998 | 4 | 4 | 5 | 5 | 4 | 4 | 3 | 2 | 81 |
| B. E32‑915T20D | 4 | 4 | 5 | 4 | 4 | 4 | 3 | 3 | 80 |
| C. E22‑900T30D | 5 | 4 | 4 | 3 | 4 | 4 | 4 | 2 | 80 |
| E. XBee‑PRO 900HP (AU) | 3 | 2 | 5 | 4 | 5 | 4 | 5 | 5 | 76 |
| I. XBee3 PRO (2,4 GHz) | 2 | 3 | 5 | 5 | 5 | 4 | 4 | 3 | 74 |
| G. SiK V3 100 mW | 3 | 2 | 5 | 3 | 4 | 4 | 5 | 4 | 71 |
| F. RFD900x | 3 | 1 | 5 | 3 | 4 | 4 | 5 | 5 | 68 |
| G'. CC1101 | 3 | 4 | 2 | 5 | 4 | 3 | 3 | 2 | 66 |
| J. nRF24L01+ PA/LNA | 2 | 5 | 2 | 4 | 4 | 5 | 2 | 2 | 66 |
| H. SX1280 | 3 | 3 | 2 | 5 | 3 | 3 | 3 | 1 | 59 |
| K. ESP‑NOW | 1 | 5 | 4 | 2 | 4 | 3 | 1 | 2 | 58 |

Los cuatro primeros están dentro de 3 puntos entre sí: **la diferencia no es significativa**. La matriz sirve para descartar (por debajo de 70 quedan las 2,4 GHz y las caras) y para ordenar prioridades, no para elegir entre A, B, C y D. Esa elección depende de cuánto firmware quieren escribir y de qué módulo consigan en Argentina a tiempo.

---

## 13. Diseño propuesto para el PDR (opción principal con plan B)

**Plan A (control total):** RFM95W‑915 o módulo SX1262 por SPI.

- Frecuencia: un canal fijo entre 916 y 927 MHz asignado por CONAE; cambio de canal por una constante o comando.
- Modulación: LoRa BW250 / SF9 / CR 4/5, preámbulo de 8 símbolos, encabezado explícito, CRC activado (164 ms para 49 B).
- Potencia: empezar en +17 dBm y subir si el ensayo de alcance lo pide (máx. +20 dBm).
- Paquete: sección 5, con byte de destino, byte de estado y CRC‑16 propio (G14).
- Ciclo de 1 s: TX telemetría → RX 300 ms (comandos) → reposo/RX.
- Antena: monopolo de cuarto de onda (≈ 8 cm a 915 MHz) con plano de masa; probar dentro del contenedor.

**Plan B (integración rápida):** E32‑915T20D, E22‑900T30D o RYLR998.

- E32 y E22: modo de transmisión fija con dirección propia distinta de 0x0000/0xFFFF; canal en 916–927 MHz; velocidad en el aire de 4,8 kbps o más para que el paquete tarde menos de ≈ 250 ms (medir).
- Alimentación: preferir la versión de 20 dBm (≤ 130 mA de pico) a 3,3 V.

**Plan C (probado, si el dinero y la disponibilidad lo permiten):** XBee‑PRO 900HP variante AU o par SiK, con unicast.

**Estación terrena:** el mismo módulo con antena omnidireccional o direccional de mesa (G11); alimentado por USB de la notebook (≤ 500 mA: sirve para los módulos de 20 dBm; los de 30 dBm necesitan fuente externa por la corriente de pico).

---

## 14. Riesgos y qué probar

| Riesgo | Cómo se mitiga / prueba |
|---|---|
| El cohete o el contenedor atenúa la señal en plataforma | Ensayo con el CanSat dentro de un tubo similar (material y diámetro del fuselaje de CONAE, dato pendiente) y medir RSSI a 100 m y 500 m |
| Rotación del CanSat y orientación de antena | Probar con el CanSat girando; antena vertical, ganancia baja |
| Radio con salto de frecuencia fuera de 915–928 MHz | Usar variante AU o limitar el rango; verificarlo con un analizador de espectro o un SDR |
| Modo broadcast activo por defecto | Revisar dirección (E32: ≠ 0x0000/0xFFFF; XBee: unicast) y documentarlo en el PDR |
| Pérdida de paquetes al aterrizar | Registrar RSSI; asumir pérdida en los últimos metros |
| Colisión con otros equipos del campo | Canal asignado por CONAE; considerar LBT (E22) o salto de frecuencia (XBee/SiK) |
| Picos de corriente que reinician el microcontrolador | Capacitores de desacople y regulador con capacidad de pico (sobre todo E32‑T30D y RFD900x) |
| Plazos de entrega | Ordenar ya: los módulos de revendedores internacionales tardan semanas; XBee y RFD900x pueden tardar meses |
| Vibración y golpe (15 G / 30 G) | Soldar el módulo, evitar conectores de pines sin fijación, fijar el cable de la antena |
| Mensaje de inicio de misión perdido (X3) | Repetirlo en los 3–5 paquetes siguientes con la bandera activa |

**Ensayos mínimos recomendados antes del PDR:** (1) medir el tiempo en el aire real del paquete con un analizador lógico; (2) ensayo de alcance a 1, 2 y 3 km con la antena dentro del contenedor, en terreno abierto; (3) ensayo con el módulo dentro de un tubo del mismo diámetro que el fuselaje; (4) medir el consumo con la batería elegida durante 2 h.

---

## 15. Pendientes y consultas

Pendientes de este documento:

1. Precios y stock en Argentina (MercadoLibre, comercios locales, cotizaciones de importadores). No los pude confirmar.
2. Manuales completos de E22‑900T30D, E220 y RYLR998 (corriente exacta, tamaño de paquete, canales válidos en 915–928 MHz).
3. Nombre exacto de los parámetros de frecuencia mínima y máxima en SiK/RFD900x.
4. Si el módulo SX127x en LoRa filtra direcciones por hardware (creo que no) y si el CC1101 filtra direcciones (creo que sí).
5. Valor real de la sensibilidad de ESP32 en 802.11b y en modo de largo alcance.
6. Si las variantes AU del XBee‑PRO 900HP siguen en producción y a qué plazo.

Para **CONAE** (además de las de `investigacion-sota.md`):

- ¿Se exige homologación ENACOM (ENACOM‑Q2‑60.14) a las radios del concurso, o basta con trabajar en las bandas del Anexo?
- ¿Qué frecuencia o canal asignan a cada equipo, y cómo se coordinan los equipos que vuelan en simultáneo?
- ¿Hay límite de potencia propio del concurso, además del de ENACOM (30 dBm conducidos, 36 dBm de PIRE)?
- Dónde se ubican las estaciones terrenas respecto de la plataforma y de la línea de vuelo (afecta la distancia de diseño).

---

## 16. Fuentes

**Normativa**

- Resolución 581/2018 (Ministerio de Modernización), bandas de uso compartido y Anexo con potencias: https://argentina.gob.ar/normativa/314174_res581_pdf/archivo y https://argentina.gob.ar/normativa/306992_res2_pdf/archivo
- Resolución 1132/2023 (ENACOM), Boletín Oficial: https://www.boletinoficial.gob.ar/detalleAviso/primera/292414/20230818
- Resolución 551/2021 (ENACOM‑Q2‑60.14 V20.1): https://ib-lenhardt.com/news/enacom-revised-the-regulation-4-653-2019 y https://www.eleoscompliance.com/en/article/阿根廷-低功率设备修订要求
- Resolución SC 67/2012 y novedades de ENACOM: https://ib-lenhardt.com/de/news/additional-frequency-bands-for-low-power-radio-equipment
- Cotización del dólar oficial (20 de octubre de 2026): https://eleconomista.com.ar/economia/dolar-hoy-cuanto-cotiza-martes-20-octubre-n99012

**LoRa SX127x y módulos**

- RFM95W (HopeRF): https://html.alldatasheet.com/html-pdf/750169/ETC2/RFM95W-915S2/101/1/RFM95W-915S2.html, https://digikey.cn/en/products/detail/rf-solutions/RFM95W-915S2R/15792834, https://st.robu.in/?p=1031106
- Adafruit RFM95W y RFM95CW: https://www.adafruit.com/products/5684 y https://littlebirdelectronics.com.au/products/adafruit-rfm95w-lora-radio-transceiver-breakout-868-or-915-mhz
- uelectronics (RFM95W 915 MHz): https://uelectronics.com/producto/rfm95w-915mhz-transceptor-lora/
- Semtech SX1262: https://www.semtech.com/products/wireless-rf/lora-connect/sx1262 y https://www.digikey.com/en/products/detail/semtech-corporation/SX1262IMLTRT/8564369
- Fórmula de tiempo en el aire (Semtech AN1200.13): https://d-central.tech/lora-airtime-calculator/ y https://github.com/tanupoo/lorawan_toa
- AU915 (plan de canales): https://www.rfessentials.com/resources/rf-glossary/au915/

**Módulos UART**

- EBYTE E32, manual de usuario: https://pdf.direnc.net/upload/e32-usermanual-en-v1.pdf
- EBYTE E32‑915T20D (CNX Software): https://www.cnx-software.com/?p=56426
- E22‑900T30D (Rotorama): https://www.rotorama.com/product/ebyte-e22-900t30s-2
- RobotShop (E32‑900T30D, E32‑900T30S, E220‑900T30D): https://www.robotshop.com/products/ebyte-e32-900t30d-semtechs-lora-solution-wireless-serial-port-transmitter-lora-modulation-30dbm-868-915mhz-1w-cdebyte-antenna-dip-8km
- REYAX RYLR998: https://cdn.hackaday.io/files/1853287929073152/RYLR998_EN.pdf y https://lemosint.com/product/rylr998

**XBee, SiK, RFD900**

- XBee‑PRO 900HP (datasheet): https://lv.mouser.com/pdfdocs/Digi_International_XbeePro_900HP_Datasheet.pdf
- Variantes y números de parte: https://static.chipdip.ru/lib/508/DOC030508946.pdf y https://forums.digi.com/t/xbee-pro-900hp-model-002-america-change-af-or-cm-field-in-xctu/12609
- Guía de usuario XBee‑PRO 900HP: https://digi.com/resources/documentation/digidocs/PDFs/90002173.pdf
- Libelium, red 900HP: https://development.libelium.com/900-hp-networking-guide/networking-methods
- RFD900x: https://cdn.hackaday.io/files/276031180966752/RFD900x DataSheet.pdf, https://irlock.com/products/rfd900x-us-telemetry-modem-fcc-approved, https://irlock.com/products/rfd-900ux
- SiK 3DR y Holybro: https://docs.3dr.com/telemetry/sik-telemetry-radio-v2, https://holybro.com/products/sik-telemetry-radio-1w, https://core-electronics.com.au/sik-telemetry-radio-v3-915mhz-100mw.html
- XBee3: https://www.digikey.es/en/products/detail/digi/XB3-24Z8RS/8130941

**Otras familias**

- CC1101: https://www.ti.com/product/CC1101 y https://www.directindustry.com/prod/radiocontrolli-srl/product-126049-2810962.html
- SX1280: https://www.mouser.com/datasheet/2/761/DS_SX1280-1_V2.2-1511144.pdf y https://uk.rs-online.com/web/p/lpwan/1812482/
- nRF24L01+: https://www.nicerf.com/item/nrf24l01, https://shop.boxtec.ch/nrf24l01-module-enhanced-edition-p-42032.html, https://rf24.readthedocs.io/en/latest/examples/Python/AcknowledgementPayloads.html
- ESP‑NOW: https://docs.espressif.com/projects/esp-idf/en/latest/api-reference/network/esp_now.html, https://esp32.com/viewtopic.php?p=127608, https://blog.atomic14.com/videos/posts/oz0a7Ur7nko

**Comparaciones y experiencias en CanSat / telemetría**

- LoRa vs. WiFi en CanSat (SBrT 2025): https://biblioteca.sbrt.org.br/articles/4984
- LoRa vs. nRF en CanSat: https://civilica.com/doc/2637222/
- LoRa vs. XBee y ZigBee: https://community.particle.io/t/lora-vs-xbee-vs-xenon/40771, https://www.fed4fire.eu/wp-content/uploads/sites/10/2022/03/f4fp-sme-stage-2-09m09-fec10-presentation-pawl-pangaesa.pdf, https://portaldeperiodicos.marinha.mil.br/index.php/ocasopeano/article/view/7138
