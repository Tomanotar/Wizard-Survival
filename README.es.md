<div align="center">

# ⚡ WIZARD SURVIVAL

### *Rogue-lite · RPG de Acción · Cooperativo Multijugador*

> 🌐 **Lenguaje:** Español | [Read on English](README.md)

<br/>

[![Versión](https://img.shields.io/badge/Versión-1.0.0--beta-blueviolet?style=for-the-badge&logo=roblox)](https://www.roblox.com)
[![Motor](https://img.shields.io/badge/Motor-Roblox%20Studio-red?style=for-the-badge&logo=roblox)](https://create.roblox.com)
[![Lenguaje](https://img.shields.io/badge/Lenguaje-Luau-orange?style=for-the-badge)](https://luau-lang.org)
[![Estado](https://img.shields.io/badge/Estado-Producción%20Activa-brightgreen?style=for-the-badge)](https://www.roblox.com)
[![Visitas](https://img.shields.io/badge/Visitas-1.2K%2B-blue?style=for-the-badge)](https://www.roblox.com)
[![Plataforma](https://img.shields.io/badge/Plataforma-PC%20%7C%20Móvil-lightgrey?style=for-the-badge)](https://www.roblox.com)

<br/><br/>

### 🎮 **[👉 Haz clic aquí para jugar Wizard Survival en Roblox 👈](https://www.roblox.com/es/games/121152223798271/Wizard-Survival)**

<br/>

> **Sobrevivís a la horda. Dominás lo arcano. Construís tu leyenda.**

*Wizard Survival* es un RPG cooperativo multijugador de supervivencia en tiempo real ambientado en un mundo de fantasía oscura invadido por los no-muertos. Los jugadores seleccionan, combinan y evolucionan un sistema de hechizos de gran profundidad para resistir oleadas interminables de enemigos en tres mapas atmosféricos — cada uno con dificultad escalable, mecánicas ambientales propias y devastadores encuentros contra jefes.

El juego toma la accesibilidad y profundidad de progresión de *Vampire Survivors*, la sinergia de builds de *Hades* y la tensión cooperativa de los títulos clásicos de defensa por oleadas, todo construido sobre una arquitectura cliente-servidor artesanal que mantiene el rendimiento sólido incluso con cientos de entidades en pantalla.

</div>

---

## 📖 Tabla de Contenidos

1. [Ciclo de Juego Principal](#-ciclo-de-juego-principal)
2. [Arquitectura Técnica — División Cliente-Servidor](#-arquitectura-técnica--división-cliente-servidor)
3. [Mecánicas Multijugador y Diseño de UX](#-mecánicas-multijugador-y-diseño-de-ux)
4. [Sistema de Magias y Combinaciones Legendarias](#-sistema-de-magias-y-combinaciones-legendarias)
5. [Profundidad de Contenido — Artefactos y Progresión](#-profundidad-de-contenido--artefactos-y-progresión)
6. [Mapas, Level Design y World Building](#-mapas-level-design-y-world-building)
7. [Meta-Juego, Lobby y Economía](#-meta-juego-lobby-y-economía)
8. [UI/UX y Polish Visual](#-uiux-y-polish-visual)
9. [Pipeline de Desarrollo y Notas de Producción](#-pipeline-de-desarrollo-y-notas-de-producción)
10. [Métricas de Lanzamiento, Créditos y Roadmap](#-métricas-de-lanzamiento-créditos-y-roadmap)

---

## 🎮 Ciclo de Juego Principal

```
LOBBY  ──►  SELECCIÓN DE MAPA  ──►  SUPERVIVENCIA  ──►  ENCUENTRO CON JEFE
  ▲                                                             │
  │        ┌───────────────────────────────────────────┐        │
  └────────┤  Cofre Obelisco  ·  Evolución de Magia    │◄───────┘
           │  Caída de Artefactos  ·  Subida de Nivel  │
           └───────────────────────────────────────────┘
```

Cada **run** sigue este ritmo:

| Fase | Duración | Eventos Clave |
|---|---|---|
| **Oleadas Iniciales** | 0 – 3 min | Calibración del enjambre, primera selección de magia, construcción del pool de maná |
| **Escalada Media** | 3 – 8 min | Las sinergias de artefactos se activan, evolucionan las magias raras, la densidad de enemigos alcanza su pico |
| **Fase de Jefe** | Desencadenada por umbral de oleadas | La Arena del Sello Ritual se activa; encuentros con jefes de mecánicas telegráficas |
| **Respiro Entre Oleadas** | ~5s tras la muerte del jefe | Descenso del Cofre Obelisco, ventana de elección de magia/artefacto, reposicionamiento estratégico |

El objetivo de diseño es una **curva de poder no lineal**: el jugador siente progresión exponencial gracias a builds inteligentes, no simplemente por tiempo invertido.

---

## 🔧 Arquitectura Técnica — División Cliente-Servidor

### Ciclo de Vida de un Hechizo: Separación de Autoridad y Renderizado

La decisión arquitectónica central de *Wizard Survival* es una **separación estricta de autoridad**. Cada aspecto de la ejecución de un hechizo está dividido en dos dominios para eliminar la explotación del lag, prevenir desincronizaciones y garantizar equidad en todas las calidades de conexión.

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                    FLUJO DE CICLO DE VIDA DE UN HECHIZO                      │
├─────────────────────────────┬────────────────────────────────────────────────┤
│   CLIENTE (Cosmético)       │         SERVIDOR (Autoridad)                   │
├─────────────────────────────┼────────────────────────────────────────────────┤
│ 1. Input del jugador        │                                                │
│    (clic / tecla)           │                                                │
│         │                   │                                                │
│ 2. Spawn inmediato de VFX   │ 3. RemoteEvent recibido                        │
│    Mesh + estela de         │    Tipo de hechizo, origen y dirección         │
│    partículas (predictivo,  │    validados contra lista blanca               │
│    puramente cosmético)     │         │                                      │
│         │                   │ 4. Cálculo de hitbox                           │
│         │                   │    Álgebra espacial + matemática vectorial     │
│         │                   │    Sin dependencia visual alguna               │
│         │                   │         │                                      │
│         │                   │ 5. Aplicación de daño                          │
│         │                   │    Humanoid:TakeDamage()                       │
│         │                   │    Cooldowns + protección anti-exploit         │
│         │                   │         │                                      │
│ 7. VFX de impacto visual ◄──┼─── 6. Resultados despachados via              │
│    Números de daño,         │        señal de ReplicatedStorage              │
│    screenshake, audio       │        a TODOS los clientes                    │
│    (sincronizado global)    │                                                │
└─────────────────────────────┴────────────────────────────────────────────────┘
```

### Por Qué Esta Arquitectura Es Fundamental

En un juego de supervivencia con **80 a 300 enemigos simultáneos** y 4 jugadores disparando hechizos a ~2 Hz cada uno, el servidor nunca procesa datos visuales. La matemática de hitboxes opera sobre vectores 3D puros y consultas espaciales, inmune a fluctuaciones de framerate o jitter de red del cliente.

La capa de renderizado predictivo del cliente significa que **el jugador siempre ve su propio hechizo impactar de forma instantánea**, incluso en conexiones de alta latencia, mientras el servidor resuelve el daño de forma autoritativa y a prueba de exploits.

### Presupuesto de Rendimiento y Renderizado Adaptativo

| Escenario | Optimización Aplicada |
|---|---|
| Hechizos propios del jugador | VFX completos a máxima fidelidad |
| Hechizos aliados (densidad moderada) | Conteo de partículas reducido, menor complejidad de malla |
| Hechizos aliados (alta densidad: 4 jugadores, disparo masivo) | Culling agresivo — emisores de partículas pausados, opacidad de malla reducida |
| 200+ enemigos en pantalla | Escalado LOD de mallas, tick rate de IA escalonado en servidor |

Este sistema garantiza que el juego sea jugable en **hardware móvil** sin degradar la experiencia de los jugadores de escritorio.

---

## 👥 Mecánicas Multijugador y Diseño de UX

### El Problema de la Pausa — Un Desafío de Diseño Único del Cooperativo

En los juegos de supervivencia para un solo jugador, pausar es trivial. En el cooperativo en tiempo real de *Wizard Survival*, la interacción individual con la UI de un jugador (abrir un cofre, subir de nivel una magia) no puede congelar el juego para sus compañeros. Esto requirió diseñar dos modos de interacción diferenciados:

#### UX Individual No Bloqueante

| Interacción | Implementación |
|---|---|
| **Selección al Subir de Nivel** | Overlay de UI personal con temporizador propio; los enemigos redistribuyen el aggro dinámicamente hacia los compañeros activos mientras el jugador está en el menú |
| **Apertura de Cofre Individual** | Ventana de recompensa por jugador; los demás continúan luchando sin interrupción |
| **Selección de Artefacto** | Menú local con temporizador; selección aleatoria automática si el tiempo expira |

#### Mecánicas de Consenso Sincronizado

| Mecánica | Umbral | Propósito |
|---|---|---|
| **Pausa Grupal** | ≥70% de los jugadores activos votan "Pausar" | Auditoría táctica de builds, revisión de configuración de rendimiento |
| **Cofre Obelisco** | Evento de mundo compartido | Todos los jugadores ven el cofre descender simultáneamente; las recompensas son por jugador pero la ventana de apertura está sincronizada |
| **Voto de Mapa** | Mayoría al reiniciar la run | Garantiza consenso grupal en dificultad y selección de mapa |

### Redistribución Dinámica del Aggro

Cuando un jugador abre cualquier menú, el sistema de pathfinding de IA enemiga en el servidor reevalúa las prioridades de objetivo. Los enemigos redirigen suavemente hacia el **jugador activo** más cercano, previniendo el abuso de menús como táctica de evasión y protegiendo a quienes legítimamente necesitan un momento para tomar una decisión de build.

---

## 🔥 Sistema de Magias y Combinaciones Legendarias

### Repertorio de Magias Base

| Magia | Tipo | Perfil de Daño | Mecánica Clave |
|---|---|---|---|
| **Disparo Mágico** | Proyectil | Moderado · Individual | Dispersión de triple disparo, alta velocidad |
| **Bola de Fuego** | Explosión en Área | Alto · AoE | Trayectoria en arco, radio de explosión escala con nivel |
| **Rayo Arcano** | Hitscan Penetrante | Muy Alto · Línea | Lanzamiento instantáneo, atraviesa todos los enemigos en el eje |
| **Tormenta de Rayos** | AoE Dirigido | Extremo · Múltiples golpes | Rayo en cadena, impacto estroboscópico |
| **Meteoro** | Global | Masivo · Punto singular | Anticipo prolongado, sacudida de cámara, marca de terreno |
| **Ventisca** | Control de Zona | Bajo · Sostenido | Estado de congelación, negación de área, sinergia con magias de agua |
| **Tsunami** | Desplazamiento | Medio · Empuje | Reencuadre de enemigos, cadena de knockback |
| **Tornado** | Ambiente | Medio · Sostenido | Atrae enemigos al vórtice, DoT acumulativo |

### Sistema de Fusiones Legendarias

Las combinaciones de hechizos desbloquean **Fusiones Definitivas** — habilidades cualitativamente distintas imposibles de replicar con las magias base.

#### ⚡ Láser Longinus *(Singularity Star)*
*Fusión: Meteoro (Asteroide) + Tormenta (Juicio)*

La habilidad más poderosa del juego, ejecutada en tres fases cinemáticas:

```
Fase 1 — Convocatoria (0.0s – 3.0s)
  · Anillos de designación celestial descienden desde la órbita
  · Disco de vacío negro se forma en el epicentro, absorbiendo la luz ambiental
  · El cielo se oscurece globalmente alrededor de la zona de impacto

Fase 2 — Colapso Pre-Impacto (3.0s – 4.0s)
  · Los anillos implotan hacia la singularidad a velocidad supersónica
  · El exterior del haz se engrosa hasta alcanzar masa crítica

Fase 3 — Cataclismo de Singularidad (4.0s – 10.5s)
  · Haz divino de 32m de diámetro se dispara desde arriba
  · Un agujero negro se manifiesta en el epicentro
  · Enemigos en un radio de 60m son arrastrados en vórtice centrípeto y obliterados
  · Domo de distorsión gravitacional se expande hasta 50m
```

#### 🌊 Gran Inundación *(Great Flood Wall)*
*Fusión: Tsunami (Canto de Sirena) + Tornado (Clima Extremo)*

Una muralla oceánica de 100m × 20m que emerge desde las espaldas del mago y avanza linealmente a 18 m/s durante 7 segundos, desplazando físicamente a cada enemigo en su trayectoria con simulación de knockback por cuerpo rígido.

#### 🔮 Genocidio *(Arcane Genocide)*
*Fusión: Rayo Arcano (Desintegración) + Disparo Mágico (Lanzamiento en Cadena)*

Siete sellos rúnicos orbitan al mago, disparando 32 ráfagas de láser secuenciales en 4.8 segundos. Cada impacto primario encadena automáticamente dos objetivos adicionales cercanos (radio de cadena: 12m), creando una cascada capaz de barrer corredores enteros simultáneamente.

---

## 💎 Profundidad de Contenido — Artefactos y Progresión

### Resumen del Sistema de Artefactos

*Wizard Survival* cuenta con **más de 199 artefactos únicos y funcionales** organizados en un sistema de rareza escalonado. Cada artefacto modifica una o más de las siguientes dimensiones:

- **Estadísticas Base** — Vida máxima, velocidad de movimiento, reducción de cooldown, probabilidad de crítico
- **Comportamiento de Hechizos** — Cantidad de proyectiles, radio de área, multiplicadores de daño, frecuencia de lanzamiento
- **Sinergias Pasivas** — Bonificaciones condicionales activadas por tipo de hechizo, arquetipo de enemigo o estado del jugador
- **Arquetipos de Build** — Habilitan estilos de juego específicos (Cañón de Cristal, Tanque Barrera, Mago Cadena, Mago del Tiempo)

### Tiers de Rareza

| Tier | Etiqueta | Tamaño del Pool | Adquisición |
|---|---|---|---|
| ⬜ Común | Estándar | ~60 | Cofres de oleada, drops del suelo |
| 🔵 Raro | Poco común | ~70 | Cofres Obelisco, recompensas de jefe |
| 🟣 Épico | Infrecuente | ~40 | Kills de jefe, cofres legendarios |
| 🔴 Legendario | Ultra-raro | ~20 | Obelisco de oleada tardía, logros desbloqueados |
| 🔴 Pasiva Especial | Comodín | ~9 | Baja probabilidad en todos los tiers |

### Artefactos Notables

| Artefacto | Tier | Efecto |
|---|---|---|
| **Lanza de Longinus** | Legendario | Aumenta la duración del haz Longinus y el radio de colapso |
| **Ouroboros** | Legendario | Al matar con hechizo: probabilidad de reiniciar el cooldown de una magia equipada aleatoria |
| **Excalibur** | Legendario | Aura cuerpo a cuerpo activa cuando la vida es < 30%, inflige daño masivo en área |
| **Registros Akáshicos** | Legendario | Acumula pasivamente una estadística de daño secundaria igual al total de kills de por vida |
| **Gaia** | Legendario | La regeneración de vida escala con el número de jugadores activos |
| **Necronomicón** | Épico | Todos los hechizos obtienen vampirismo proporcional a los enemigos alcanzados por lanzamiento |
| **Singularidad** | Épico | Reduce el cooldown de Longinus y añade atracción gravitacional a todos los hechizos AoE |
| **Puerta Dimensional** | Épico | Teletransporta proyectiles a través de obstáculos |
| **Circuito Espacio-Tiempo** | Especial | Ralentiza el tiempo local para los enemigos en un radio de 15 studs alrededor del mago |

### Profundidad del Buildcrafting

El espacio de interacción entre 199 artefactos y 8+ magias (cada una con múltiples caminos de evolución) crea un sistema de construcción de builds enorme y emergente. Los patrones de diseño intencionales incluyen:

- **Amplificación Elemental** — Los artefactos de Fuego/Rayo se acumulan de forma multiplicativa con las magias fundidas del mismo elemento
- **Bucles de Reacción en Cadena** — *Ouroboros* + *Genocidio* pueden crear ciclos de uptime casi permanente
- **Híbridos Tanque-Mago** — *Escudo de Maná* + *Escudo Orgánico* + *Segundo Corazón* convierte la supervivencia en una fuente de daño

---

## 🗺️ Mapas, Level Design y World Building

Tres mapas artesanales, cada uno con tres variantes de dificultad que alteran la iluminación, composición de enemigos, velocidad y atmósfera ambiental:

### 🌿 El Jardín Ancestral

El punto de entrada — geometría abierta y legible, permisiva en layout pero implacable en volumen de oleadas.

| Dificultad | Atmósfera | Modificador Ambiental |
|---|---|---|
| **Fácil** | Día · cielo despejado | Visibilidad estándar, parámetros enemigos de base |
| **Normal** | Hora dorada · sol rasante a 15° | Sombras largas que ocultan posiciones enemigas; incremento de velocidad medio |
| **Difícil** | Noche profunda · `#282C42` | Niebla volumétrica rasante + **ojos rojos brillantes como única señal de visibilidad enemiga**; el farol esmeralda del mago (`#78BE8C`, r=8m) es la única fuente de luz |

### 🏙️ La Ciudad Maldita

La geometría urbana introduce bloqueo de línea de visión, puntos de estrangulamiento y decisiones de ruta ausentes en el combate en campo abierto.

- **Arquitectura**: Asfalto agrietado, rascacielos en ruinas perdiéndose en la niebla superior, vehículos oxidados como cobertura parcial
- **Iluminación**: Frío desaturado de alto contraste, farolas parpadeantes a 2–4 Hz, reflejos en superficies húmedas
- **Comportamiento Enemigo**: Los *Crawlers* navegan por grietas bajas; los *Brutes* atraviesan obstáculos

### 🟡 Las Backrooms

El mapa más psicológicamente desorientador — la repetición de corredores infinitos crea confusión espacial por diseño deliberado.

- **Arquitectura**: Alfombra húmeda beige, papel tapiz descascarado, columnas repetitivas retrocediendo hasta el infinito
- **Iluminación**: Paneles fluorescentes a `#FFF2C6` con parpadeo secundario a 6 Hz; niebla negra a partir de los 45 studs
- **Intención de Diseño**: La claustrofobia es una mecánica — los hechizos AoE de largo alcance se vuelven críticos para la supervivencia cuando el flanqueo es imposible

### Arena de Jefe — El Sello Ritual

Cuando se desencadena una oleada de jefe, el mapa entra en **Modo de Confinamiento**:

```
Muro Perimetral (La Jaula)
  Prisma de energía dodecagonal · radio 50m · altura 25m
  Energía carmesí translúcida: #FF0000 · Alpha 0.5 · Emisión ×8
  Glifos arcanos animados desplazándose verticalmente por las caras del muro

Sello Central (Piso)
  Estrella rúnica de 12 puntas en el centro del mapa (0, 0, 0)
  Líneas de energía pulsan violeta/dorado en los umbrales de vida del jefe

Cofre Obelisco (Post-Muerte)
  Monolito de obsidiana de 4m desciende en una columna de luz dorada
  Se asienta con impacto físico en el centro del sello
```

**Secuencia de Muerte del Jefe:**
1. Flash de pantalla blanca → decaimiento suave en 1.5s
2. Todos los enemigos supervivientes se congelan 2 segundos (el servidor aplica invulnerabilidad temporal para prevenir destrucción prematura por AoE residual)
3. En t=2.0s: detonación masiva sincronizada — la física de ragdoll expulsa extremidades y fragmentos de partículas radialmente
4. Supresión de oleadas durante 5 segundos → descenso del Cofre Obelisco

---

## 🏰 Meta-Juego, Lobby y Economía

### Economía Dual

| Moneda | Fuente | Uso |
|---|---|---|
| 💰 **Monedas** | Rendimiento en run, bonificaciones por completar oleadas | Compras en tiendas NPC, modificadores temporales de run |
| 💎 **Diamantes** | Puntuación al final de run, hitos de logros | Desbloqueos permanentes, mejoras del lobby, expansión del pool de artefactos |

### Estructura del Lobby

```
LOBBY
├── 🧙 NPC Tomo de Hechizos   — Desbloquear magias base, ver requisitos de evolución
├── 🏺 Vendedor de Artefactos  — Explorar pool de artefactos, comprar expansiones permanentes
├── 📊 Tablero de Estadísticas — Récords personales, clasificación global
├── 🗺️ Selector de Mapa        — Elegir mapa destino y dificultad
└── 🎒 Vista Previa de Build   — Revisar mejoras permanentes y desbloqueos obtenidos
```

### Progresión Persistente

El avance del jugador persiste entre sesiones mediante **DataStore**:
- Totales de monedas acumuladas de por vida
- Pool de magias desbloqueadas
- Progreso de colección de artefactos
- Métricas de rendimiento personal (oleada máxima alcanzada, mayor daño en run, etc.)

---

## 🎨 UI/UX y Polish Visual

La interfaz de *Wizard Survival* fue construida mediante **calibración manual de cada propiedad de UI dentro de Roblox Studio** — un rechazo deliberado a los layouts autogenerados en favor de fidelidad visual ajustada a mano.

### Filosofía de Diseño

Cada pantalla pasó por afinamiento propiedad por propiedad:
- **Escalado de canvas** configurado para renderizado consistente en PC, tablet y móvil
- **Animaciones Tween** en cada elemento interactivo (hover de botones, deslizamiento de paneles, fundidos)
- **Tipografía** consistente en todas las pantallas usando la fuente `Fondamento` para unidad temática
- **Sistema de color** derivado directamente de la paleta de hechizos del juego, creando continuidad entre jugabilidad y menús

### Pantallas Clave

| Pantalla | Aspectos Técnicos Destacados |
|---|---|
| **Selección de Magia (Subida de Nivel)** | Overlay a pantalla completa, backdrop difuminado, cartas de hechizos con easing elástico, efectos de brillo por rareza |
| **HUD (En Partida)** | Barra de vida, contador de oleadas, íconos de magia con animación de arco de cooldown — todo en un único `ScreenGui` con Z-index gestionado manualmente |
| **Menú de Pausa** | UI de votación grupal con porcentaje de aprobación en tiempo real, sin bloquear el estado del servidor |
| **Selector de Artefactos** | Grilla desplazable con código de color por rareza, sistema de tooltip hover, animación de confirmación de selección |
| **Estadísticas del Lobby** | Tarjetas de stats impulsadas por DataStore con contadores numéricos animados |

### Sistema de Layout Responsivo

| Factor de Forma | Adaptación |
|---|---|
| Escritorio (16:9) | Paneles completos, HUD expandido |
| Tablet (4:3) | Paneles escalados, tamaño de botones optimizado para táctil |
| Móvil (9:16 vertical) | HUD contraído, posicionamiento de botones en zona de pulgar |

---

## ⚙️ Pipeline de Desarrollo y Notas de Producción

### Historial de Iteración

*Wizard Survival* fue construido desde cero a lo largo de varios meses de iteración continua, comenzando por los sistemas fundamentales y expandiéndose progresivamente:

```
Mes 1 — Fundamentos
├── Sistema de spawn de enemigos (GeneradorEnemigos) con escalado por tiempo
├── Primera magia: Disparo Mágico (prototipo de proyectil server-side)
├── Ciclo de juego base: temporizador de oleadas, contador de kills, máquina de estados de run
└── Layout inicial del mapa: El Jardín (prototipo Fácil)

Mes 2 — Refactorización de Arquitectura de Hechizos
├── División de autoridad Cliente-Servidor implementada en todas las magias
│   ├── Servidor: matemática de hitboxes, daño, aplicación de estados
│   └── Cliente: VFX, mallas, emisores de partículas
├── Bola de Fuego: assets visuales obtenidos de Toolbox, completamente re-arquitectados
│   en sistema de movimiento autoritativo server-side con desacoplamiento
│   visual del cliente
└── Rayo Arcano: primera implementación de hitscan penetrante

Mes 3 — Expansión de Contenido
├── Sistema de artefactos (199+ ítems) con persistencia por DataStore
├── Sistema de fusiones legendarias: Longinus, Gran Inundación, Genocidio
├── Framework de encuentros con jefes: Sello Ritual, Cofre Obelisco, secuencia de muerte
├── Mapas La Ciudad Maldita y Las Backrooms
└── Sistemas cooperativos: redistribución de aggro, voto-pausa, cofre grupal

Continuo — Polish, Balance, Meta del Lobby
├── Calibración de UI/UX (ajuste manual propiedad por propiedad en todas las pantallas)
├── Árboles de diálogo de NPCs del lobby y sistemas de economía
├── Optimización de rendimiento: culling, LOD, presupuestos adaptativos de partículas
└── Marketing: campaña de miniaturas personalizadas (arte por pickle_remolacha)
```

### Estrategia del Pipeline de Assets

Un desafío profesional recurrente en el desarrollo de Roblox es la distinción entre **adquisición de assets crudos** e **integración arquitectónica**. *Wizard Survival* utiliza un pipeline deliberado:

1. **Obtención** — Mallas y sistemas de partículas adquiridos desde Toolbox o creados en Studio como herramientas de prototipado rápido
2. **Auditoría** — Cada asset se analiza en cuanto a características de rendimiento y hooks de comportamiento
3. **Re-Arquitectura** — Los assets se desacoplan completamente de sus scripts originales y se reintegran a la arquitectura autoritativa del servidor
4. **Validación** — La matemática de hitboxes server-side se implementa de forma independiente usando álgebra espacial; el asset visual y la lógica de gameplay son responsabilidades separadas en todo momento

> *Ejemplo: La malla y el sistema de partículas de la Bola de Fuego fueron obtenidos externamente, pero el modelo de daño, el radio de la hitbox y la física de trayectoria están implementados íntegramente en Luau server-side mediante álgebra vectorial — la malla es cosmética, la matemática es autoritativa.*

### Stack Tecnológico

| Herramienta | Rol |
|---|---|
| **Roblox Studio** | Entorno de desarrollo principal, autoría de escenas, testing integrado |
| **Luau** | Scripts de servidor, controladores de cliente, librerías de módulos, lógica de UI |
| **Roblox DataStore API** | Persistencia del jugador: monedas, desbloqueos, estadísticas de runs |
| **RemoteEvent / RemoteFunction** | Frontera de comunicación Cliente ↔ Servidor |
| **Blender 3D** | Creación de assets, tráiler cinemático (scripting Python `bpy`) |
| **Python** | Herramientas externas: procesamiento masivo de assets, scripts de automatización |
| **Git** | Control de versiones |
| **Workflows asistidos por IA** | Resolución de problemas algorítmicos, aceleración de scripts, validación de matemática edge-case — siempre bajo revisión humana, testing y refactorización |

---

## 📈 Métricas de Lanzamiento, Créditos y Roadmap

### Rendimiento en Lanzamiento

| Métrica | Valor |
|---|---|
| **Visitas Únicas** | 1.200+ en la ventana de lanzamiento |
| **Canal de Marketing** | Campaña de miniaturas personalizadas — dirección de arte por `pickle_remolacha` |
| **Presencia en Leaderboards** | Posicionamiento activo en tablas de clasificación globales por conteo de oleadas |
| **Plataforma** | Roblox (PC + Móvil) |
| **Señal de Retención** | Visitas orgánicas de retorno de jugadores persiguiendo completaciones en dificultad alta |

### Créditos

| Rol | Crédito |
|---|---|
| **Dirección de Juego, Arquitectura y Level Design** | Director del Proyecto — meses de desarrollo y iteración continua en solitario |
| **Arte de Miniaturas, Imágenes y Branding Visual** | **pickle_remolacha** |

### Roadmap

#### Corto Plazo (v1.1 – v1.2)

- [ ] **Nuevo Mapa: El Abismo** — entorno de cueva subterránea con diseño de nivel vertical y spawners en techo
- [ ] **Roster de Jefes Expandido** — 3 nuevos arquetipos con patrones de mecánicas telegráficas únicas
- [ ] **UI del Árbol de Evolución de Magias** — grimorio visual mostrando caminos de combinación completos
- [ ] **Lobby Completo de 4 Jugadores** — expansión de cooperativo de 2 a 4 jugadores

#### Mediano Plazo (v2.0)

- [ ] **Modo Raids** — runs de desafío de alta dificultad diseñados para escuadrones coordinados
- [ ] **Crafteo de Artefactos** — sistema de combinación para potenciar variantes de artefactos
- [ ] **Soporte de Idiomas** — UI bilingüe ES / EN con selector regional

#### Visión a Largo Plazo

- [ ] **Eventos de Temporada** — mapas por tiempo limitado y drops de artefactos exclusivos
- [ ] **Temporadas de Leaderboard** — ciclos competitivos con reseteos periódicos y recompensas exclusivas para los mejores
- [ ] **Constructor de Dificultad Personalizada** — pila de modificadores configurada por el jugador para runs de desafío autoimpuesto

---

<div align="center">

*Construido desde cero. Refinado a través de la iteración.*

**[▶ Jugar en Roblox](#) · [💬 Discord](#) · [📋 Roadmap](#)**

</div>
