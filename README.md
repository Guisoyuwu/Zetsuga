# Zangetsu Mod (Fabric, Minecraft Java 1.21.11)

## ⚠️ Estado real del proyecto (léelo primero)
- **Código escrito pero NO compilado ni probado en el juego.** El entorno donde se generó no tenía acceso a internet ni Gradle, por lo que no se pudo ejecutar `./gradlew clean build`, no hay `.jar` y no se probó nada en Minecraft.
- Minecraft 1.21.11 cambió varias APIs (sistema de permisos, `Identifier`, `teleportTo`, `hurtServer`…). Se escribió con mappings oficiales de Mojang según lo mejor conocido; es **probable que haya 1–5 errores de compilación puntuales** (ver "Puntos a verificar").
- El ZIP incluye `gradle-wrapper.properties`, pero no el binario `gradle-wrapper.jar` ni los scripts `gradlew`/`gradlew.bat` (no se pudieron descargar desde este entorno). El workflow de GitHub Actions instala Gradle 9.2.1 y genera el Wrapper automáticamente antes de compilar.

## Compilar
```
gradle wrapper --gradle-version 9.2.1 --distribution-type bin
./gradlew clean build                    # JAR en build/libs/zangetsu-mod-1.0.0.jar
```
Versiones fijadas para esta corrección: Minecraft 1.21.11, Java 21, Fabric Loom 1.14.10 y Gradle 9.2.1. Las versiones de Loader y Fabric API incluidas en `gradle.properties` se mantienen; conviene contrastarlas con Fabric Develop si se actualiza el proyecto.

## Instalación
Servidor **y** clientes necesitan Fabric Loader + Fabric API + este JAR (el HUD del timer y el modelo de Zangetsu son del lado cliente).

## Comandos (OP nivel 2+)
| Comando | Función |
|---|---|
| `/timer 20s \| 20m \| 20hs` | Temporizador global visible arriba a la derecha para todos |
| `/timer stop` | Detiene y oculta |
| `/timerplayer <jugador> <20s\|20m\|20hs>` | Temporizador individual; al terminar envía `(Tiempo Acabado)` |
| `/tagvanish` `/untagvanish` | Oculta/muestra solo el nombre sobre la cabeza |
| `/vanish` `/unvanish` | Invisibilidad completa (ver limitaciones) |

Objetos (`/give @s zangetsu_mod:zangetsu`, `zangetsu_mod:a_casa_pa`, `zangetsu_mod:tp_0_0`; también en la pestaña Combate del creativo).

## Funcionamiento
- **Timer global:** estado en el servidor, sincronizado cada segundo por paquete propio; persiste en `<mundo>/zangetsu_mod_state.json`; sigue si quien lo inició se desconecta; sin mensaje al terminar.
- **Timer individual:** sigue corriendo si el jugador se desconecta (o se pausa con `playerTimerPausesWhileOffline`); si termina estando offline, recibe `(Tiempo Acabado)` al volver a entrar.
- **A Casa Pa:** solo al golpear a otro jugador (`AttackEntityCallback`, servidor) → lo expulsa con el mensaje configurable.
- **TP 0 0:** solo al golpear → teletransporta a X0 Z0 en la dimensión del golpeado, buscando altura segura (suelo sólido, 2 bloques libres, sin lava/fuego/cactus/magma/etc.). Si (0,0) no tiene sitio seguro prueba columnas vecinas hasta 16 bloques; si no hay, avisa y no hace nada.
- **Zangetsu:** clic derecho = Getsuga Tensho (proyectil azul, 30 de daño por defecto, cooldown 15 s).

## Configuración: `config/zangetsu_mod.json`
`zangetsuCooldownSeconds`, `getsugaDamage`, `getsugaMaxRange`, `getsugaSpeedBlocksPerTick`, `casaPaKickMessage`, `persistVanish`, `maxTimerSeconds`, `playerTimerPausesWhileOffline`, `notifyPlayerTimerOnJoin`.

## Decisiones técnicas y limitaciones (sin ocultar nada)
1. **Getsuga Tensho sin entidad custom:** el proyectil es una simulación del servidor (raycast de bloques y entidades cada tick) dibujada con partículas vanilla (polvo azul, llama de alma, chispas, media luna). Ventaja: todos lo ven igual, no se puede duplicar con paquetes. Costo: no es una entidad con modelo propio.
2. **"Exactamente 30 de daño":** se usa daño mágico indirecto (ignora armadura) y se resetea la invulnerabilidad, pero efectos como Resistencia, Absorción o encantamientos de protección mágica pueden reducirlo. Los jugadores en creativo son inmunes.
3. **Vanish:** oculta entidad y TAB, simula salida/entrada, suprime los mensajes reales de join/leave y persiste si `persistVanish=true`. **No** oculta: `/list`, selectores (`@a`), el chat que escriba, sonidos propios, ni mods/plugins que consulten la lista de jugadores del servidor. Es una limitación real del enfoque sin plugins.
4. **/tagvanish** usa un equipo de scoreboard (`zm_tagvanish`) con `nameTagVisibility=NEVER`. Un jugador solo puede estar en un equipo: si ya está en otro, el comando avisa y no actúa.
5. **Modelo de Zangetsu:** modelo 3D (16 cuboides rotados 45° + textura 64×64 generada por código) con silueta de la imagen de referencia: hoja ancha negra con filo plateado, mango blanco envuelto y cinta. Es una aproximación en el límite de lo que permite el formato de modelos de bloque/ítem; no es una conversión fotográfica de la imagen. Funciona sin OptiFine. **No lo vi renderizado en el juego**; puede requerir ajustar el `display` (posición en mano) en `models/item/zangetsu.json`.
6. **A Casa Pa y TP 0 0** usan la textura del palo vanilla con brillo de encantamiento.

## Puntos a verificar al compilar (APIs que cambiaron en 1.21.11)
- `Commands.hasPermission(Commands.LEVEL_GAMEMASTERS)` en `ModCommands` (sistema de permisos nuevo).
- `ServerPlayer.teleportTo(level, x, y, z, Set<Relative>, yRot, xRot, boolean)` en `ZangetsuMod`.
- `LivingEntity.hurtServer(...)`, `Item.use(...)` devolviendo `InteractionResult`, `Item.Properties#sword(...)`.
- Nombre del campo `entityMap` (ChunkMap) y `entity` (TrackedEntity) en los mixins.
- `HudElementRegistry` / `GuiGraphics` en el cliente.

## Pruebas pendientes (nada de esto está verificado todavía)
Compilación, registro de comandos/objetos, timers con 2 jugadores, persistencia tras reinicio, vanish/tagvanish, palos, colisión y daño de Getsuga, cooldown en servidor, aspecto del modelo.
