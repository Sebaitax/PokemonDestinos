# Pokémon Destinos — Plan de Desarrollo

## 1. Premisa y pilares

En una región donde el linaje y el dinero dictan el valor de una persona, el viaje Pokémon clásico es un lujo reservado para las élites. Pokémon Destinos es una historia de supervivencia, rebelión y lucha de clases, donde el protagonista no busca simplemente ganar medallas, sino sobrevivir en un mundo diseñado para aplastarlo.

Pilares de diseño:
- **Lucha de clases**: nobles/élite vs. pueblo bajo, con mecánicas y narrativa que reflejan esa fricción.
- **Decisiones con peso**: robar, ayudar, delatar — todo se traduce en dos medidores persistentes.
- **Supervivencia sobre coleccionismo**: las medallas no son el objetivo, sobrevivir y definir tu bando sí.

## 2. Stack técnico

Base: **pokefirered** (descompilación de pret) — reconstrucción en C de FireRed que compila a una ROM idéntica byte a byte. Es la única vía realista para todo lo pedido (karma persistente, robo de objetos, robo de Pokémon), porque requiere tocar el motor (save blocks, batallas, uso de objetos clave), no solo scripts de diálogo.

Herramientas:
- **pokefirered** (repo pret) — código del juego en C.
- **agbcc / binutils / make** — toolchain de compilación GBA.
- **Porymap** — editor visual de mapas.
- **poryscript** — lenguaje de scripting legible que compila a los scripts nativos del juego.
- **Git** para versionar; **GitHub Actions** (opcional) para verificar que cada commit sigue compilando.

Nota legal: nunca se distribuye la ROM de FireRed. Solo se distribuye el parche (`.ups`/`.xdelta`); cada usuario aplica el parche sobre su propia copia legítima.

## 3. Sistemas de decisión

### 3.1 Karma y Respeto (dos medidores)

- **Karma**: moralidad de tus acciones (robar, ayudar, delatar, perdonar). Eje bueno↔malo.
- **Respeto**: cómo te percibe cada clase social por separado (pueblo bajo vs. élite). Permite combinaciones tipo "Robin Hood": karma bajo, pero muy respetado por el pueblo.

Implementación:
- 2+ variables persistentes en el save (o campos nuevos en `SaveBlock1`/`SaveBlock2` si los VARS estándar no alcanzan).
- Funciones C reutilizables: `AdjustKarma(amount)`, `AdjustRespeto(clase, amount)`, con clamps a un rango fijo (ej. -100..100).
- Helpers de poryscript para condicionales narrativos (`if_karma_gte`, `if_respeto_pueblo_gte`, etc.) que ramifican diálogos, precios, aliados disponibles y, al final, el desenlace.
- Pendiente de decidir en diseño: ¿el jugador ve los números exactos en el menú de estado, o solo recibe pistas narrativas/reacciones de NPCs? (lo definimos en la fase 1).

### 3.2 Robo de objetos — "Guantes del Ladrón"

- Objeto clave, registrable al botón **Select** (mismo patrón que el Buscador de Objetos o la Bicicleta en `item_use.c`).
- Uso: el jugador se posiciona **justo detrás** de un NPC (posición relativa + dirección de mirada opuesta) y usa el objeto.
- Tabla de NPCs robables: mapa + id de objeto de evento → ítem robable + probabilidad de éxito + flag de estado.
- Resultado:
  - **Éxito**: se transfiere el ítem, -Karma, posible +Respeto con el pueblo bajo si la víctima es un noble.
  - **Fallo**: el NPC se alerta (diálogo/hostilidad/guardias), -Respeto, sin ítem.
- Reseteo: cada NPC vuelve a tener el ítem robable tras un temporizador (tiempo de juego / pasos) o al disparar un evento de capítulo específico — a definir NPC por NPC en el documento de diseño.
- Regla: cada NPC solo puede ser robado una vez por ciclo de reseteo (no infinito).

### 3.3 Robo de Pokémon — "Cortaplumas"

- Objeto clave. Solo usable sobre un **entrenador ya derrotado en combate normal** y que no haya sido robado antes (flag por entrenador, uno por cada `TRAINER_ID`).
- Al usarlo: se dispara un combate especial contra **un Pokémon aleatorio del equipo ya definido de ese entrenador**.
- Ese combate permite captura (excepción al comportamiento normal de "no Poké Ball en combate de entrenador"), adaptando el patrón que el juego ya usa para el combate scripteado de Wally o la Safari Zone.
- Tasa de captura con modificador reducido respecto a lo normal (ej. ×0.75) para que no sea trivial.
- Tras el combate (captura o huida del Pokémon), el entrenador queda marcado como "ya robado" **permanentemente** — no se puede repetir con ese mismo entrenador.
- Consecuencia narrativa: -Karma, posible variación de Respeto según si el entrenador era élite o pueblo.

Esta mecánica es la pieza de ingeniería más pesada del plan (toca el motor de batalla), así que la tratamos como un *spike* técnico dedicado antes de construir contenido alrededor de ella.

## 4. Hoja de ruta

**Fase 0 — Fundaciones técnicas**
Clonar/organizar pokefirered junto al repo, instalar toolchain, compilar una ROM base idéntica (checksum) para validar el entorno de todos los que colaboren. Definir convención de ramas y commits.

**Fase 1 — Documento de diseño**
Detallar facciones, umbrales de Karma/Respeto y sus efectos, guion de la primera zona jugable (vertical slice), lista de NPCs robables iniciales y primeros 3-5 entrenadores robables de Pokémon con su pool.

**Fase 2 — Spike: variables de Karma/Respeto**
Reservar variables persistentes, funciones de ajuste, integración mínima en menú/UI, helpers de poryscript. Esta fase es prerrequisito de todo lo demás.

**Fase 3 — "Guantes del Ladrón"**
Objeto clave + registro a Select + detección de NPC por detrás + tabla de robables + resultado éxito/fallo + reseteo por tiempo/evento.

**Fase 4 — "Cortaplumas"**
Objeto clave + flag de entrenador robado + combate especial con Pokémon aleatorio del rival + tasa de captura modificada + marca permanente post-combate.

**Fase 5 — Vertical slice narrativo**
Construir en Porymap la primera zona (barrio bajo + zona noble) con NPCs, ambos objetos obtenibles, y 2-3 ramas de historia que reaccionen a Karma/Respeto.

**Fase 6 — Playtesting y balance**
Ajustar probabilidades, rangos, consecuencias. QA de regresión sobre el juego base (gimnasios, guardado, batallas normales).

**Fase 7 — Pulido y distribución**
Gráficos/tiles temáticos, música, empaquetado final como parche `.ups`/`.xdelta`.

## 5. Próximos pasos inmediatos

1. Decidir si `pokefirered` vive como submódulo git dentro de este repo o si el código se fusiona directamente aquí.
2. Instalar el toolchain y confirmar que compila una ROM idéntica a la original (paso de validación de entorno).
3. Escribir el documento de diseño detallado (Fase 1): umbrales exactos de Karma/Respeto, lista concreta de NPCs y entrenadores robables de la primera zona.
4. Prototipar el spike de Karma/Respeto (Fase 2) antes de tocar cualquier mecánica de robo.
