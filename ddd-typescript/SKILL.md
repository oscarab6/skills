---
name: ddd-typescript
description: Diseña, implementa, revisa y refactoriza código TypeScript con Domain-Driven Design (DDD) usando Entity/ValueObject con props, UniqueEntityID, AggregateRoot con eventos del dominio, gestión de errores con Either (left/right) en lugar de excepciones, y repositorios con TypeORM. Incluye diseño estratégico (bounded contexts, context map, capa anticorrupción) y un diagnóstico con puntuación del modelo. Úsala siempre que el usuario mencione DDD, entidades, value objects, agregados, eventos del dominio, repositorios, casos de uso, Either, TypeORM, lenguaje ubicuo o arquitectura hexagonal en un proyecto TypeScript/Node.js, aunque no diga "DDD" explícitamente (p. ej. "cómo modelo el dominio de pedidos", "dónde pongo esta validación", "hazme el repositorio de Order", "revisa mi agregado").
---

# DDD en TypeScript con Either y TypeORM

Guía para modelar dominios con DDD en TypeScript: las reglas de negocio viven en el dominio y se expresan con el lenguaje ubicuo. Los errores del dominio se declaran en la firma con `Either` en lugar de lanzarse, y la persistencia se hace con TypeORM detrás de repositorios.

Responde en el idioma del usuario. El código usa las clases base de esta skill (`Entity`, `ValueObject`, `AggregateRoot`, `UniqueEntityID`, `Either`, `DomainEvents`, `Repository`); los nombres del dominio siguen la convención que ya use el proyecto.

**Antes de escribir código, lee `references/patterns.md`**: contiene las clases base y un ejemplo completo: valores, agregado con entidad hija, eventos, repositorio en memoria y TypeORM, casos de uso y tests. Si el proyecto ya tiene sus propias clases base, usa las suyas y no las redefinas.

**Para diseño estratégico o evaluación, lee `references/strategic-design.md`**: subdominios, bounded contexts, los nueve patrones de context map, capa anticorrupción con `Either`, integración entre módulos con eventos, Event Storming y un diagnóstico de 7 preguntas con puntuación sobre 10. Léelo cuando el usuario quiera dividir un sistema en módulos, integrar un servicio externo, partir de un enunciado, o pida revisar, evaluar o puntuar un modelo.

Si el proyecto del usuario ya sigue otra convención (excepciones de dominio, `Result`, `neverthrow`, `fp-ts`, otro ORM), respétala y adapta los patrones; esta skill no impone `Either` sobre un código base existente.

## Cómo trabajar

0. **Si es un sistema nuevo o con varios módulos**, empieza por lo estratégico (`references/strategic-design.md`): qué subdominio es el core, qué bounded contexts hay y cómo se relacionan. Para un único agregado o una clase concreta, salta este paso.
1. **Entiende el dominio**: lenguaje ubicuo, invariantes (reglas de negocio) y qué debe ser consistente a la vez. Si falta algo crítico para decidir los límites de un agregado, pregunta; si no, asume lo razonable y dilo.
2. **Clasifica cada concepto**:
   - ¿Tiene identidad que persiste en el tiempo? → entidad.
   - ¿Se define por su valor y se valida? → valor.
   - ¿Es la entrada a un grupo de entidades y protege sus reglas? → aggregate root.
3. **Modela con las clases base** (sección siguiente).
4. **Declara errores con `Either`** en toda operación que pueda fallar por motivos de negocio.
5. **Persiste con un repositorio**: abstracción en el dominio, implementación TypeORM en infraestructura que traduce dominio ↔ entidad persistente.
6. **Orquesta en casos de uso** (servicios de aplicación) y prueba el dominio con el repositorio en memoria.

## Bloques tácticos

### `UniqueEntityID`
Identificador de entidad: `string` UUID v4 (paquete `uuid`). Constructor privado; `static create(plainId?)` genera uno nuevo si no se pasa y lo valida si se pasa. Igualdad por valor del id.

### Entidades — `Entity<T>`
- Las propiedades se declaran en una interfaz (`OrderLineProps`) y se guardan en `props`.
- El id es opcional en la construcción: **sin id = entidad nueva; con id = viene de persistencia**.
- Igualdad por identificador.
- Constructor `private`/`protected` y métodos factoría estáticos.

### Valores — `ValueObject<T>`
- Sin identidad, inmutables (`Object.freeze(props)`) y con igualdad estructural (`shallowEqual` del paquete `shallow-equal-object`).
- Contienen la validación: `static create(props): Either<InvalidXError, X>`, de modo que no pueda existir un valor inválido.
- Sustituye primitivos por valores en cuanto haya una regla (`Money`, `Quantity`, `Email`...). Prefiere valores a entidades siempre que no haga falta identidad.

### Agregados — `AggregateRoot<T>`
Especializa `Entity<T>` y gestiona eventos: `addDomainEvent(event)` (protegido) y `pullDomainEvents()`.
- Es el único punto de entrada al grupo de entidades y el responsable de las invariantes. Las entidades hijas (`OrderLine`) solo se modifican a través de la raíz (`order.addLine(...)`), y la raíz expone las colecciones como `ReadonlyArray`.
- Comportamiento con nombres del negocio (`order.confirm()`), no setters genéricos. Cada método que puede violar una regla devuelve `Either<Error, ...>`.
- Mantén los agregados **pequeños** y referencia a otros agregados **por id**, no por objeto.
- Un caso de uso modifica **un solo agregado**. Si necesita tocar varios, usa un evento del dominio.
- Separa **creación** de **reconstrucción**: `create(...)` aplica las reglas de nacimiento y emite el evento de creación; `restore(props, id)` reconstruye desde persistencia sin emitir eventos (nada nuevo ha ocurrido).

### Eventos del dominio
- `interface DomainEvent { readonly occurredOn: Date }`. Cada evento es una clase con nombre en pasado (`OrderConfirmedEvent`) y solo datos primitivos (ids, importes).
- `DomainEvents` es un servicio estático: `register(NombreEvento.name, handler)` y `dispatch(events)`, que despacha por `event.constructor.name`.
- Los eventos se extraen y se despachan **después** de persistir con éxito.

### Gestión de errores — `Either<L, R>`
- `Left` = error del dominio, `Right` = resultado correcto. Se construyen con `left(...)` y `right(...)` y se comprueban con `isLeft()`/`isRight()`, que son guardas de tipo: tras `isRight()`, `result.value` es el valor.
- Los errores son clases que implementan `DomainError { errorMessage?; error? }`, con constructor privado y `static create(...)`. Hay uno por situación de negocio (`EmptyOrderError`, `OrderNotEditableError`), más `UnexpectedError` para fallos técnicos y `EntityNotFoundError`.
- Declara todos los errores posibles en la firma con una unión: `Either<OrderNotEditableError | InvalidQuantityError, void>`. El compilador muestra así qué puede fallar, cosa que no hacen las excepciones ni un `Result` sin tipo de error.
- Propaga sin envolver: `if (r.isLeft()) return left(r.value)`.
- No lances excepciones para errores de negocio. Reserva `throw` para errores de programación o datos corruptos. En los casos de uso, captura los fallos de infraestructura con `try/catch` y conviértelos en `left(UnexpectedError.create(err))`.
- Para distinguir errores en tiempo de ejecución usa `instanceof`. El campo privado `_msg` de cada clase de error evita que TypeScript los considere estructuralmente intercambiables.

### Repositorios
- Clase **abstracta** genérica `Repository<T extends Entity<any>>` con `save`, `findById` y `remove`, más clases abstractas por agregado que añaden consultas con lenguaje del negocio (`OrderRepository.findOpenByCustomer`).
- **Las búsquedas devuelven `Either`, no lanzan**: `findById(id): Promise<Either<EntityNotFoundError, T>>`, y lo mismo para cualquier consulta que busque un único elemento.
- `save` devuelve `Promise<void>` y `remove` devuelve `Promise<boolean>`. Las excepciones técnicas del ORM no se modelan en el repositorio: las captura el caso de uso.
- Repositorio en memoria (con `Map`) para tests.
- Implementación TypeORM en infraestructura (`OrderRepositoryTypeORM`), con dos métodos privados: `toPersistence(agg)` y `fromPersistence(row)`. Las entidades TypeORM (`OrderEntity`, `OrderLineEntity`, con decoradores) **no** son las entidades de dominio y nunca salen del repositorio.
- El repositorio guarda y devuelve **el agregado completo**. Las hijas van con `@OneToMany(..., { cascade: true, eager: true })` y `orphanedRowAction: 'delete'` en el `@ManyToOne`, para que las líneas eliminadas del agregado se borren de la tabla.
- En `fromPersistence` reconstruye con `restore` y recrea los valores con su `create`. Como los datos ya estaban validados, un `left` ahí indica datos corruptos: lanza error.
- Inyecta el `DataSource` por constructor en lugar de importar uno global; así se puede probar con SQLite en memoria.

### Servicios
- **Servicios del dominio**: lógica de negocio que no encaja en una entidad o valor (p. ej. calcular un precio con reglas de varios agregados). No tienen estado.
- **Casos de uso** (servicios de aplicación):
  - Reciben los repositorios por constructor.
  - Siguen siempre la misma secuencia: cargar el agregado, invocar su comportamiento, guardar y después `DomainEvents.dispatch(agg.pullDomainEvents())`.
  - Devuelven `Promise<Either<Errores, Dto>>`.
  - No contienen reglas de negocio.

## Estructura de carpetas

```
src/
├── shared/
│   ├── domain/          # UniqueEntityID, Entity, ValueObject, AggregateRoot,
│   │                    # DomainEvent, DomainEvents, Either, DomainError, Repository
│   └── infrastructure/  # data-source.ts
├── <modulo>/            # p. ej. orders/, billing/ (un bounded context por módulo)
│   ├── domain/          # Order.ts, OrderLine.ts, Money.ts, eventos, errores, OrderRepository.ts
│   ├── application/     # CreateOrder.ts, ConfirmOrder.ts (casos de uso)
│   └── infrastructure/
│       └── persistence/ # OrderEntity.ts (TypeORM), MemoryOrderRepository.ts, OrderRepositoryTypeORM.ts
└── index.ts             # dataSource.initialize() y cableado de dependencias
```

Regla de dependencias: `infrastructure → application → domain`. El dominio no importa `typeorm`, el `DataSource`, Express ni nada de infraestructura.

## Configuración

- Paquetes: `uuid`, `shallow-equal-object`, `typeorm`, `reflect-metadata` y un driver (p. ej. `better-sqlite3`, `pg`).
- `tsconfig.json`:
  - `experimentalDecorators` y `emitDecoratorMetadata` a `true`, que TypeORM necesita.
  - Mantén `strict` y usa `!` en las columnas de las entidades TypeORM (`name!: string`) en vez de desactivar `strictNullChecks`.
- Añade cada entidad TypeORM nueva al array `entities` del `DataSource`.

## Revisión de código

Señala cada problema con su porqué, empezando por lo que rompe invariantes o acopla capas:
- Modelo anémico: setters públicos y lógica fuera de la entidad.
- Primitivos donde debería haber un valor; validación fuera del `create` del valor.
- `throw` para errores de negocio en lugar de `left` (incluido "entidad no encontrada" en los repositorios), o firmas `Either` que no declaran todos los errores posibles.
- Valores mutables o con constructor público.
- Entidades hijas modificables desde fuera de la raíz; colecciones mutables expuestas.
- Agregados que referencian otros agregados por objeto, o casos de uso que modifican varios.
- Eventos emitidos al reconstruir desde persistencia.
- Entidades TypeORM usadas en el dominio o en los casos de uso; el `DataSource` importado fuera de infraestructura.
- Nombres técnicos (`Manager`, `Helper`, `Processor`, `Utils`, `setStatus(n)`) en lugar del lenguaje ubicuo.
- Un módulo que importa entidades, valores o repositorios de otro; tipos o códigos de un servicio externo dentro del dominio (falta una capa anticorrupción).

Si el usuario pide **evaluar o puntuar** el modelo, aplica además el diagnóstico de 7 preguntas de `references/strategic-design.md`. Da la nota sobre 10 con las filas que fallan y cómo arreglar cada una.

## Formato de salida

- Diseño: glosario breve del lenguaje ubicuo; entidades, valores y agregados con sus invariantes, errores y eventos; después, el código.
- Código: archivos completos con su ruta según la estructura de carpetas, y al menos un test del agregado principal con el repositorio en memoria.
- Explica brevemente las decisiones no obvias: por qué algo es valor y no entidad, por qué ese límite de agregado, qué errores declara cada `Either`. Relaciónalas con el concepto de DDD (Evans, Vernon) cuando ayude a aprender.
