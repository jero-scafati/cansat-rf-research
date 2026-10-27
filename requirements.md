# Acciones para las que hay que buscar alternativas

## 1. Enlace de radio CanSat ⇄ estación (dos sentidos)

**1A. Transmitir telemetría (CanSat → estación)**
- Un paquete por segundo (X4, C5).
- Contenido mínimo:
  - tiempo de misión, resolución 1 s (G3, F2)
  - contador de paquetes transmitidos, que persiste ante reinicios (F1)
  - altitud, presión, temperatura (X5)
  - aceleraciones y rotaciones en 3 ejes (X5)
  - GPS: latitud, longitud, altitud y número de satélites (X5)
  - voltaje y corriente de batería (G7)
  - estado del software de vuelo (G8)
- Enviar automáticamente un mensaje de comienzo de misión al separarse del cohete (X3).
- Los datos deben permitir verificar el cumplimiento de los requisitos de la misión (Bases §4). Con altitud y tiempo a 1 Hz se verifican las velocidades de descenso de C3 (15 ± 3 m/s) y C4 (5 ± 3 m/s).

**1B. Recibir telecomandos (estación → CanSat)**
- Comandos exigidos: ver tabla más abajo (G1, F3, X6, F4, G15).
- Se recibe en la plataforma antes del despegue (G1, F3) y, según X6 y F4, posiblemente durante el vuelo.

**Restricciones que valen para cualquier alternativa del enlace:**
- **Banda:** solo 915–928 MHz o 2400–2483,5 MHz (X1). Lo que quede fuera se descarta.
- **Sin modo broadcast** (X2). Aplica a "las radios", o sea a ambos lados.
- **Convivencia en el campo:** la FRR verifica que la frecuencia sea la asignada para no interferir con otros equipos (Bases §6). Hay que poder cambiar frecuencia o canal.
- **Energía:** el CanSat funciona al menos 2 h integrado en el cohete (E7). Baterías permitidas: alcalina, Ni-Cd, Ni-MH, litio en empaque metálico tipo 18650 o botón. No LiPo (E1, E2). No contactos de resorte para conectar baterías (M4).
- **Masa:** CanSat + contenedor = 500 g ± 10 g (S1).
- **Costo:** el CanSat completo cuesta menos de $1.000.000 (C9). La estación terrena no cuenta, porque C9 excluye las "herramientas de análisis y soporte en tierra".
- **Volumen:** ver antena del CanSat (punto 2).
- **Mecánica:** vibración 15 G (S4), impacto 30 G (S5), todo montado fijo (S8).

**Datos que tenés que sacar de cada alternativa:**
- bandas y canales disponibles
- potencia de salida
- sensibilidad de recepción
- velocidad de datos
- tiempo en el aire de un paquete del tamaño que definan (tiene que entrar en 1 s con margen, considerando también la escucha de comandos)
- si es half o full duplex, y cómo recibe comandos mientras transmite
- si permite direccionamiento punto a punto
- si permite cambiar de canal sin reprogramar
- consumo en transmisión, recepción y reposo, y tensión de alimentación
- masa y dimensiones
- precio y plazo de entrega en Argentina
- alcance reportado en condiciones comparables (línea de vista, altura)

## 2. Antena del CanSat
- **Acción:** irradiar y captar la señal desde dentro del cuerpo del CanSat.
- **Restricciones:**
  - Banda de X1.
  - Entra en Ø 90 mm y 200–250 mm de largo, incluyendo los sistemas de descenso (S2, S3). Las Bases §4 dicen Ø 95 mm y 200 mm (ver inconsistencias).
  - El contenedor debe deslizar libremente fuera del cohete (S7).
  - Si el contenedor es impreso en 3D, la pared mide al menos 2 mm (S6). El material del contenedor afecta la propagación y lo define Estructuras.
  - Soporta 15 G y 30 G (S4, S5) y va montada fija (S8).
  - El CanSat rota durante el descenso (X5 mide rotaciones), así que el patrón de radiación importa.
- **Datos a sacar:** tipo, ganancia, patrón, polarización, dimensiones, masa, conector, banda de adaptación, forma de montaje, y si va dentro o fuera del contenedor.

## 3. Antena de la estación terrena
- **Acción:** captar y emitir hacia el CanSat desde el sitio de operación.
- **Restricciones:**
  - De mesa o portátil (G11).
  - La estación es portátil, se ubica a lo largo de la línea de vuelo y no hay corriente alterna (G10).
  - Incluida en el equipo junto con la notebook y la radio (G9).
  - Banda de X1.
- **Datos a sacar:** ganancia, directividad (si hay que apuntarla o cubre todo el cielo), polarización, banda, masa, forma de montaje, conector, alcance estimado.

## 4. Interfaz radio ⇄ notebook y energía de la estación
- **Acción:** que los datos lleguen a la notebook y que los comandos salgan de ella.
- **Restricciones:**
  - La notebook tiene al menos 2 h de batería (G9).
  - No hay corriente alterna en el sitio (G10), así que la radio de la estación debe funcionar con la batería de la notebook o con otra fuente propia, durante al menos 2 h.
- **Datos a sacar:** tipo de conexión, quién alimenta el receptor y con qué consumo, drivers o librerías para el sistema operativo, formato en que entrega los datos, cómo se envían comandos por esa misma interfaz.

## 5. Protocolo del enlace
- **Acción:** definir cómo se arma, identifica y valida cada paquete y cada comando.
- **Restricciones:**
  - La estación cuenta los paquetes recibidos con éxito. No es lo mismo que el contador de transmitidos (G14).
  - Hay que mostrar paquetes recibidos y perdidos (G8). Para calcular los perdidos el paquete necesita el contador F1, y para contar "con éxito" necesita alguna forma de validar integridad.
  - Direccionamiento punto a punto, por X2.
  - G5 pide que la telemetría se muestre en formato texto en pantalla. No obliga a que el paquete en el aire sea texto.
- **A comparar:** formato del paquete, método de validación de errores, método de identificación de destinatario, orden y estructura de los comandos.

## 6. Software de la estación terrena
- **Acción:** recibir, decodificar, mostrar, graficar, guardar y enviar comandos.
- **Restricciones:**
  - Cada equipo desarrolla su propia estación (G4).
  - Todo en una sola interfaz, simultáneo, sin pestañas (G13).
  - Gráficos en tiempo real de altitud, voltaje, corriente, acelerómetro y tasas de rotación (G7).
  - Mostrar tiempo de misión, temperatura, posición GPS, paquetes recibidos, paquetes perdidos y estado del software de vuelo (G8).
  - Unidades SI indicadas en pantalla (G6).
  - Fuentes de al menos 14 pt, trazos y ejes de gráficos en negrita, texto oscuro sobre fondo claro (G12).
  - Archivos CSV con todos los datos de sensores, guardados en la notebook y en un disco de respaldo (G2).
  - Poder enviar telecomandos (G15).
  - La FRR hace una prueba de estación: telemetría en vivo, gráficos y despliegue funcionando (Bases §6).
- **Datos a sacar de cada alternativa:** si permite una GUI de una sola ventana con gráficos en tiempo real, control de fuentes y estilos, lectura de la interfaz del punto 4, escritura de CSV a dos destinos, envío de comandos desde la misma pantalla, y su consumo de energía (cuenta para las 2 h de G9).

# Roadmap de comandos

| Comando | Sentido | Cuándo | Req. |
|---|---|---|---|
| Calibrar altitud a cero | Estación → CanSat | En plataforma, antes del despegue | G1, F5 |
| Configurar hora (± 1 s vs UTC) | Estación → CanSat | Antes del lanzamiento | F3, F2 |
| Pedir estado de batería y señal de paneles solares | Estación → CanSat, responde por telemetría | No especificado | X6 |
| Pedir información adicional | Estación → CanSat | No especificado. Se documenta en el manual de la misión | F4 |
| Comienzo de misión | CanSat → Estación, automático | Al separarse del cohete | X3 |

Los estados de calibración y de software sobreviven a un reinicio (F5). Los documentos no piden confirmación (ACK) de comandos, así que eso lo deciden ustedes.

# Datos que los documentos no dan y tenés que conseguir

1. **Altitud máxima.** Las Bases dicen solo "superior a 500 m sobre el nivel del terreno". El "1 km" salió de la reunión, no de los documentos, así que confirmen la fuente.
2. **Distancia horizontal máxima** entre la estación y el CanSat. Depende de viento y deriva. Pregunten dónde se pueden ubicar las estaciones en Falda del Cañete (Bases §7).
3. **Tiempo de enlace requerido.** Estimación mía a validar, con 500 m de altitud máxima: el primer 20 % baja a 15 ± 3 m/s (unos 6–8 s) y el 80 % restante a 5 ± 3 m/s (unos 50–200 s). Son entre 56 y 208 paquetes a 1 Hz. Más el ascenso, si hay que transmitir (ver ambigüedad a).
4. **Normativa de ENACOM** para esas bandas. El Anexo solo da las frecuencias. Averigüen si hay límite de potencia u otras condiciones.
5. **Asignación de frecuencias de CONAE:** cuántos equipos vuelan en simultáneo y cómo se asigna el canal a cada uno.
6. **Fuselaje del cohete de CONAE:** material y diámetro interior. Importa si hay que transmitir durante el ascenso.
7. **Material del contenedor** (lo define Estructuras) y **tensión y corriente de la batería** (la define Aviónica).
8. **Tamaño del paquete en bytes.** Lo calculan ustedes a partir de la lista de campos de la acción 1A.

# Ambigüedades e inconsistencias para consultar a CONAE

Las Bases dicen que las consultas por mail (cansat.universitario@conae.gob.ar) se hacen durante el período de inscripción, así que verifiquen si todavía están a tiempo.

- **a. Ascenso.** G5 pide mostrar telemetría durante el ascenso y el descenso. X3 define el comienzo de misión en la separación, y E7 pide 2 h de funcionamiento integrado. ¿Hay que transmitir antes de la separación?
- **b. X3.** Dice que la telemetría "enviará un comando" al separarse, sin decir a quién ni cuál. Mi lectura es que se refiere a un mensaje de inicio de misión hacia la estación, pero conviene confirmarlo.
- **c. X2.** No define qué entienden por "modo de difusión".
- **d. Voltaje y corriente.** G7 los grafica en tiempo real, lo que sugiere que van en cada paquete. X6 dice que el estado de batería se transmite mediante telecomando.
- **e. Dimensiones.** Las Bases §4 dicen Ø 95 mm y 200 mm. El Anexo (S2, S3) dice Ø 90 mm y 200–250 mm. Esto afecta el espacio de la antena.
- **f. Video.** Los documentos solo exigen grabarlo (C6, C7, SN6, SN7), no transmitirlo. No hay requisito de enlace de video.

--------

Para cada una te digo por qué es ambigua, cuál es la lectura más probable y cómo diseñar para que valga aunque me equivoque. Son inferencias mías del texto, CONAE no confirmó nada. Varias frases suenan a traducción literal de otro reglamento (las Bases mencionan CANSAT USA), y por eso se leen raro.

## 1. Telemetría en el ascenso y X3 ("comienzo de misión")

**Por qué es ambigua:** G5 pide telemetría en ascenso y descenso. X3 pone el "comienzo de misión" en la separación y no dice qué se transmite antes. Además dice que "la telemetría enviará un comando", que literalmente no tiene sentido.

**Lectura más probable:**
- El CanSat se enciende dentro del cohete con el interruptor y queda en STAND-BY (E4): prendido, con radio activa, recibiendo comandos (G1, F3).
- Al separarse cambia solo de estado y empieza la misión.
- C3 lo apoya: dice "SE ACTIVARÁ" en mayúsculas después del despliegue.
- Para detectar la separación, E3 y X6 sugieren el panel solar: adentro del fuselaje no hay luz y afuera sí. Esto es lectura mía, el texto no lo dice.

**Cómo cubrir ambas lecturas:**
- Transmitir a 1 Hz desde STAND-BY, con un campo de estado en el paquete (stand-by, ascenso, misión).
- Cambiar de estado automáticamente al separarse y mandar un paquete marcado como inicio de misión.
- Así G5 queda cubierto y X3 también.
- Costo: transmitir durante todo el tiempo en plataforma entra en las 2 h de E7. Pasales ese consumo a Nacho y Martín.

## 2. Voltaje y corriente: ¿en cada paquete o por telecomando?

**Por qué es ambigua:**
- G7 pide graficar voltaje y corriente en tiempo real, lo que exige datos continuos.
- X5, que lista el contenido de la telemetría, no los incluye.
- X6 dice que el estado de batería y la señal de los paneles se transmiten "mediante un telecomando".

**Lectura más probable:** el paquete base lleva todo lo de X5 más tiempo, contador y estado, y el telecomando de X6 devuelve un reporte de batería y paneles.

**Cómo cubrir ambas lecturas:** poner voltaje y corriente en cada paquete (G7) e implementar además el telecomando que devuelve el reporte de batería y paneles (X6). Cuesta unos pocos bytes y no se contradicen.

## 3. X2: "sin modo broadcast"

**Por qué es ambigua:** no define "difusión". En RF todo transmisor emite hacia todos, así que solo puede referirse a direccionamiento.

**Lectura más probable:** que cada paquete tenga un destinatario específico (el de tu estación) y no vaya a "todos". La razón de la regla es no inundar a los otros equipos en el campo.

**Cómo cubrirla:** dirección o ID propio de equipo en el paquete, filtrado en el receptor y canal o frecuencia distinta de los otros equipos. Si el módulo no trae direccionamiento por hardware, hacelo por software y documentalo en el PDR.

## 4. Dimensiones: Bases Ø95 × 200 vs Anexo Ø90 × 200–250

**Por qué es ambigua:** son números distintos para el mismo objeto.

**Lectura más probable:** el Ø95 × 200 de las Bases es el espacio disponible (la bodega del cohete), y el Anexo da las medidas del CanSat dentro de ese espacio. S7 apoya esto: pide que el contenedor "se deslice libremente" fuera del cohete, o sea que tiene que haber juego.

**Cómo cubrirla:** diseñar Ø90 × 200 mm, que es el único punto que cumple ambos documentos. Ojo, el margen en el largo es cero: 200 mm es el mínimo de S3 y el máximo de las Bases. Es la única ambigüedad donde igual consultaría a CONAE, si todavía están a tiempo.

## 5. C4: "80% de la altitud máxima"

**Por qué es ambigua:** puede ser 80% de la altura (baja el primer 20%) o 80% del recorrido (queda el último 20%). En la reunión lo anotaron de otra forma ("80% de altitud restante").

**Lectura más probable:** la literal, que es el 80% de la altitud máxima.

**Efecto en tu parte:** solo cambia la cantidad de paquetes. Con apogeo de 500 m:
- Lectura literal: unos 87 s nominales (56–208 s con las tolerancias).
- Otra lectura: unos 47 s (35–83 s).

Diseñá el enlace para el caso largo.

## 6. Altitud y distancia: no están en los documentos

Las Bases solo dicen "superior a 500 m". El "1 km" salió de la reunión. Como supuesto de diseño, tomá 1 km de altura, descenso de unos 170 s y viento de 10 m/s. Eso da unos 1,7 km de deriva horizontal y unos 2 km en línea recta, así que el enlace debería cubrir 2–3 km. Es una estimación mía, no un dato.

## 7. Enlace dentro del cohete: casi seguro

G1 pide calibrar altitud "en la plataforma de lanzamiento", y ahí el CanSat ya está dentro del cohete (C1). Entonces el enlace tiene que funcionar a través del fuselaje, cuyo material no sabemos. Tenelo en cuenta para la antena y la potencia.

## 8. Costo (C9)

C9 excluye "herramientas de análisis y soporte en tierra". Lo más probable es que la radio y la antena de la estación queden afuera, y que la radio y la antena del CanSat cuenten. Para el PDR armá los dos costos por separado.

## Lo que no es ambigüedad

El video: solo se exige grabarlo (C6, C7, SN6). No hay requisito de transmitirlo.

## Cómo dejarlo en el PDR

Las Bases piden que la CDR resuelva todos los TBD y TBR de la PDR. Podés escribir cada una de estas lecturas como supuesto marcado TBR en la PDR, y eso es legítimo. Así no frenás el diseño y dejás registrado qué asumiste.
