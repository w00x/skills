---
name: add-collider
description: Agrega, corrige y valida colliders y superposición por profundidad en mapas Tiled TMX de juegos 2D top-down. Usar al incorporar árboles, arbustos, piedras, troncos, utilería, agua, puentes, pilares, cuerdas, casas, techos, carteles u otros objetos; al corregir penetración visual o bloqueos incorrectos; o cuando el jugador deba pasar parcialmente delante, detrás o debajo de un objeto.
---

# Add Collider

Diseñar colisión y profundidad visual como un solo sistema. Evitar que una caja correcta físicamente produzca una superposición visual incorrecta.

## Flujo obligatorio

1. Inspeccionar el TMX, tilesets, generadores del mapa, conversores y código runtime que consume `Collisions`, capas visuales y profundidad.
2. Identificar la fuente de verdad. Si el TMX es generado, editar primero el generador y regenerar el TMX; no dejar un cambio manual que se pierda.
3. Medir el tile, el sprite del jugador y su collider real: ancho, alto, offsets y línea de los pies.
4. Clasificar cada objeto y aplicar el patrón correspondiente de [patrones de colisión y profundidad](references/patterns.md). Leer esa referencia completa antes de editar.
5. Crear colliders semánticos en una object layer `Collisions`, con nombre estable y propiedad `material`.
6. Incorporar el objeto a una capa o sprite ordenado por la coordenada Y de su base. Dividir elementos grandes en bandas cuando una sola línea de profundidad no alcance.
7. Regenerar artefactos derivados y ejecutar conversión, compilación, pruebas y lint disponibles.
8. Verificar visualmente los cuatro acercamientos: norte, sur, este y oeste. Confirmar tanto el punto de detención como qué parte del objeto cubre al jugador.

## Reglas de decisión

- Colocar colliders en el contacto con el suelo, no alrededor de toda la silueta, para árboles, arbustos, piedras, jarros, sacos, canastos y postes.
- Usar el borde inferior del collider del jugador como línea de profundidad. No ordenar por el centro del sprite.
- Compensar colliders laterales cuando el sprite sea más ancho que el collider de los pies; comprobar que la silueta no penetre visualmente.
- Mantener continuas las barreras compuestas. No dejar huecos entre pilares, cuerdas o segmentos que permitan atravesarlas en diagonal.
- Separar visualmente copa/base, techo/fachada o segmentos del puente cuando deban alternar delante y detrás del jugador.
- No dar collider de suelo a elementos aéreos como letreros colgantes; darles una profundidad que permita caminar por debajo mientras cubren al jugador.
- Mantener transitables puertas, corredores, puentes y rutas principales. Medir el espacio libre contra el collider real del jugador.
- Preservar cambios ajenos y editar únicamente los objetos en alcance.

## Criterio de terminado

No considerar terminado por pasar pruebas estructurales solamente. Exigir:

- TMX válido y artefactos derivados sincronizados.
- Collider ubicado en base/pies y con material semántico.
- Profundidad natural sin cortar sprites ni ocultar de más al jugador.
- Sin penetración lateral, huecos diagonales ni rutas bloqueadas.
- Pruebas automatizadas de geometría y una comprobación visual proporcional al cambio.

