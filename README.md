# ddd-typescript

Skill para agentes de IA (Claude Code, Claude, Cursor, Codex…) que diseña, implementa y revisa código **TypeScript con Domain-Driven Design**:

- **Bloques tácticos con clases base**:
  - `UniqueEntityID`
  - `Entity<T>` y `ValueObject<T>` con `props`
  - `AggregateRoot<T>` con eventos del dominio
  - `Repository<T>` abstracto
- **Errores con `Either`** (`left`/`right`) en lugar de excepciones: cada operación declara en su firma qué puede fallar, y eso incluye las búsquedas de los repositorios.
- **Persistencia con TypeORM**:
  - las entidades persistentes están separadas del dominio;
  - el mapeo se hace con `toPersistence`/`fromPersistence`;
  - el agregado se guarda entero, con sus entidades hijas.
- **Diseño estratégico**:
  - subdominios;
  - bounded contexts y los nueve patrones de context map;
  - capa anticorrupción;
  - integración entre módulos con eventos;
  - Event Storming.
- **Diagnóstico del modelo**: 7 preguntas y una puntuación sobre 10 para revisar un dominio existente.

Si tu proyecto ya usa otra convención (excepciones de dominio, `Result`, `neverthrow`, `fp-ts`, otro ORM), la skill la respeta y adapta los patrones.

## Instalación

Con la CLI de [skills.sh](https://www.skills.sh):

```bash
npx skills add <usuario>/<repo> --skill ddd-typescript
```

Manualmente en Claude Code: copia la carpeta `ddd-typescript/` en `~/.claude/skills/` (para todos tus proyectos) o en `.claude/skills/` dentro de un proyecto.

En la app de Claude: comprime la carpeta `ddd-typescript/` en un `.zip` y súbela como skill personalizada.

## Qué incluye

```
ddd-typescript/
├── SKILL.md                      # Cuándo se activa, convenciones y checklist de revisión
└── references/
    ├── patterns.md               # Clases base + ejemplo completo (pedidos con líneas)
    └── strategic-design.md       # Diseño estratégico, ACL y diagnóstico con puntuación
```

El ejemplo de `patterns.md` modela un pedido (`Order`) con líneas (`OrderLine`), valores `Money` y `Quantity`, eventos `OrderCreatedEvent` y `OrderConfirmedEvent`, repositorio en memoria y TypeORM (relación uno-a-muchos con borrado de líneas huérfanas), casos de uso y tests. Todo el código está comprobado: compila con `strict: true` y sus tests pasan, incluidos los de integración con SQLite en memoria.

## Ejemplos de uso

- "Modela el dominio de una biblioteca con préstamos y reservas"
- "Hazme el repositorio TypeORM de este agregado"
- "¿Esto debería ser una entidad o un value object?"
- "Revisa mi agregado y dame una puntuación"
- "Tengo que integrar la API de pagos de un proveedor, ¿cómo la aíslo de mi dominio?"

## Dependencias del código generado

`uuid`, `shallow-equal-object`, `typeorm`, `reflect-metadata` y el driver de tu base de datos (por ejemplo, `better-sqlite3` o `pg`). En `tsconfig.json` hacen falta `experimentalDecorators` y `emitDecoratorMetadata`.

## Licencia

MIT
