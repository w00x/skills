# Patrones de colisión y profundidad para Tiled TMX

## Modelo base

Tratar la posición de los pies como coordenada de orden:

```text
depth = worldDepth(playerBody.bottom)
objectDepth = worldDepth(objectGroundContactY)
```

En el proyecto de referencia:

- grid: `32 × 32 px`
- sprite del jugador: `32 × 32 px`
- collider del jugador: `12 × 8 px`
- offset del collider: `x=10`, `y=23`

Leer estos valores del proyecto actual antes de reutilizarlos. Si coinciden, conservar las medidas probadas de esta referencia.

## Objetos apoyados en el suelo

### Árboles

- Usar collider central de tronco, no `64 × 64` de la copa.
- Para árboles de `64 × 64`, usar como punto de partida probado `16 × 14 px` en la base central.
- Separar visualmente copa y base o reconstruir el árbol como sprite con profundidad en `treeY + treeHeight`.
- Permitir que el cuerpo superior del jugador quede bajo las hojas sin atravesar el tronco.

### Arbustos, piedras, troncos y utilería

- Ajustar el collider al área inferior que toca el suelo.
- Dibujar el objeto en una capa de props ordenada por `tileBottomY`.
- Aplicar el mismo patrón a sacos, canastos, barriles, pozos, jarros y letreros apoyados en el suelo.
- No usar una caja completa solo porque el gráfico llena el tile.

### Agua, orificios y límites de terreno

- Usar colisión de área completa cuando toda la superficie sea intransitable.
- Fusionar tiles contiguos en rectángulos cuando sea posible.
- Excluir explícitamente pasos transitables como la cubierta de un puente.

## Puentes con cuerdas y pilares

Modelar componentes por separado: cuatro pilares y dos barreras laterales. Mantener libre el corredor central.

Para un puente probado de `64 × 96 px`, jugador de `32 px` y collider de pies de `12 px`:

```text
ancho de cada barrera lateral = 20 px
corredor central = 24 px
alto de collider de pilar = 18 px
collider de pilares superiores = últimos 18 px de la primera fila de 32 px
cuerdas = desde y + 32 hasta el comienzo de los pilares inferiores
collider de pilares inferiores = últimos 18 px del puente
```

La anchura lateral compensa la diferencia entre el sprite de 32 px y los pies de 12 px. Ajustarla si cambia el jugador.

Nombres recomendados:

```text
bridge_pillar_top_left
bridge_pillar_top_right
bridge_pillar_bottom_left
bridge_pillar_bottom_right
bridge_rope_left
bridge_rope_right
```

Asignar `material=bridge`.

Para la superposición:

- No dejar pilares o cuerdas solamente en una tile layer plana.
- Reconstruir las barandas como sprites dinámicos.
- Dividir filas de 32 px en bandas de 16 px si una fila completa tapa al jugador durante demasiado recorrido.
- En cada tile, conservar la franja centrada del poste: 16 px para pilares y 8 px para cuerda. No enmascarar desde el borde exterior si el dibujo está centrado.
- Ordenar cada banda por su borde inferior.

## Casas y techos

- Separar el edificio en bandas horizontales de un tile, o más finas si el arte lo exige.
- Ordenar cada banda por su borde inferior; no reconstruir todo el edificio como un único sprite con línea de profundidad en el cimiento.
- Para permitir que el jugador muestre media figura detrás del borde superior, dejar solo `16 px` de solapamiento en un grid de 32 px y comenzar allí el collider largo del edificio.
- Extender ese collider hasta la base y por todo el ancho para impedir entrar profundamente o acceder desde los costados.
- Ajustar los 16 px según el arte y la línea de los pies si las dimensiones difieren.
- Mantener puertas y accesos explícitamente transitables si el diseño requiere entrar a la casa.

Asignar `material=building` y nombres estables como `inn_wall` o `house_03_body`.

## Letreros

- Letrero apoyado: collider pequeño en su base y orden por los pies.
- Letrero colgante: sin collider de suelo.
- Colocar el colgante en una capa overlay y asignar una línea de profundidad inferior para que el jugador camine debajo y el letrero lo cubra.
- Ubicarlo al costado de la casa, fuera de su huella, si así lo indica el diseño.

## Esquema TMX recomendado

```xml
<objectgroup name="Collisions" visible="0">
  <object name="tree_001" type="collision"
          x="100" y="132" width="16" height="14">
    <properties>
      <property name="material" value="vegetation"/>
    </properties>
  </object>
</objectgroup>
```

Usar capas dedicadas cuando correspondan:

```text
tree_base
tree_canopy
depth_props
buildings
building_overlays
DepthObjects
Collisions
```

Adaptar nombres al runtime existente; no inventar capas que el juego no consume sin actualizar también el cargador.

## Validación mínima

Probar de forma automatizada:

- existencia de `Collisions` y propiedades `material`
- geometría exacta de objetos críticos
- collider del jugador limitado a sus piernas
- corredor central del puente mayor o igual al ancho de los pies
- ausencia de collider en elementos aéreos
- continuidad de rutas principales
- sincronización entre TMX y formato convertido

Probar visualmente:

- acercamiento desde los cuatro lados
- detención en la base, no en la copa o parte alta
- cuerpo superior cubierto y piernas visibles cuando corresponde
- pilares completos, sin máscaras cortadas
- ninguna tabla, techo o banda tapa al jugador después de sobrepasar su línea de apoyo
- ausencia de penetración visual aunque el collider físico sea más estrecho que el sprite

