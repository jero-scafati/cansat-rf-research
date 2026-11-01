# SOTA — Puntos 2 y 3: antenas del enlace CanSat ⇄ estación terrena

**Fecha de la investigación:** 26/10/2026
**Alcance:** punto 2 (antena del CanSat) y punto 3 (antena de la estación terrena) del documento `investigacion-sota.md`, con requisitos del Anexo A (Apéndice A – Requisitos de Misión).
**Contexto:** CanSat Argentina Universitario 2026, se compra y opera en Argentina.

> **Cómo leer este documento.** Lo que viene de una ficha técnica, una norma o un listado de venta está marcado con su fuente (sección 9). Lo que es **cálculo o estimación mía** está marcado como tal. Los puntajes de las matrices de decisión son **juicio mío** y están para que ustedes los discutan y los cambien, no son datos.

---

## 0. Resumen ejecutivo

**Lo primero que hay que saber:** la antena depende de la banda que elija el equipo de Radio (punto 1). No está decidida, así que cubro las dos permitidas por X1: 915–928 MHz y 2400–2483,5 MHz. Mi opción por defecto es **915 MHz**, por tres razones:

1. A igual potencia y ganancia, la pérdida en el espacio libre es unos 8,5 dB menor que a 2,4 GHz (cálculo mío, sección 5.4).
2. La banda de 2,4 GHz está mucho más ocupada en el campo (Wi-Fi, Bluetooth, controles de drones, celulares de los equipos).
3. Con 915 MHz la antena de la estación puede ser omnidireccional y el enlace sigue cerrando con margen. Con 2,4 GHz, casi seguro hace falta una antena direccional apuntada a mano.

**Recomendaciones (resumidas):**

| Punto | Recomendación principal | Respaldo |
|---|---|---|
| **2. Antena del CanSat** | Monopolo de cuarto de onda (λ/4) vertical, a lo largo del eje, **dentro** del contenedor, de alambre rígido fijado mecánicamente (unos 77–79 mm a 915 MHz), sobre el plano de tierra de la placa y lejos de las baterías. Se ajusta con analizador de redes (NanoVNA). | "Rubber duck" λ/4 comercial de 2–3 dBi (SMA), con soporte mecánico, o látigo corto helicoidal. |
| **3. Antena de la estación** | **Omnidireccional vertical de 2–5 dBi** sobre mástil o trípode, sin apuntar. | **Yagi de 10–11 dBi** operada por un integrante, como segunda antena opcional (ojo con G9, ver 5.7). Si la banda es 2,4 GHz: panel de 12–14 dBi apuntado, más una omni de respaldo. |

**Cosas que conviene saber ya:**

- **Normativa (sección 2):** las bandas de 915–928 MHz y 2400–2483,5 MHz son de uso compartido sin autorización previa, con tope de **30 dBm conducidos y 36 dBm de PIRE**. Si la antena de transmisión tiene más de 6 dBi, hay que bajar la potencia en los dB que excedan. Con módulos de ~100 mW (20 dBm) y antenas de hasta 14 dBi, queda dentro del límite (cálculo mío).
- **Los dBi de los látigos baratos no son confiables.** Un látigo de 13–19 cm que dice "10 dBi" no puede tener esa ganancia a 915 MHz (explicación en 4.3). Contá con 2–3 dBi.
- **Costo:** las antenas del CanSat cuestan del orden de AR$ 5.000 a AR$ 70.000, entre el 0,5 % y el 7 % del tope de C9 (AR$ 1.000.000). La antena de la estación no cuenta para C9.
- **Mayor riesgo desconocido:** el material del fuselaje del cohete. Si es fibra de carbono o tiene partes metálicas, el enlace en la plataforma (G1) puede fallar aunque todo lo demás esté bien. Hay que preguntarle a CONAE (sección 8).
- **G9 dice "una antena"** en singular. Si quieren llevar omni + Yagi, conviene consultarlo (sección 5.7).

---

## 1. Alcance, supuestos y confiabilidad de los datos

### 1.1 Qué cubre

- **Punto 2:** irradiar y captar la señal desde dentro del cuerpo del CanSat.
- **Punto 3:** captar y emitir hacia el CanSat desde el sitio de operación.
- **No cubre:** elección de la radio (punto 1), interfaz con la notebook (punto 4), protocolo (punto 5) ni software (punto 6). Sí marco las interacciones (sección 6).

### 1.2 Supuestos de diseño

Los tomo del `investigacion-sota.md` y los marco como supuestos, **no son datos de CONAE**:

| Supuesto | Valor | Origen |
|---|---|---|
| Altitud máxima de vuelo | 500 m (Bases) a 1 km (reunión) | Bases §4 / reunión, a confirmar |
| Alcance a cubrir | 2–3 km en línea recta | Estimación del SOTA (1 km de altura, 10 m/s de viento) |
| Potencia de transmisión del módulo | 20 dBm (100 mW) | Típica de módulos tipo RFM95; **a confirmar con Radio** |
| Estación | Portátil, sin corriente alterna, a lo largo de la línea de vuelo | G10 |
| Orientación del CanSat en el descenso | No controlada, con balanceo y giro | X5 (mide rotaciones). En el reglamento original de CanSat (EE. UU.), el despliegue es violento y sin orientación controlada |

### 1.3 Confiabilidad de lo que encontré

- **Fichas técnicas y normas:** confiables en lo que dicen. Las normas argentinas las leí directamente del sitio oficial (sección 2). **No pude leer el texto completo de la Res. ENACOM 4.653/2019**, que es la que fija las condiciones técnicas de las bandas compartidas. Los valores de potencia que cito salen del anexo de la Res. 581/2018, que sí leí. Verifiquen que no haya modificaciones posteriores.
- **Precios argentinos:** vienen de listados de Mercado Libre Argentina, Desertcart Argentina y Frávega vistos a través de un buscador. **Son capturas de fechas distintas y algunas son viejas** (por ejemplo, un mismo tipo de antena aparece a $3.699 y a $59.000). Los uso como orden de magnitud. **Hay que reverificar precio y stock el día de comprar.** No pude abrir las páginas de producto de Mercado Libre directamente.
- **Ganancias de venta:** son las que declara el vendedor. Para la masa y el VSWR real, no hay sustituto de medir.

---

## 2. Lo que impone la normativa argentina a la antena

### 2.1 Bandas y potencias (verificado en fuente oficial)

La Resolución 581/2018 (ex Ministerio de Modernización) declara de uso compartido, sin necesidad de autorización previa, las bandas **915–928 MHz** y **2400–2483,5 MHz**, entre otras. El anexo fija:

| Banda | Potencia máx. del transmisor (conducida) | PIRE máx. |
|---|---|---|
| 915–928 MHz | 30 dBm | 36 dBm |
| 2400–2483,5 MHz | 30 dBm | 36 dBm |

**Regla de ganancia de antena:** si la antena de transmisión tiene más de 6 dBi de ganancia direccional, hay que bajar la potencia del transmisor en tantos dB como excedan los 6 dBi, para no pasar el PIRE.

**Excepción en 2,4 GHz:** en sistemas exclusivamente fijos punto a punto se permite más de 6 dBi, con una reducción de 1 dB de potencia por cada 3 dB de exceso. Una estación portátil que apunta a mano a un blanco móvil **no es** "fijo punto a punto", así que aplica la regla general.

**Cuentas mías con 20 dBm de potencia:**

| Antena de transmisión | Potencia conducida máxima permitida | PIRE con 20 dBm | ¿Cumple con 20 dBm? |
|---|---|---|---|
| 915 MHz, Yagi de 11 dBi | 30 − (11 − 6) = 25 dBm | 31 dBm | Sí |
| 915 MHz, Yagi de 15 dBi | 30 − 9 = 21 dBm | 35 dBm | Sí, pero casi en el límite |
| 2,4 GHz, panel de 14 dBi | 30 − 8 = 22 dBm | 34 dBm | Sí |

**Conclusión práctica:** con módulos de 100 mW no hay problema. Sí lo habría con amplificadores de potencia (más de 1 W). **No usen amplificadores sin consultar**, y además en la estación no hay corriente alterna (G10).

### 2.2 Otro régimen que puede aplicar: "dispositivos de baja potencia"

La Res. 551/2021 adopta la norma técnica **ENACOM-Q2-60.14 V20.1 "Dispositivos de Baja Potencia"** para equipos en 915–928 MHz, 2400–2483,5 MHz y 5725–5850 MHz. Esa norma:

- Limita el campo eléctrico a **50.000 µV/m medidos a 3 m** en 915–928 MHz y en 2400–2483,5 MHz.
- Esos 50.000 µV/m a 3 m equivalen aproximadamente a **0,75 mW de PIRE (−1,25 dBm)** (cálculo mío con la fórmula de campo lejano: P = (E·d)²/30). Es muchísimo menos que los 36 dBm del régimen de la Res. 581.
- Exige que los transmisores lleven **antena integrada o una antena externa específica**, y que el fabricante evite el uso de una antena distinta de la ensayada. Dos equipos con transmisor idéntico y antenas distintas se consideran materiales distintos.
- Pide que la solicitud de inscripción en el RAMATEL especifique tipo de antena, características y fotos.

**No pude determinar** cuál de los dos regímenes aplica a un módulo LoRa de venta común, ni si CONAE va a exigir homologación para un prototipo universitario. X1 solo dice "frecuencias autorizadas por ENACOM". Lo dejo como pregunta (sección 8). Por ahora, regla de prudencia: **usar el módulo con la antena que trae/recomienda el fabricante como punto de partida** y documentar cualquier cambio.

### 2.3 Detalles de banda que afectan a la antena

- **Argentina usa la parte alta de la banda ISM de 915 MHz.** En Argentina, el tramo 905–915 MHz se destinó a servicios móviles avanzados, por lo que los equipos LoRa operan en el plan **AU915** (915–928 MHz) y no en el US915 (902–928 MHz). Fuente: The Things Network.
- **Consecuencia para ustedes:** muchas placas LoRa de "915 MHz" vienen con canales por defecto en 902–915 MHz, **fuera** de lo permitido por X1. Verifiquen con Radio que los canales queden dentro de 915–928 MHz. Para la antena, esto significa que **conviene centrar el VSWR en ~921 MHz**, no en 868 ni en 902–915.
- Muchas antenas se venden como "868/915 MHz". Las de ese tipo suelen estar mejor ajustadas hacia 868 MHz. **Midan el VSWR en 915–928 MHz** antes de aceptarlas.

---

## 3. Fundamentos que se usan más abajo

### 3.1 Longitudes de onda y largos de antena (cálculos míos)

| Parámetro | 915–928 MHz | 2400–2483,5 MHz |
|---|---|---|
| Longitud de onda λ (en el centro de la banda) | ≈ 325 mm (a 921,5 MHz) | ≈ 123 mm (a 2440 MHz) |
| Monopolo λ/4 teórico | ≈ 81 mm | ≈ 31 mm |
| Monopolo λ/4 de alambre, valor práctico de partida | ≈ 77–79 mm (empezar en 80 mm y recortar midiendo) | ≈ 29–30 mm |
| Dipolo λ/2 teórico | ≈ 163 mm | ≈ 61 mm |

El factor de acortamiento real depende del diámetro del alambre, del plano de tierra y de lo que haya alrededor. Por eso se mide.

### 3.2 Un monopolo necesita plano de tierra

Un monopolo λ/4 irradia usando como "segunda mitad" el plano de tierra que tiene debajo. En un CanSat ese plano es la masa de la placa y, en parte, las baterías metálicas. Una ficha de Taoglas lo ilustra: su antena helicoidal de 10 mm declara 2 dBi de pico y 68 % de eficiencia **sobre una placa de 127 × 80 mm**, y aclara que necesita plano de tierra para irradiar bien. En un CanSat de Ø86 mm interior el plano disponible es de ese orden o menor. **Esperen 0–2 dBi reales**, no los 3 dBi de la etiqueta.

### 3.3 Patrón de radiación de una antena vertical

Un monopolo o dipolo vertical irradia como una "dona" alrededor del eje: máximo en el plano horizontal y **nulo en la dirección del eje**. Si el CanSat cuelga con el eje vertical, el nulo apunta al cenit y al nadir. Lo evaluamos con números en 4.2.

### 3.4 Polarización

Casi todas las antenas baratas son de **polarización lineal vertical**. Si la antena del CanSat se inclina un ángulo φ respecto de la de la estación, la pérdida por desalineación es 20·log(cos φ). **Cuentas mías:**

| Inclinación φ | Pérdida |
|---|---|
| 15° | 0,3 dB |
| 30° | 1,25 dB |
| 45° | 3,0 dB |
| 60° | 6,0 dB |
| 75° | 11,7 dB |
| 85° | 21 dB |
| 90° | nulo |

Con el CanSat colgando bajo el paracaídas y oscilando unos ±30°, la pérdida es de ~1 dB. El problema aparece si queda tumbándose antes de abrirse el paracaídas.

---

## 4. Punto 2 — Antena del CanSat

### 4.1 Restricciones

**De los documentos:**

| Restricción | Requisito | Efecto sobre la antena |
|---|---|---|
| Banda | X1 | 915–928 MHz o 2400–2483,5 MHz |
| Diámetro | S2 (Anexo): 90 mm. Bases §4: 95 mm | Interior útil ≈ 86 mm (90 mm menos dos paredes de 2 mm), según S6 si es impreso en 3D |
| Largo | S3: 200–250 mm **incluyendo sistemas de descenso** | La antena compite por espacio con paracaídas y el segundo sistema de descenso |
| Pared impresa en 3D | S6: ≥ 2 mm | El plástico (PLA, PETG, ABS) casi no atenúa a 915 MHz ni a 2,4 GHz |
| Deslizamiento | S7 | La antena no puede sobresalir del contenedor |
| Vibración / impacto | S4: 15 G; S5: 30 G | La antena y su conector deben soportarlo |
| Montaje | S8 | Todo montado fijo: nada de antenas sueltas o articuladas |
| Baterías | E1, E2, M4 | Celdas metálicas tipo 18650 o de botón, sin contactos de resorte |
| Masa total | S1: 500 ± 10 g | Cada gramo de antena y soporte cuenta |
| Costo | C9 | Antena y conectores del CanSat cuentan |
| Rotación | X5 | Hay que cuidar el patrón de radiación |

**Derivadas (mías):**

- **La antena va dentro del contenedor.** Una antena que sobresalga rompe S7 y no sobrevive al despliegue violento. Si alguien quiere sacarla afuera, hay que consultarlo (sección 8).
- **Evitar plásticos cargados.** Filamentos con fibra de carbono o relleno metálico cerca de la antena la desajustan o la bloquean. Usen PLA o PETG simples.
- **Si el CanSat se separa en dos cuerpos** (C4 habla de liberar otro sistema de descenso, no queda claro si se separa algo), la radio y la antena tienen que quedar en el cuerpo que sigue transmitiendo hasta el suelo y no depender de la geometría del cuerpo que se va.
- **El GPS puede degradarse cuando transmite la radio.** Es un problema conocido en la práctica. Mantengan antena de radio y antena GPS separadas, y prueben el fix de GPS con la radio transmitiendo.

### 4.2 Geometría de vuelo: qué pasa con el nulo del patrón

**Planteo (cálculo mío, idealizado):** el CanSat cuelga con el eje vertical y la antena a lo largo del eje. Para una estación a distancia horizontal *d* y un CanSat a altura *h*, el ángulo de elevación es e = atan(h/d) y el ángulo respecto del eje de la antena es ψ = 90° − e. El patrón de una antena corta vertical varía como sen ψ.

| d / h | Elevación e | ψ (respecto del eje) | Pérdida por patrón |
|---|---|---|---|
| 4 | 14° | 76° | 0,3 dB |
| 2 | 27° | 63° | 1,0 dB |
| 1 | 45° | 45° | 3,0 dB |
| 0,6 | 59° | 31° | 5,8 dB |
| 0,3 | 73° | 17° | 10,8 dB |
| 0,1 | 84° | 6° | 20 dB |

**Lectura:** con la estación a una distancia horizontal de al menos la mitad de la altura del CanSat (d/h ≥ 0,5), la pérdida por patrón es de ~5 dB o menos. **Si el CanSat pasa casi encima de la estación, cae en el nulo.** Mitigaciones:

1. Ubicar la estación **a cierta distancia de la zona de caída** (a centenares de metros), no debajo.
2. El balanceo del CanSat rellena parcialmente el nulo.
3. Un plano de tierra finito inclina el lóbulo hacia arriba, lo que ayuda un poco.

Esto es un modelo idealizado. **Se verifica con la prueba de patrón (sección 7).**

### 4.3 Alternativas

#### A1. "Rubber duck" λ/4 comercial (látigo flexible con SMA o RP-SMA)

- Es la antena de plug and play: se enrosca en el conector del módulo.
- Ejemplos con ficha: SparkFun WRL-14875 (2 dBi, 110 mm, RP-SMA); Seeed (2,6 dBi, 195 mm, SMA); ELSYS ANT915 (3 dBi, 210 mm, SMA articulada); uElectronics (3 dBi, 100 × 17 × 8 mm, 9 g, VSWR ≤ 1,5, SMA).
- **Pros:** VSWR conocido, trae conector, se compra en Argentina, barata.
- **Contras:** 100–110 mm de largo consumen más de la mitad de la longitud del CanSat; es una masa en voladizo sobre un conector SMA; las versiones articuladas violan el espíritu de S8.
- **Cuenta mía de carga:** 9 g a 30 G ejercen unos 2,6 N sobre la antena, con un momento de ~0,13 N·m sobre el conector si el centro de masa está a 5 cm. **Necesita apoyo mecánico** (abrazadera impresa o soporte), no confiar solo en el conector soldado a la placa.

#### A2. Monopolo λ/4 de alambre rígido (hecho por ustedes)

- Un alambre de cobre rígido (o varilla de latón) de unos 77–79 mm, soldado al módulo o a un conector, fijado con un soporte.
- **Pros:** el más liviano (<1 g), el más corto de los que irradian bien, sin cuerpo plástico que atenúe, barato, totalmente fijable (S8). Se ajusta recortando.
- **Contras:** requiere **medirlo** con un analizador (NanoVNA o laboratorio); la eficiencia depende del plano de tierra y de la cercanía a las baterías.

#### A3. Látigo corto helicoidal / "spring" / "stubby" (≈ 50–100 mm)

- Ejemplos en Argentina: "Antena Whip 50 mm 900 MHz" (ITYT) y "Antena LoRa Spring 915 MHz, 2 dBi, 10 cm" (Dimalls, compra internacional); la antena del kit de Starware de ~5 cm con pigtail de 12 cm.
- **Pros:** muy compacta, robusta a impacto.
- **Contras:** menor eficiencia y menor ancho de banda que un λ/4 completo; más sensible al entorno. La ganancia declarada rara vez se cumple.

#### A4. Dipolo λ/2 (≈ 156–163 mm)

- **Pros:** no depende tanto del plano de tierra; es más inmune a lo que haya cerca.
- **Contras:** el largo es incompatible con 200–250 mm de CanSat si se comparte espacio con los sistemas de descenso. Se puede doblar o plegar, pero se complica.

#### A5. Antena cerámica (chip) o SMD helicoidal

- Ejemplo: Taoglas HA.31.A (10 × 10 × 10,7 mm, 0,7 g, 2 dBi de pico, 68 % de eficiencia sobre una placa de 127 × 80 mm).
- **Pros:** diminuta, pesa casi nada, sobrevive a cualquier vibración.
- **Contras:** pide **plano de tierra grande y diseño de la placa con red de adaptación**; muy sensible a baterías y carcasa; Silicon Labs, en una nota de aplicación, resume que las antenas de placa/chip sacrifican ganancia a cambio de espacio. No se consigue en comercios argentinos (habría que importar).

#### A6. Antena impresa en la placa (meandro, ILA, IFA, F invertida)

- Silicon Labs (AN847) probó nueve tipos de antena para 915 MHz. Para la versión impresa alrededor del perímetro de la placa reporta una **ganancia máxima de −1 dBi**, y advierte que es sensible al efecto de la mano (hay que ajustarla en el banco).
- **Pros:** costo cero de antena, ningún conector.
- **Contras:** −1 dBi o peor, ajuste fino, depende de la placa final, y hay baterías metálicas cerca. **No recomendada** para el CanSat.

#### A7. Parche (patch)

- Direccional, con máximo de radiación perpendicular a su cara.
- **Contras:** con el CanSat oscilando y orientación incierta, el lóbulo puede apuntar a cualquier lado; a 2 km de distancia y 15–27° de elevación la estación queda lejos del eje de un parche mirando al nadir. A 915 MHz un parche sobre FR4 mide unos 8 cm de lado. **No recomendada.**

#### A8. Antena desplegable (alambre o cinta que se abre tras la expulsión)

- Permite una antena más larga que el contenedor. S7 admite piezas desplegables de la carga útil.
- **Contras:** agrega un mecanismo (M3: debe mantener su estado bajo todas las fuerzas), riesgo de no abrir o de enredarse con las líneas del paracaídas, y el enlace durante la espera en la plataforma igual necesita una antena plegada que funcione. **No recomendada** en esta etapa.

#### Sobre los "10 dBi" de los látigos baratos

En Mercado Libre y Amazon abundan látigos de 13–19 cm que dicen **10 dBi**. Para una antena omnidireccional, llegar a ~10 dBi a 915 MHz requiere del orden de **1 m o más** de largo (varias secciones colineales), como las de fibra de vidrio de 6–8 dBi que miden entre 0,5 y 1 m. Un látigo de 19 cm, con λ/4 ≈ 82 mm, solo da para ~2–3 dBi. Un comprador que midió "antenas de 3 dBi" las describe como de ~2,5 dBi reales. **Tomen 2–3 dBi y verifiquen.**

### 4.4 Tabla comparativa — 915 MHz (datos a sacar de cada alternativa, según el SOTA)

| Alt. | Tipo | Ganancia (declarada → real esperada) | Patrón / pol. | Largo o dimensiones | Masa | Conector / montaje | Banda de adaptación | Dentro / fuera del contenedor |
|---|---|---|---|---|---|---|---|---|
| **A1** | Rubber duck λ/4 | 2–3 dBi → 1–2 dBi | Omni "dona", lineal vertical | 100–110 mm (algunas 195–210 mm) | ~9 g (ficha uElectronics) | SMA / RP-SMA; soporte mecánico propio | 860–960 MHz típico | Dentro, a lo largo del eje |
| **A2** | Monopolo λ/4 de alambre | 0–2 dBi (estimación mía) | Omni "dona", lineal vertical | ≈ 77–79 mm de alambre | < 1 g | Soldado a placa / U.FL / pigtail; abrazadera impresa | Se ajusta a 915–928 MHz recortando | Dentro, a lo largo del eje |
| **A3** | Látigo corto helicoidal / spring | 0–2 dBi (declaran 2 dBi) | Omni, lineal vertical | 50–100 mm | pocos gramos | SMA / U.FL con pigtail | Más angosta, ajustar | Dentro |
| **A4** | Dipolo λ/2 | ~2 dBi | Omni, lineal vertical | ≈ 156–163 mm | 1–5 g | Cable coaxial + soporte | Sin dependencia fuerte del plano | Dentro (difícil por largo) |
| **A5** | Chip / SMD helicoidal | ≈ 2 dBi pico con placa grande; menos con placa chica | Irregular | 10 × 10 × 11 mm | 0,7 g (HA.31.A) | Soldada a la placa, con adaptación | Angosta, diseño específico | Dentro, sobre la placa |
| **A6** | Impresa en placa | ≈ −1 dBi o menos | Irregular | Tramo de pista en la placa | 0 g | Ninguno | Angosta, sensible a la mano/entorno | Dentro, en la placa |
| **A7** | Parche | 3–5 dBi → direccional | Direccional, lineal | ≈ 80 × 80 mm | 10–20 g | Soldado / coaxial | Angosta | Dentro (ocupa mucho) |
| **A8** | Desplegable | 2–3 dBi | Omni | Variable | Variable | Mecanismo | Variable | Plegada dentro, abierta fuera |

### 4.5 Tabla comparativa — 2,4 GHz

Valores de referencia de la tabla de antenas aprobadas de Digi (módulos XBee 2,4 GHz) y listados argentinos.

| Alt. | Tipo | Ganancia | Dimensiones | Observaciones |
|---|---|---|---|---|
| **B1** | Dipolo λ/2 articulado (RP-SMA) | 2,1 dBi (Digi) | 4,5" ≈ 115 mm | Típico de módulos XBee; la articulación no es apta para S8 sin fijarla |
| **B2** | Látigo λ/4 integrado | 1,5 dBi (Digi) | ≈ 30 mm de elemento | Corto, cabe fácil |
| **B3** | Monopolo λ/4 de alambre | ~0–2 dBi (estimación mía) | ≈ 29–30 mm | Hecho por ustedes, se mide |
| **B4** | Chip de montaje superficial | −1,5 dBi (Digi) | milímetros | Pierde 3–4 dB frente al látigo |
| **B5** | Antena de placa integrada del módulo | −0,5 a 0 dBi (Digi) | en la placa | Sensible al entorno |
| **B6** | Adhesiva flexible (FPC) con U.FL | ~2–5 dBi declarados | 30–50 mm | Se pega adentro de la pared de plástico; se consigue barata, **medir** |
| **B7** | Omni de Wi-Fi RP-SMA (2–5 dBi) | 2–5 dBi | 10–15 cm | En Argentina cuestan ~AR$ 5.500–6.500 (2–3 dBi) |

**Observación:** a 2,4 GHz el CanSat tiene mucho más margen de espacio (λ/4 ≈ 30 mm), pero **cada dB que se pierde en el CanSat hay que recuperarlo en tierra**, y la pérdida de espacio libre es mayor.

### 4.6 Matriz de decisión — antena del CanSat a 915 MHz (juicio mío)

Puntaje de 1 (malo) a 5 (muy bueno). Los pesos suman 100.

| Criterio | Peso | A1 Rubber duck | A2 Alambre λ/4 | A3 Látigo corto | A4 Dipolo λ/2 | A5 Chip | A6 Impresa | A7 Parche |
|---|---|---|---|---|---|---|---|---|
| Cobertura del patrón en el descenso | 25 | 4 | 4 | 3 | 4 | 2 | 2 | 1 |
| Eficiencia / ganancia real | 15 | 4 | 4 | 2 | 4 | 1 | 1 | 3 |
| Robustez mecánica (15 G / 30 G) | 15 | 2 | 4 | 4 | 2 | 5 | 5 | 3 |
| Espacio y masa | 15 | 2 | 4 | 5 | 1 | 5 | 4 | 2 |
| Facilidad de integración y ajuste | 10 | 4 | 3 | 4 | 3 | 2 | 2 | 2 |
| Costo y disponibilidad en Argentina | 10 | 5 | 5 | 4 | 4 | 3 | 5 | 3 |
| Resistencia al desajuste (baterías, carcasa) | 10 | 3 | 3 | 2 | 4 | 1 | 1 | 3 |
| **Total (sobre 100)** | | **68** | **78** | **68** | **63** | **55** | **56** | **45** |

**Lectura:** el monopolo de alambre λ/4 (A2) gana por masa, espacio y robustez; la rubber duck (A1) y el látigo corto (A3) empatan en segundo lugar. Los pesos y puntajes son discutibles: si el equipo no tiene acceso a un analizador de redes, A1 sube a ser la opción más segura.

### 4.7 Recomendación, montaje y plan B

**Principal: A2 (monopolo λ/4 de alambre, ~77–79 mm a 915 MHz).**

**Plan B / baseline de ensayo: A1 (rubber duck de 2–3 dBi), con soporte mecánico.** Recomiendo **comprar una A1 de todas formas** para tener una referencia de enlace conocida contra la cual comparar la A2 en las pruebas de alcance.

**Reglas de montaje:**

1. **Eje vertical y plano de tierra:** la antena sale del borde de la placa, perpendicular a ella, con la placa lo más grande posible como plano de tierra.
2. **Distancia a metales:** al menos ~30 mm (λ/10 aproximadamente) de las baterías 18650 y de cualquier masa metálica. Las baterías en el extremo opuesto.
3. **Extremo del CanSat:** el lado del paracaídas está libre de metales (tela y cuerdas), aunque hay que cuidar que las cuerdas no se enreden en la antena. El lado inferior suele tener cámara nadir, baterías y servos, que son metálicos.
4. **Fijación mecánica (S4, S5, S8):** abrazadera impresa o soporte sobre la placa, no depender solo del cordón de soldadura ni del conector. Dejar un pequeño alivio de tensión.
5. **Conector:** U.FL a SMA o pigtail corto. Cada conector agrega unos 0,1–0,2 dB. Cables finos (pigtails U.FL, RG174) pierden del orden de 1 dB por metro a 900 MHz y más a 2,4 GHz (orden de magnitud, verificar con la ficha del cable). Usar el más corto posible.
6. **Material del contenedor:** PLA o PETG simples. Nada con fibra de carbono ni relleno metálico. Estructuras define el material, avisenle.
7. **GPS:** separar físicamente la antena de radio de la antena GPS. Probar el fix con la radio transmitiendo.
8. **Mantener la polarización vertical** con el CanSat colgado, que es lo más probable.

### 4.8 Dónde conseguirlas en Argentina

**Precios: capturas de buscador, fechas mezcladas, reverificar el día de comprar.**

| Producto | Tipo / ganancia declarada | Precio visto (AR$) | Vendedor / nota |
|---|---|---|---|
| Antena LoRa 915 MHz "Spring" 10 cm (Dimalls) | Helicoidal corta, 2 dBi | ≈ 59.000 | Mercado Libre; **compra internacional, envío desde EE. UU.** |
| Antena Whip 50 mm 900 MHz / 915 / 916 MHz (ITYT) | Látigo corto | ≈ 16.500 | Mercado Libre; conector según listado |
| Antena LoRa 915 MHz 5 dBi SMA macho (GLIC) | Omni, declara 5 dBi | ≈ 65.000–70.000 | Mercado Libre; envío internacional. Tomar la ganancia con cautela |
| Antena LoRa U.FL/IPEX a SMA 868–915 MHz (Starware) | Látigo ~5 cm con pigtail de 12 cm, declara 3 dBi | $3.699 en una captura **vieja** | Tienda oficial Starware en Mercado Libre; hoy debe costar bastante más |
| Rabbit-Labs SMA 915 MHz, 3,5 dBi | Omni | ≈ 51.900 | Frávega (vendido por GLIC, compra internacional) |
| RFM95 / módulos LoRa 915 MHz (referencia del radio) | — | ≈ 20.000–61.000 según versión | EC Electrocomponentes y otros en Mercado Libre; sirve de referencia para el costo total de C9 |

**Cómo comprar bien:**

- Pregunten el **tipo de conector**: SMA y RP-SMA no son intercambiables (las "Wi-Fi" suelen ser RP-SMA). Verifiquen que el conector de la antena coincida con el del módulo, o compren el adaptador.
- Las tiendas con envío desde el exterior (Amazon vía Tiendamia, Desertcart, Mercado Libre internacional) demoran más y pueden agregar impuestos. Comparen el plazo con el calendario de la PDR.
- En importaciones directas de distribuidores (por ejemplo DigiKey), los plazos de fábrica de algunas antenas llegan a 5–17 semanas cuando no hay stock (ejemplos que vi: JC Antenna 5 semanas, TE/Laird PC9013N entre 4 y 17 semanas). Para un CanSat, **conviene priorizar lo que está en stock local**.
- **Compren al menos 2 o 3 unidades** de cada antena candidata: una para ensayos destructivos (vibración e impacto).

---

## 5. Punto 3 — Antena de la estación terrena

### 5.1 Restricciones

| Restricción | Requisito | Efecto |
|---|---|---|
| Antena de mesa o portátil | G11 | Una Yagi en trípode o una omni en mástil son portátiles. Una torre no |
| Portátil, a lo largo de la línea de vuelo, sin CA | G10 | Solo antenas pasivas. Nada con amplificador alimentado de la red |
| Incluida en el equipo con notebook y radio | G9 | Dice "una antena" (singular). Ver 5.7 |
| Banda | X1 | 915–928 MHz o 2400–2483,5 MHz |
| Límite regulatorio | Res. 581/2018 | Más de 6 dBi obliga a bajar la potencia de transmisión (sección 2) |
| Costo | C9 | La estación no cuenta, porque C9 excluye las "herramientas de análisis y soporte en tierra" (lectura del SOTA) |

**Derivadas (mías):**

- La estación **transmite poco** (solo telecomandos: calibrar altitud, configurar hora, pedir estado). **El enlace crítico es el de bajada**: el CanSat transmite a 1 Hz y la estación tiene que recibirlo.
- **Hay otros equipos en el campo** con CanSats transmitiendo en la misma banda. Una antena directiva rechaza parte de esas señales; una omni, no.
- **El CanSat se mueve**: baja desde 500 m–1 km en ~1–3 minutos y deriva con el viento. Una antena que hay que apuntar le suma una tarea operativa a alguien del equipo.

### 5.2 Alternativas

#### S1. Omnidireccional de ganancia baja (látigo/dipolo, 2–3 dBi)

- Es la antena que ya trae la mayoría de los dongles y placas. Se puede alargar con una extensión y ponerla en un mástil.
- **Pros:** sin apuntar, cobertura de 360°, patrón amplio en elevación (el dipolo tiene ~78° de ancho de haz vertical), simple, barata.
- **Contras:** poca ganancia, no rechaza interferencia, igual tiene el nulo hacia el cenit.

#### S2. Omnidireccional colineal de fibra de vidrio (5–8 dBi)

- Ejemplos: antenas "Helium / LoRa" de 5,8 dBi o 8 dBi para exterior, con conector N. En Mercado Libre Argentina vi desde unos $25.000 (captura vieja) hasta $145.000–$224.000 (5,8 dBi) y $278.000–$398.000 (8 dBi Raigen).
- **Pros:** más ganancia sin apuntar, robusta, pensada para exteriores.
- **Contras:** mide entre 0,5 y 1 m; **mayor ganancia significa menor ancho de haz vertical** (del orden de 30° para 5 dBi y 15–20° para 8 dBi, valores típicos orientativos), así que el CanSat puede quedar fuera del haz a mucha elevación. No usar más de ~5 dBi.

#### S3. Yagi pequeña de 3 elementos (≈ 7–8 dBi)

- Ejemplos: Amphenol PCTEL MYA9153 (3 elementos, 8,1 dBi, N hembra); diseño DIY de Meshtastic de 3 elementos con ~7 dBi y un haz de 120–130° (según el autor).
- **Pros:** ganancia útil con un haz ancho (poca necesidad de apuntar fino), liviana.
- **Contras:** se vende poco en Argentina; DIY requiere medir.

#### S4. Yagi de 6–10 elementos (10–15 dBi)

- Ejemplos: SIGNALPLUS 10 dBi (8 elementos, ~0,7 m, 0,8 kg, haz ~54° × 48°); NiceRF SW915-BM-11DB (11 dBi, 890 × 180 × 58 mm, 510 g, haz 42° × 38°, VSWR < 1,5); LPRS YAGI-868/915A (9 elementos, 13 dBi, 790 × 190 × 55 mm, SMA); Sirio SY-906 (11,5 dBi) y SY-910 (15 dBi); Ventev TerraWave 900 MHz (15 dBi).
- **Pros:** +8 a +13 dB respecto de una omni, rechaza señales de otras direcciones (útil para la convivencia con otros equipos).
- **Contras:** hay que **apuntarla** y seguir al CanSat; es larga (0,8–1,4 m) y se vuelve incómoda de portar; a 900 MHz la oferta en Argentina es pobre (vi un Yagi de 10 dBi en Mercado Libre Argentina, sin precio legible, y la TerraWave de 15 dBi en Desertcart Argentina por AR$ 460.418, sin stock).

#### S5. Panel o parche direccional (8–10 dBi a 900 MHz; 12–14 dBi a 2,4 GHz)

- A 2,4 GHz hay muchísima oferta de paneles de 14 dBi: TP-Link TL-ANT2414A (14 dBi, haz de 30° × 30°, RP-SMA, cable de 1 m), D-Link ANT24-1400 (14 dBi, 30° × 30°), Alfa APA-L2414A (14 dBi, haz 15° × 30°, 230 × 230 mm, N hembra), L-com 14 dBi.
- A 915 MHz los paneles se consiguen poco y son grandes.
- **Pros:** compactos a 2,4 GHz, planos, fáciles de montar en trípode.
- **Contras:** haz angosto (30° o menos a 2,4 GHz); hay que apuntarlos.

#### S6. Helicoidal de modo axial (polarización circular)

- Da polarización circular, que **elimina los nulos por desalineación de polarización** a cambio de 3 dB fijos frente a una antena lineal. A 2,4 GHz una hélice de 6 vueltas mide ~50–60 cm (estimación mía) y da ~10–12 dBic. A 915 MHz es enorme.
- **Contras:** más difícil de conseguir y de armar; igualmente hay que apuntarla. Solo tiene sentido a 2,4 GHz.

#### S7. Combinación: omni como principal + Yagi o panel como respaldo

- Las dos conectadas a un conmutador o intercambiadas a mano antes del vuelo.
- **Pros:** cobertura total más margen extra si hace falta.
- **Contras:** complejidad operativa y duda con G9 (sección 5.7).

### 5.3 Tabla comparativa (datos a sacar, según el SOTA)

| Alt. | Banda | Ganancia | Directividad | Pol. | Largo | Masa | Conector | Montaje | Alcance (estimación mía, ver 5.4) |
|---|---|---|---|---|---|---|---|---|---|
| **S1** Omni baja | 915 o 2,4 | 2–3 dBi | Todo el cielo en horizontal, ~78° vertical | Lineal vertical | 9–21 cm | 10–30 g | SMA / RP-SMA | Directo al dongle o con extensión en mástil | 1–3 km con margen a 915 MHz |
| **S2** Omni colineal | 915 | 5–8 dBi (declarado) | 360° horizontal; vertical ~30° (5 dBi), 15–20° (8 dBi) | Lineal vertical | 0,5–1 m | ~100–400 g (estimación) | N o SMA | Mástil | 3–5 km |
| **S3** Yagi 3 el. | 915 | 7–8 dBi | Haz ancho (~100–130°) | Lineal | ~0,3–0,4 m (estimación) | < 300 g | N / SMA | Trípode | 3–5 km |
| **S4** Yagi 6–10 el. | 915 | 10–15 dBi | Haz 40–55° | Lineal (se puede girar) | 0,7–1,4 m | 0,5–0,8 kg | N / SMA / FME | Trípode con rótula | 5 km o más, **apuntada** |
| **S5** Panel | 2,4 (y 915 menos común) | 12–14 dBi (2,4) | 15–30° | Lineal (se puede girar) | 23 cm de lado (2,4) | ~0,5–1 kg | N / RP-SMA | Trípode con rótula | 3–5 km, **apuntada** |
| **S6** Helicoidal | 2,4 | 10–12 dBic (estimación) | 40–50° | Circular | 50–60 cm | ~0,3–0,5 kg | N / SMA | Trípode con rótula | 3 km o más, apuntada |

**Cable y conectores:** usar el cable más corto posible. Como orden de magnitud, el RG58 pierde del orden de 0,5–0,6 dB por metro a 900 MHz y más a 2,4 GHz; cables más gruesos (tipo LMR-240 o RG213) pierden aproximadamente la mitad. Cada adaptador agrega ~0,1–0,2 dB. Verificar con la ficha del cable que elijan.

### 5.4 Presupuesto de enlace (cálculo mío)

**Supuestos:** distancia de 3 km (el límite superior del SOTA); potencia de transmisión de 20 dBm; antena del CanSat de 2 dBi con un **margen de patrón y desajuste de −6 dB** (caso razonable) y **−15 dB** (caso malo, con orientación desfavorable); pérdida de espacio libre FSPL = 32,44 + 20·log(d en km) + 20·log(f en MHz):

- 915 MHz a 3 km: **101,2 dB**
- 2440 MHz a 3 km: **109,7 dB**

| Caso | Antena de estación | Cable | Potencia recibida (−6 dB de patrón) | Margen sobre −100 dBm | Margen con −15 dB de patrón |
|---|---|---|---|---|---|
| 915 MHz + omni 3 dBi | 3 dBi | 1 dB | **−83,2 dBm** | 16,8 dB | 7,8 dB |
| 915 MHz + Yagi 11 dBi (apuntada) | 11 dBi | 1 dB | **−75,2 dBm** | 24,8 dB | 15,8 dB |
| 2,4 GHz + omni 3 dBi | 3 dBi | 1,5 dB | **−92,2 dBm** | 7,8 dB | **−1,2 dB (no cierra)** |
| 2,4 GHz + panel 14 dBi (apuntado) | 14 dBi | 1,5 dB | **−81,2 dBm** | 18,8 dB | 9,8 dB |

**Cómo leer la tabla:**

- Uso **−100 dBm como sensibilidad conservadora**. Las modulaciones tipo LoRa pueden recibir con mucho menos (en el orden de −111 a −123 dBm según parámetros), lo que da 10–25 dB más de margen. **El dato exacto lo da Radio.**
- **A 915 MHz una omni alcanza con holgura.** No hace falta apuntar nada.
- **A 2,4 GHz con omni, el caso malo no cierra** si la modulación no es muy sensible. Con un panel apuntado sí cierra.
- Es el cálculo de espacio libre. **No incluye** pérdida por vegetación, terreno ni el fuselaje, ni el desvanecimiento por reflexión en el suelo.
- El enlace en la plataforma (G1) es a corta distancia y sobra margen, pero **el material del fuselaje puede atenuar mucho**: la fibra de vidrio y el cartón son casi transparentes, la fibra de carbono y el metal pueden bloquear (sección 8).

### 5.5 Apuntado y seguimiento (cálculo mío)

Para una antena direccional hay que saber cuánto hay que girar:

- Con el CanSat a ~1 km de distancia horizontal y bajando de 1 km de altura, la elevación pasa de 45° a ~0° en unos 200 s: **~0,2 °/s en promedio**.
- La deriva horizontal con 10 m/s de viento a 1 km de distancia agrega unos **0,6 °/s** en el peor momento.
- Con un haz de ~40° de ancho (Yagi de 11 dBi) alcanza con **reapuntar cada 20–30 s**. Con un haz de 15–30° (panel de 14 dBi) hay que reapuntar más seguido y con más cuidado.

**Recomendaciones operativas:**

1. Un integrante dedicado al apuntado, con una marca visual (mira) en el soporte.
2. Mástil o trípode de **2–3 m** de altura si es posible, para reducir pérdidas por el suelo.
3. Antena **lejos de la notebook y de personas** (al menos 1 m): el USB 3.0 es una fuente conocida de ruido a 2,4 GHz, y el cuerpo humano bloquea. Usar extensión USB y coaxial para separar.
4. **Ubicar la estación a cierta distancia lateral de la zona de caída** (ver 4.2) para no quedar bajo el nulo del patrón del CanSat.

### 5.6 Matriz de decisión — antena de la estación a 915 MHz (juicio mío)

| Criterio | Peso | S1 Omni baja | S2 Omni colineal | S3 Yagi 3 el. | S4 Yagi 6–10 el. | S5 Panel 900 | S6 Helicoidal | S7 Omni + Yagi |
|---|---|---|---|---|---|---|---|---|
| Cobertura con CanSat móvil y rotando | 25 | 4 | 3 | 3 | 2 | 2 | 3 | 5 |
| Ganancia / margen de enlace | 20 | 2 | 3 | 3 | 5 | 4 | 4 | 4 |
| Simplicidad operativa (sin apuntar) | 15 | 5 | 5 | 3 | 2 | 2 | 2 | 3 |
| Portabilidad (G10, G11) | 10 | 5 | 3 | 4 | 2 | 3 | 1 | 3 |
| Rechazo de interferencia de otros equipos | 10 | 1 | 1 | 3 | 5 | 4 | 4 | 4 |
| Costo y disponibilidad en Argentina | 10 | 5 | 3 | 3 | 3 | 2 | 1 | 3 |
| Cumplimiento (límite de PIRE) | 10 | 5 | 5 | 5 | 4 | 5 | 5 | 5 |
| **Total (sobre 100)** | | **75** | **66** | **66** | **64** | **60** | **59** | **80** |

**Lectura:** la combinación omni + Yagi (S7) puntúa más alto, pero **depende de que G9 lo permita**. Si solo se permite una antena, **S1 gana** por simplicidad y porque a 915 MHz el margen alcanza.

### 5.7 Recomendación

**915 MHz (mi default):**

1. **Principal: S1 — omnidireccional vertical de 2–5 dBi**, en un mástil o trípode de 2–3 m, con el cable más corto posible. No pasar de ~5 dBi para no angostar demasiado el haz vertical (5.2, S2).
2. **Opcional: S4 — Yagi de 10–11 dBi** (SIGNALPLUS o equivalente) con un integrante apuntando, para ganar 8 dB y filtrar interferencia.
3. **Sobre G9:** dice que la estación incluye "una radio y una antena". Interpreto que es el mínimo, pero **conviene consultarlo antes de llevar dos**. Alternativa sin riesgo: llevar la Yagi como repuesto/ensayo, y declarar solo la omni.

**2,4 GHz (si Radio elige esa banda):**

1. **Principal: S5 — panel de 12–14 dBi** (TP-Link, Ubiquiti, Mikrotik o similares, comunes en Argentina), con trípode y apuntado.
2. **Respaldo:** omni de 3–8 dBi (por ejemplo, TP-Link TL-ANT2408C de 8 dBi, ~AR$ 15.460 en una captura).
3. Evaluar **S6 (helicoidal)** si hay tiempo, por la inmunidad a la polarización.

### 5.8 Dónde conseguirlas en Argentina

**Precios: capturas de buscador, fechas mezcladas, reverificar.**

**915 MHz:**

| Producto | Tipo | Precio visto (AR$) | Nota |
|---|---|---|---|
| Antena LoRa 915 MHz 5,8 dBi para exteriores (GLIC / Boobrie, N) | Colineal fibra | ≈ 145.000–224.000 | Mercado Libre, envío internacional |
| Antena LoRa Raigen 8 dBi exterior | Colineal fibra | ≈ 278.000–398.000 | Mercado Libre; 14 días de plazo según listado |
| Antena Helium Raigen 5,8 dBi | Colineal | $25.000 en una captura **vieja** | Mercado Libre |
| Antena 915 MHz 10 dBi "Meshtastic" (Yagi) | Yagi 10 dBi | no pude leerlo | Mercado Libre Argentina (MLA-2027482204) |
| Ventev TerraWave 900 MHz 15 dBi | Yagi 15 dBi | 460.418 | **Desertcart Argentina, sin stock** al 2026-05 |

**2,4 GHz (Mercado Libre Argentina):**

| Producto | Tipo | Precio visto (AR$) |
|---|---|---|
| Omni Wi-Fi 2,4 GHz 2 dBi (CANDYHO) | Omni | ≈ 5.500 |
| Omni Wi-Fi 2,4 GHz 3 dBi | Omni | ≈ 6.500 |
| Omni Wi-Fi 2,4 GHz 12 dBi (CANDYHO) | Omni | ≈ 26.900 |
| TP-Link TL-ANT2408C | Omni 8 dBi | ≈ 15.460 |
| Antena bibanda 2,4/5,8 GHz RP-SMA 2 dBi, 108 mm (ITYT) | Omni | ≈ 19.184 |
| Yagi de refuerzo para control de dron 2,4/5,8 GHz | Yagi chica | ≈ 8.999–19.999 (**ganancia y haz no especificados**) |
| Paneles Ubiquiti / TP-Link / Mikrotik de 14–15 dBi | Panel | **no verifiqué precio ni stock en Argentina**; son productos de red comunes, hay que consultar |

**Referencia de precio en el exterior** (para comparar importar vs. comprar acá): Yagi de 13 dBi de Embedded Works a US$ 53–75, Sirio SY-910 en ~32 € (revendedor europeo), panel de 14 dBi a 2,4 GHz de L-com a US$ 109,99.

**Consejos:**

- Pregunten por **conector (N, SMA, RP-SMA, FME)** y por **polarización** (las Yagi se pueden montar en vertical u horizontal; el CanSat va vertical).
- Para una Yagi o un panel hace falta **trípode con rótula**. Se consigue en comercios de fotografía o de construcción.
- Las antenas de la estación **no cuentan para C9**, así que acá conviene gastar en calidad.

---

## 6. Interacciones con otros puntos y decisiones a coordinar

| Con quién | Qué hay que decidir en conjunto |
|---|---|
| **Radio (punto 1)** | Banda (915 vs 2,4 GHz), potencia de transmisión, sensibilidad, conector del módulo (SMA, RP-SMA, U.FL o solo pad de soldadura), canales dentro de 915–928 MHz |
| **Estructuras** | Material del contenedor (PLA o PETG, no cargado), espacio para la antena y su soporte, el lado del CanSat donde va (alejado de las baterías), si hay separación en dos cuerpos |
| **Aviónica** | Posición de baterías, GPS y paneles solares respecto de la antena; plano de tierra de la placa |
| **Interfaz notebook (punto 4)** | Ubicación del dongle y longitud del cable de antena; ruido de USB 3.0 |
| **Protocolo (punto 5)** | X2 ("sin broadcast"): en el reglamento original de CanSat (EE. UU.) se refiere a los módulos XBee, que tienen un modo de difusión. La lectura del SOTA (direccionamiento por destinatario) es consistente con eso |
| **Energía** | La antena es pasiva, no consume. Pero una potencia mayor para compensar una mala antena sí consume (E7: 2 h) |

---

## 7. Plan de ensayos de antena

Para dejar evidencia en la PDR/CDR y detectar problemas a tiempo:

1. **VSWR / S11** con NanoVNA o analizador del laboratorio, **con el CanSat armado** (baterías, cámara, contenedor cerrado). Meta: VSWR < 2 (idealmente < 1,5) en toda la banda de trabajo (915–928 MHz, centro ~921 MHz). Repetir con la mano cerca y con el paracaídas guardado.
2. **Desajuste por entorno:** comparar VSWR antes y después de armar el contenedor y de cargar el paracaídas.
3. **Patrón de radiación:** CanSat girando sobre un soporte no metálico a 20–50 m de la estación, registrando RSSI cada 15° en dos planos (horizontal y vertical). Verifica el nulo de 4.2.
4. **Alcance en línea de vista:** a 500 m, 1 km y 2–3 km, con el CanSat sostenido en alto (edificio, mástil o con autorización, un globo).
5. **Atravesar el fuselaje:** con el CanSat dentro de un tubo del mismo material que el cohete (cuando CONAE lo informe) o, mientras tanto, de fibra de vidrio y de cartón, para medir cuántos dB se pierden.
6. **Vibración y choque:** 15 G de vibración (S4) y 30 G de impacto (S5), con revisión visual y de VSWR antes y después.
7. **Convivencia del GPS:** medir el fix de GPS con la radio transmitiendo a 1 Hz.
8. **Comparación A2 vs A1:** misma prueba de alcance con la antena de alambre y con la rubber duck de referencia.
9. **Laboratorios y ayuda:** universidades con laboratorio de RF y radioaficionados (por ejemplo, vía el Radio Club Argentino) suelen tener analizadores y experiencia con estos ajustes.

---

## 8. Datos faltantes y preguntas

**Para CONAE** (las consultas son por mail en la etapa de inscripción, según el SOTA: cansat.universitario@conae.gob.ar):

1. **Material y diámetro interior del fuselaje del cohete.** Determina si el enlace funciona en la plataforma y en el ascenso. Si es fibra de carbono o tiene partes metálicas, la antena tendría que ir afuera o el enlace de G1 se complica.
2. **¿Se permite una antena que sobresalga del contenedor o desplegable?** (S3, S7). Mi recomendación es no usarla, pero conviene que esté claro.
3. **G9 ("una antena"):** ¿se admite llevar más de una antena en la estación (por ejemplo, omni y Yagi)?
4. **Ubicación de las estaciones** y distancia máxima a la zona de caída en Falda del Cañete (Bases §7).
5. **Normativa:** ¿rige el régimen de la Res. 581/2018 (30/36 dBm) o el de dispositivos de baja potencia (campo de 50.000 µV/m a 3 m)? ¿Hay exigencia de homologación para el prototipo?
6. **Asignación de canales:** cuántos equipos vuelan a la vez, y cómo se reparte el canal.

**Para otros equipos del proyecto:**

7. **Banda elegida por Radio**, potencia, sensibilidad y tipo de conector del módulo.
8. **Material del contenedor** (Estructuras) y **posición de las baterías** (Aviónica).
9. **Distancia horizontal máxima real** entre la estación y el CanSat.

**Para verificar el día de comprar:**

10. Precio y stock reales (las capturas de este documento son de fechas mezcladas).
11. Tipo de conector (SMA vs RP-SMA) de cada antena y de cada módulo.
12. VSWR medido de las antenas compradas en 915–928 MHz.

---

## 9. Fuentes

**Normativa argentina**

- Res. 581/2018, anexo "Bandas de frecuencias de uso compartido": https://argentina.gob.ar/normativa/314174_res581_pdf/archivo
- Norma técnica ENACOM-Q2-60.14 V20.1 "Dispositivos de Baja Potencia": https://argentina.gob.ar/normativa/349321_ressint551-2_pdf/archivo
- Resumen de la Res. 581/2018 (Marval): https://www.marval.com/Publicacion/nueva-resolucion-que-favorece-el-desarrollo-de-la-internet-de-las-cosas-13246
- Res. ENACOM 1132/2023, considerandos sobre las Res. 581/2018 y 4.653/2019 (Boletín Oficial): https://www.boletinoficial.gob.ar/detalleAviso/primera/292414/20230818
- Aviso sobre la Res. 551/2021: https://ib-lenhardt.com/news/enacom-revised-the-regulation-4-653-2019
- Cambios de la Res. 570/2018 (cetecom): https://cetecomadvanced.com/en/news/regulatory-changes-for-low-power-devices-in-argentina/
- The Things Network, Argentina (plan AU915): https://www.thethingsnetwork.org/country/argentina/

**Fichas técnicas de antenas de 915 MHz**

- Taoglas HA.31 (helicoidal): https://taoglas.com/product/ha-31-915mhz-helical-monopole-antenna
- Silicon Labs AN847, antenas de placa para 915 MHz: https://silabs.com/documents/public/application-notes/AN847.pdf
- Maxtena MEA-915-01-SMA: https://shop.stepglobal.com/documents/productpdf/MEA-915-01-SMA.pdf
- SparkFun WRL-14875 (λ/4, 2 dBi): https://www.sparkfun.com/915mhz-lora-antenna-rp-sma-1-4-wave-2dbi.html
- Seeed 2,6 dBi, 195 mm: https://core-electronics.com.au/2-6dbi-lora-antenna-sma-male-915mhz-195mm.html.plain.md
- ELSYS ANT915 (3 dBi, articulada): https://embeddedworks.net/product/ante1424/
- uElectronics 868–915 MHz 3 dBi: https://uelectronics.com/producto/antena-de-868-a-915-mhz-3dbi-sma-macho/
- Data Alliance 915 MHz 3 dBi: https://www.data-alliance.net/antenna-915mhz-omnidirectional-3dbi-w-rp-sma-lorawan-gsm

**Yagi y direccionales de 900 MHz**

- LPRS YAGI-868/915A: https://www.farnell.com/datasheets/4794655.pdf
- NiceRF SW915-BM-11DB: https://www.nicerf.com/rf-antenna/yagi-antenna-sw915-bm-11db.html
- Amphenol PCTEL MYA9153: https://www.digikey.es/en/products/detail/amphenol-pctel/MYA9153/15998835
- TE Connectivity PC9013N: https://www.digikey.es/en/products/detail/te-connectivity/PC9013N/3520921
- SIGNALPLUS Yagi 915 MHz 10 dBi: https://amazon.com/-/es/SIGNALPLUS-direccional-915mhz-Sensecap-p%C3%A9rdida/dp/B09SYJ91SJ
- Sirio SY-910: https://radiocenter.es/sirio-sy-910-antena-yagi-lora-868-915-gsm/
- Yagi DIY para Meshtastic: https://meshtastic.discourse.group/t/meshtastic-diy-antennas/3855

**2,4 GHz**

- Antenas aprobadas con módulos XBee (Digi): https://docs.digi.com/resources/documentation/Digidocs/90000991/reference/r_antennas_xb_fcc.htm
- TP-Link TL-ANT2414A: https://www.tp-link.com/pt/products/details/tl-ant2414a.html
- D-Link ANT24-1400: https://www.dlink.com.au/ANT24-1400-2-4ghz-outdoor-14dbi-high-gain-directional-panel-antenna
- Alfa APA-L2414A: https://www.alfa.com.tw/collections/antenna-2-4ghz/products/apa-l2414a
- L-com panel 14 dBi: https://www.l-com.com/wireless-antenna-24-ghz-14-dbi-flat-panel-range-extender-antenna-4ft-rp-sma-plug-connector

**Disponibilidad y precios en Argentina (capturas de buscador, reverificar)**

- Mercado Libre, antena LoRa 915 MHz: https://listado.mercadolibre.com.ar/antena-lora-915-mhz
- Mercado Libre, antena LoRa 915: https://listado.mercadolibre.com.ar/antena-lora-915
- Mercado Libre, antena 915 MHz: https://electronica.mercadolibre.com.ar/antena-915-mhz y https://computacion.mercadolibre.com.ar/conectividad-redes/antenas-wireless/antena-915-mhz
- Mercado Libre, antenas Wi-Fi 2,4 GHz: https://listado.mercadolibre.com.ar/antena-grillada-wifi-2.4ghz-de-24-dbi-hasta-45-km-de-alcance
- Mercado Libre, Yagi 915 MHz "Meshtastic": https://articulo.mercadolibre.com.ar/MLA-2027482204-antena-915mhz-lora-10dbi-para-dispositivos-meshtastic-_JM
- Mercado Libre, Starware U.FL a SMA: https://articulo.mercadolibre.com.ar/MLA-858585794-antena-lora-u-fl-ipex-a-sma-c-antena-868-915mhz-_JM
- Mercado Libre, módulos RFM95: https://listado.mercadolibre.com.ar/rfm95-lora
- Desertcart Argentina, Ventev TerraWave: https://www.desertcart.com.ar/products/35582114-terrawave-900-mhz-15-dbi-yagi-antenna
- Frávega, Rabbit-Labs 915 MHz: https://www.fravega.com/rabbit-labs-antena-sma-macho-compatible-con-22755334/p
- Tiendamia Argentina (importación desde Amazon): https://tiendamia.com.ar

**Contexto CanSat**

- Guía de misión CanSat (reglamento original; XBee sin modo broadcast, despliegue violento): https://www.scribd.com/document/611536709/CanSat-Mission-Guide-2023e
