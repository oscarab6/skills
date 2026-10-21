# Diseño estratégico y diagnóstico del modelo

Lee este archivo cuando el usuario esté definiendo el sistema completo o varios módulos, cuando pregunte cómo dividir un sistema, cómo integrar módulos o un servicio externo, o cuando pida evaluar o puntuar un modelo de dominio. La implementación (clases base, `Either`, TypeORM) sigue estando en `patterns.md`.

Contenido:
1. Lenguaje ubicuo en la práctica
2. Subdominios: core, soporte y genérico
3. Bounded contexts
4. Context map: los nueve patrones
5. Capa anticorrupción (ACL) con Either
6. Integración entre módulos con eventos
7. Event Storming
8. Diagnóstico y puntuación del modelo

---

## 1. Lenguaje ubicuo en la práctica

- Sale de hablar con expertos del dominio (o de los requisitos), no de un glosario escrito al final. Extrae sustantivos (candidatos a entidades y valores) y verbos (candidatos a métodos del agregado y eventos).
- Si un concepto cuesta nombrarlo, probablemente el modelo está mal: la dificultad para nombrar es una señal de diseño.
- Sospecha de nombres técnicos: `Manager`, `Helper`, `Processor`, `Utils`, `Data`, `Info`, `setStatus(n)`. Pregunta qué término usaría el negocio (`ClaimAdjudicator`, `policy.underwrite()`, `order.confirm()`).
- Organiza el código por concepto del dominio (`users/`, `orders/`, `billing/`), no por tipo técnico (`controllers/`, `services/`).
- Mantén un glosario corto (término → significado → en qué contexto) cuando el dominio tenga más de unos pocos conceptos. Si el término cambia, cambia el código.
- El mismo término puede significar cosas distintas en contextos distintos, y eso es correcto (ver bounded contexts).

## 2. Subdominios: core, soporte y genérico

| Tipo | Qué es | Inversión |
|------|--------|-----------|
| **Core** | Lo que diferencia al negocio; donde están las reglas complejas | DDD táctico completo: agregados ricos, valores, eventos, tests de dominio |
| **Soporte** | Necesario pero no diferenciador | Modelo sencillo; DDD táctico solo si hay reglas reales |
| **Genérico** | Problema resuelto (autenticación, email, pagos) | Librería o servicio externo detrás de un adaptador/ACL |

- No apliques DDD táctico a un CRUD sin reglas: es sobreingeniería, y conviene decirlo.
- Identifica explícitamente cuál es el core: es donde más compensa invertir en el modelo.
- Lo que hoy es core puede volverse genérico; revisa la clasificación si el problema cambia.

## 3. Bounded contexts

- Un bounded context es una **frontera de modelo y de lenguaje**: dentro, cada término tiene un único significado. `Product` en Catálogo (descripción, fotos) no es el `Product` de Facturación (precio, impuestos).
- **No es un microservicio.** Al empezar, cada contexto es un módulo de primer nivel en `src/` dentro del mismo monolito (`src/catalog/`, `src/billing/`), con sus capas `domain/application/infrastructure`.
- Un contexto no importa el dominio de otro. Se comunican por ids, eventos o una API/servicio de aplicación publicada.
- Evita un único modelo para todo: un `User` con campos de facturación, envío y marketing acaba siendo contradictorio. Cada contexto tiene su versión con lo que necesita.
- Para sistemas existentes, primero dibuja lo que hay (aunque sea una "gran bola de barro") y luego propone fronteras objetivo.

## 4. Context map: los nueve patrones

| Patrón | Relación | Cuándo usarlo |
|--------|----------|---------------|
| **Partnership** | Dos contextos evolucionan coordinados, ninguno manda | Equipos/módulos muy acoplados que se planifican juntos |
| **Shared Kernel** | Comparten un trozo pequeño de modelo | Por ejemplo, `src/shared/domain` (clases base). Mantenlo mínimo: todo lo que entra acopla a todos |
| **Customer/Supplier** | El proveedor (upstream) atiende necesidades del cliente (downstream) | Un módulo consume datos de otro que puede adaptar su API |
| **Conformist** | El downstream adopta el modelo del upstream tal cual | No compensa traducir (modelo externo bueno o integración trivial) |
| **Anticorruption Layer (ACL)** | El downstream traduce el modelo ajeno al suyo | Siempre que integres un sistema externo o legacy en el core |
| **Open Host Service** | El upstream publica una API estable para muchos consumidores | Un módulo que usan varios otros |
| **Published Language** | Formato de intercambio documentado (JSON schema, eventos) | Junto con Open Host Service |
| **Separate Ways** | Sin integración; cada uno resuelve por su cuenta | Cuando integrar cuesta más que duplicar |
| **Big Ball of Mud** | Zona sin modelo claro | Para señalarla y aislarla tras una ACL, no para imitarla |

Cuando el usuario pida un context map, entrégalo como lista de relaciones (`Pedidos → Facturación: Customer/Supplier, integración por evento OrderConfirmedEvent`) o como diagrama.

## 5. Capa anticorrupción (ACL) con Either

El puerto se define en el dominio con el lenguaje propio; el adaptador en infraestructura traduce el modelo externo y convierte sus fallos en `Either`.

```ts
// src/orders/domain/PaymentGateway.ts  — puerto, en lenguaje del dominio
import { Either } from '../../shared/domain/Either';
import { UnexpectedError } from '../../shared/domain/DomainError';
import { PaymentRejectedError } from './PaymentErrors';

export interface PaymentGateway {
    charge(orderId: string, amountCents: number): Promise<Either<PaymentRejectedError | UnexpectedError, { paymentId: string }>>;
}
```

```ts
// src/orders/domain/PaymentErrors.ts
import { DomainError } from '../../shared/domain/DomainError';

export class PaymentRejectedError implements DomainError {
    private _msg: string;
    private constructor(reason: string) { this._msg = `Payment rejected: ${reason}`; }
    get errorMessage() { return this._msg; }
    public static create(reason: string): PaymentRejectedError {
        return new PaymentRejectedError(reason);
    }
}
```

```ts
// src/orders/infrastructure/payments/AcmePayClient.ts  — modelo AJENO (lo que devuelve el proveedor)
export interface AcmeChargeResponse {
    txn_ref: string | null;
    status_code: 0 | 1 | 2;          // 0 = ok, 1 = rechazado, 2 = error interno
    decline_reason?: string;
}

export interface AcmePayClient {
    createCharge(req: { reference: string; amt: number; ccy: 'EUR' }): Promise<AcmeChargeResponse>;
}
```

```ts
// src/orders/infrastructure/payments/AcmePaymentGateway.ts  — la ACL: traduce y protege el dominio
import { Either, left, right } from '../../../shared/domain/Either';
import { UnexpectedError } from '../../../shared/domain/DomainError';
import { PaymentGateway } from '../../domain/PaymentGateway';
import { PaymentRejectedError } from '../../domain/PaymentErrors';
import { AcmePayClient } from './AcmePayClient';

export class AcmePaymentGateway implements PaymentGateway {
    constructor(private readonly client: AcmePayClient) {}

    async charge(orderId: string, amountCents: number):
        Promise<Either<PaymentRejectedError | UnexpectedError, { paymentId: string }>> {
        try {
            const res = await this.client.createCharge({ reference: orderId, amt: amountCents, ccy: 'EUR' });

            // Ningún código de estado ni nombre del proveedor sale de aquí
            if (res.status_code === 0 && res.txn_ref !== null) return right({ paymentId: res.txn_ref });
            if (res.status_code === 1) return left(PaymentRejectedError.create(res.decline_reason ?? 'unknown'));
            return left(UnexpectedError.create(res));
        } catch (err) {
            return left(UnexpectedError.create(err));
        }
    }
}
```

Señales de que falta una ACL: tipos o nombres del proveedor (`status_code`, `txn_ref`) dentro de `domain/` o `application/`, o casos de uso que comparan códigos numéricos externos.

## 6. Integración entre módulos con eventos

- **Dentro de un agregado**: consistencia inmediata (una transacción).
- **Entre agregados o módulos**: consistencia eventual mediante eventos del dominio. El caso de uso guarda y luego hace `DomainEvents.dispatch(...)`; el otro módulo registra un manejador que llama a **su propio** caso de uso.
- El manejador solo usa los datos primitivos del evento (ids, importes), nunca clases del dominio del otro módulo.
- Publica solo lo que al negocio le importa: no todo cambio de estado merece un evento.

```ts
// src/billing/infrastructure/subscriptions.ts  — billing reacciona a un hecho de orders
import { DomainEvents } from '../../shared/domain/DomainEvents';
import { OrderConfirmedEvent } from '../../orders/domain/OrderEvents';
import { IssueInvoice } from '../application/IssueInvoice';

export function registerBillingSubscriptions(issueInvoice: IssueInvoice) {
    DomainEvents.register<OrderConfirmedEvent>(OrderConfirmedEvent.name, async (event) => {
        const result = await issueInvoice.execute({ orderId: event.orderId, totalCents: event.totalCents });
        if (result.isLeft()) console.error('Invoice not issued:', result.value.errorMessage);
    });
}
```

Importar la *clase del evento* de otro módulo es aceptable (es su lenguaje publicado); importar sus entidades, valores o repositorios no.

## 7. Event Storming

Para descubrir el dominio a partir de requisitos o con expertos:
1. **Eventos** (naranja): todo lo que pasa, en pasado (`PedidoConfirmado`, `PagoRechazado`). Ordénalos en el tiempo.
2. **Comandos** (azul): qué acción provoca cada evento (`ConfirmarPedido`). Serán casos de uso.
3. **Agregados** (amarillo): qué objeto acepta o rechaza el comando según sus reglas. Agrupa comandos y eventos alrededor de él.
4. **Políticas** (lila): "cuando ocurre X, hacer Y" (`Cuando PedidoConfirmado → EmitirFactura`). Serán manejadores de eventos.
5. **Fronteras**: donde cambia el vocabulario o el experto, hay otro bounded context.

Entrega el resultado como tabla (evento | comando | agregado | política) o diagrama, y de ahí deriva módulos, agregados y casos de uso.

## 8. Diagnóstico y puntuación del modelo

Úsalo cuando el usuario pida revisar, evaluar o puntuar un modelo o un proyecto. Responde cada pregunta con sí/no y la evidencia concreta (archivo, clase, método).

| # | Pregunta | Si la respuesta es no |
|---|----------|-----------------------|
| 1 | ¿Un experto del dominio entendería los nombres de clases, métodos y eventos? | Renombrar al lenguaje ubicuo |
| 2 | ¿Están definidas las fronteras entre módulos/contextos y ninguno importa el dominio de otro? | Dibujar el context map; separar modelos |
| 3 | ¿Los agregados son pequeños (una raíz y lo mínimo) y referencian a otros por id? | Partir agregados; referencias por id |
| 4 | ¿Entidades y valores tienen comportamiento y protegen sus reglas (no son bolsas de datos)? | Mover reglas de los servicios al dominio |
| 5 | ¿Los errores de negocio se declaran con `Either` en las firmas, sin `throw` para casos esperables? | Migrar a `left(...)` con errores de dominio |
| 6 | ¿La comunicación entre agregados/módulos usa eventos del dominio? | Introducir eventos y manejadores |
| 7 | ¿Cada integración externa y la persistencia están tras un puerto con su adaptador/ACL (TypeORM fuera del dominio)? | Añadir repositorio o ACL |

**Puntuación (sobre 10)**: 1 punto por cada fila cumplida (máx. 7) más hasta 3 por profundidad:
- +1 si el subdominio core tiene un modelo rico (no CRUD).
- +1 si todas las invariantes viven en agregados y valores, no en casos de uso.
- +1 si el lenguaje es coherente entre requisitos, código y tests.

**Bandas**:
- **9–10**: nombres legibles por un experto, fronteras explícitas, agregados pequeños con comportamiento, `Either` coherente, eventos entre agregados.
- **7–8**: buena base con uno o dos huecos concretos.
- **5–6**: algo de lenguaje del dominio pero fronteras que se filtran o modelo anémico.
- **≤4**: nombres técnicos, un único modelo para todo, lógica dispersa en servicios.

Informa la puntuación, las filas que fallan y, para cada una, el cambio concreto que la arreglaría, empezando por lo que más sube la nota.
