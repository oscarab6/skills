# Plantillas DDD en TypeScript (Either + TypeORM)

Si el proyecto ya tiene sus propias clases base, usa las del proyecto y adapta los ejemplos.

Contenido:
1. Shared kernel: UniqueEntityID, Entity, ValueObject, Either, DomainError, DomainEvent, DomainEvents, AggregateRoot, Repository
2. Ejemplo `orders`: errores, valores (Money, Quantity), entidad hija (OrderLine), eventos, agregado (Order)
3. Repositorio: abstracto, en memoria y TypeORM (con relación uno-a-muchos)
4. Casos de uso
5. Adaptador HTTP (opcional)
6. Arranque
7. Tests (dominio, caso de uso, integración TypeORM)

Paquetes: `uuid`, `shallow-equal-object`, `typeorm`, `reflect-metadata`, `better-sqlite3` (o el driver que uses).

---

## 1. Shared kernel

```ts
// src/shared/domain/UniqueEntityID.ts
import { v4 as uuidv4, validate } from 'uuid';

export class UniqueEntityID {
    readonly id: string;

    private constructor(plainId?: string) {
        this.id = plainId ?? uuidv4();
    }

    get value(): string { return this.id; }

    equals(other: UniqueEntityID): boolean {
        return this.id === other.id;
    }

    static create(plainId?: string): UniqueEntityID {
        // Un id mal formado es un error técnico (datos corruptos), no de negocio: se lanza
        if (plainId !== undefined && !validate(plainId))
            throw new Error('Invalid UUID v4: ' + plainId);
        return new UniqueEntityID(plainId);
    }
}
```

```ts
// src/shared/domain/Entity.ts
import { UniqueEntityID } from './UniqueEntityID';

export abstract class Entity<T> {
    readonly id: UniqueEntityID;
    protected props: T;

    // Sin id -> entidad nueva. Con id -> reconstruida desde persistencia.
    protected constructor(props: T, id?: UniqueEntityID) {
        this.id = id ?? UniqueEntityID.create();
        this.props = props;
    }

    public equals(object?: Entity<T>): boolean {
        if (object == null) return false;
        if (this === object) return true;
        return this.id.equals(object.id);
    }
}
```

```ts
// src/shared/domain/ValueObject.ts
import { shallowEqual } from 'shallow-equal-object';

export abstract class ValueObject<T extends object> {
    protected readonly props: Readonly<T>;

    protected constructor(props: T) {
        this.props = Object.freeze({ ...props });
    }

    public equals(vo?: ValueObject<T>): boolean {
        if (vo === null || vo === undefined) return false;
        if (vo.props === undefined) return false;
        return shallowEqual(this.props, vo.props);
    }
}
```

```ts
// src/shared/domain/Either.ts
interface EitherProps<L, A> {
    value: L | A;
    isLeft(): this is Left<L, A>;
    isRight(): this is Right<L, A>;
}

export class Left<L, A> implements EitherProps<L, A> {
    readonly value: L;
    constructor(value: L) { this.value = value; }
    isLeft(): this is Left<L, A> { return true; }
    isRight(): this is Right<L, A> { return false; }
}

export class Right<L, A> implements EitherProps<L, A> {
    readonly value: A;
    constructor(value: A) { this.value = value; }
    isLeft(): this is Left<L, A> { return false; }
    isRight(): this is Right<L, A> { return true; }
}

export type Either<L, A> = Left<L, A> | Right<L, A>;

export function left<L, A>(l: L): Either<L, A> {
    return new Left(l);
}

export function right<L, A>(a: A): Either<L, A> {
    return new Right(a);
}
```

```ts
// src/shared/domain/DomainError.ts
export interface DomainError {
    errorMessage?: string;
    error?: any;
}

// Fallos técnicos (BD caída, excepción inesperada) convertidos a Either en los casos de uso
export class UnexpectedError implements DomainError {
    private _error: any;
    private constructor(error: any) { this._error = error; }

    get errorMessage() { return 'An unexpected error occurred'; }
    get error() { return this._error; }

    public static create(err: any): UnexpectedError {
        return new UnexpectedError(err);
    }
}

export class EntityNotFoundError implements DomainError {
    private _msg: string;
    private constructor(entity: string, id: string) {
        this._msg = `${entity} not found: ${id}`;
    }
    get errorMessage() { return this._msg; }

    public static create(entity: string, id: string): EntityNotFoundError {
        return new EntityNotFoundError(entity, id);
    }
}
```

```ts
// src/shared/domain/DomainEvent.ts
export interface DomainEvent {
    readonly occurredOn: Date;
}
```

```ts
// src/shared/domain/DomainEvents.ts
import { DomainEvent } from './DomainEvent';

export class DomainEvents {
    private static handlers: Map<string, ((event: DomainEvent) => void)[]> = new Map();

    static register<T extends DomainEvent>(eventName: string, handler: (event: T) => void) {
        if (!this.handlers.has(eventName)) {
            this.handlers.set(eventName, []);
        }
        this.handlers.get(eventName)!.push(handler as any);
    }

    // Se despacha por el nombre de la clase del evento: register(OrderConfirmedEvent.name, ...)
    static dispatch(events: DomainEvent[]) {
        for (const event of events) {
            const handlers = this.handlers.get(event.constructor.name) || [];
            handlers.forEach(handler => handler(event));
        }
    }

    // Útil en tests para no arrastrar manejadores entre casos
    static clearHandlers() {
        this.handlers.clear();
    }
}
```

```ts
// src/shared/domain/AggregateRoot.ts
import { Entity } from './Entity';
import { UniqueEntityID } from './UniqueEntityID';
import { DomainEvent } from './DomainEvent';

export abstract class AggregateRoot<T> extends Entity<T> {
    private domainEvents: DomainEvent[] = [];

    protected constructor(props: T, id?: UniqueEntityID) {
        super(props, id);
    }

    protected addDomainEvent(event: DomainEvent): void {
        this.domainEvents.push(event);
    }

    public pullDomainEvents(): DomainEvent[] {
        const events = [...this.domainEvents];
        this.domainEvents = [];
        return events;
    }
}
```

```ts
// src/shared/domain/Repository.ts
import { Entity } from './Entity';
import { Either } from './Either';
import { EntityNotFoundError } from './DomainError';

// "No encontrado" es un error del dominio: se devuelve con Either, no se lanza.
export abstract class Repository<T extends Entity<any>> {
    abstract save(obj: T): Promise<void>;
    abstract findById(id: string): Promise<Either<EntityNotFoundError, T>>;
    abstract remove(id: string): Promise<boolean>;
}
```

---

## 2. Ejemplo: módulo `orders`

Invariantes del agregado `Order`:
- Solo se puede modificar mientras está en `draft`.
- No se puede confirmar sin líneas.
- Todas las líneas usan la moneda del pedido.
- Un mismo producto aparece en una sola línea (se acumula la cantidad), hasta un máximo de 99 unidades.

### Errores

```ts
// src/orders/domain/OrderErrors.ts
import { DomainError } from '../../shared/domain/DomainError';

export class InvalidQuantityError implements DomainError {
    private _msg: string;
    private constructor(value: number) { this._msg = `Quantity must be an integer between 1 and 99: ${value}`; }
    get errorMessage() { return this._msg; }
    public static create(value: number): InvalidQuantityError { return new InvalidQuantityError(value); }
}

export class InvalidMoneyError implements DomainError {
    private _msg: string;
    private constructor(cents: number) { this._msg = `Amount must be a non-negative integer of cents: ${cents}`; }
    get errorMessage() { return this._msg; }
    public static create(cents: number): InvalidMoneyError { return new InvalidMoneyError(cents); }
}

export class OrderNotEditableError implements DomainError {
    private _msg: string;
    private constructor(orderId: string) { this._msg = `Order ${orderId} is no longer a draft`; }
    get errorMessage() { return this._msg; }
    public static create(orderId: string): OrderNotEditableError { return new OrderNotEditableError(orderId); }
}

export class EmptyOrderError implements DomainError {
    private _msg: string;
    private constructor(orderId: string) { this._msg = `Order ${orderId} has no lines`; }
    get errorMessage() { return this._msg; }
    public static create(orderId: string): EmptyOrderError { return new EmptyOrderError(orderId); }
}
```

### Valores

```ts
// src/orders/domain/Money.ts
import { ValueObject } from '../../shared/domain/ValueObject';
import { Either, left, right } from '../../shared/domain/Either';
import { InvalidMoneyError } from './OrderErrors';

export type Currency = 'EUR' | 'USD';

interface MoneyProps {
    cents: number;       // céntimos enteros: evita errores de coma flotante
    currency: Currency;
}

export class Money extends ValueObject<MoneyProps> {
    get cents(): number { return this.props.cents; }
    get currency(): Currency { return this.props.currency; }

    private constructor(props: MoneyProps) {
        super(props);
    }

    static create(props: MoneyProps): Either<InvalidMoneyError, Money> {
        if (!Number.isInteger(props.cents) || props.cents < 0)
            return left(InvalidMoneyError.create(props.cents));
        return right(new Money(props));
    }

    static zero(currency: Currency): Money {
        return new Money({ cents: 0, currency });
    }

    // Sumar monedas distintas es un error de programación: el agregado garantiza la misma moneda
    add(other: Money): Money {
        if (other.currency !== this.currency) throw new Error('Currency mismatch');
        return new Money({ cents: this.cents + other.cents, currency: this.currency });
    }

    times(factor: number): Money {
        return new Money({ cents: Math.round(this.cents * factor), currency: this.currency });
    }
}
```

```ts
// src/orders/domain/Quantity.ts
import { ValueObject } from '../../shared/domain/ValueObject';
import { Either, left, right } from '../../shared/domain/Either';
import { InvalidQuantityError } from './OrderErrors';

interface QuantityProps {
    value: number;
}

export class Quantity extends ValueObject<QuantityProps> {
    static readonly MAX = 99;

    get value(): number { return this.props.value; }

    private constructor(props: QuantityProps) {
        super(props);
    }

    static create(value: number): Either<InvalidQuantityError, Quantity> {
        if (!Number.isInteger(value) || value < 1 || value > Quantity.MAX)
            return left(InvalidQuantityError.create(value));
        return right(new Quantity({ value }));
    }

    // Inmutable: devuelve un valor nuevo (y valida el resultado)
    plus(other: Quantity): Either<InvalidQuantityError, Quantity> {
        return Quantity.create(this.value + other.value);
    }
}
```

### Entidad hija

```ts
// src/orders/domain/OrderLine.ts
import { Entity } from '../../shared/domain/Entity';
import { UniqueEntityID } from '../../shared/domain/UniqueEntityID';
import { Money } from './Money';
import { Quantity } from './Quantity';

export interface OrderLineProps {
    productId: string;      // referencia a otro agregado (Product) por id
    quantity: Quantity;
    unitPrice: Money;
}

// Solo el agregado Order crea y modifica líneas
export class OrderLine extends Entity<OrderLineProps> {
    get productId(): string { return this.props.productId; }
    get quantity(): Quantity { return this.props.quantity; }
    get unitPrice(): Money { return this.props.unitPrice; }

    private constructor(props: OrderLineProps, id?: UniqueEntityID) {
        super(props, id);
    }

    static create(props: OrderLineProps, id?: UniqueEntityID): OrderLine {
        return new OrderLine(props, id);
    }

    subtotal(): Money {
        return this.props.unitPrice.times(this.props.quantity.value);
    }

    // Uso interno del agregado
    replaceQuantity(quantity: Quantity): void {
        this.props.quantity = quantity;
    }
}
```

### Eventos

```ts
// src/orders/domain/OrderEvents.ts
import { DomainEvent } from '../../shared/domain/DomainEvent';

export class OrderCreatedEvent implements DomainEvent {
    readonly occurredOn = new Date();
    constructor(public readonly orderId: string, public readonly customerId: string) {}
}

export class OrderConfirmedEvent implements DomainEvent {
    readonly occurredOn = new Date();
    constructor(
        public readonly orderId: string,
        public readonly totalCents: number,
        public readonly currency: string,
    ) {}
}
```

### Agregado

```ts
// src/orders/domain/Order.ts
import { AggregateRoot } from '../../shared/domain/AggregateRoot';
import { UniqueEntityID } from '../../shared/domain/UniqueEntityID';
import { Either, left, right } from '../../shared/domain/Either';
import { Currency, Money } from './Money';
import { Quantity } from './Quantity';
import { OrderLine } from './OrderLine';
import { OrderCreatedEvent, OrderConfirmedEvent } from './OrderEvents';
import { EmptyOrderError, InvalidMoneyError, InvalidQuantityError, OrderNotEditableError } from './OrderErrors';

export type OrderStatus = 'draft' | 'confirmed' | 'cancelled';

export interface OrderProps {
    customerId: string;     // referencia a otro agregado por id
    currency: Currency;
    status: OrderStatus;
    lines: OrderLine[];
}

export class Order extends AggregateRoot<OrderProps> {

    get customerId(): string { return this.props.customerId; }
    get currency(): Currency { return this.props.currency; }
    get status(): OrderStatus { return this.props.status; }
    get lines(): ReadonlyArray<OrderLine> { return [...this.props.lines]; }

    private constructor(props: OrderProps, id?: UniqueEntityID) {
        super(props, id);
    }

    /** Creación: reglas de nacimiento + evento. */
    static create(props: { customerId: string; currency: Currency }): Order {
        const order = new Order({ ...props, status: 'draft', lines: [] });
        order.addDomainEvent(new OrderCreatedEvent(order.id.id, props.customerId));
        return order;
    }

    /** Reconstrucción desde persistencia: sin eventos (no ha ocurrido nada nuevo). */
    static restore(props: OrderProps, id: UniqueEntityID): Order {
        return new Order(props, id);
    }

    addLine(productId: string, quantity: number, unitPriceCents: number):
        Either<OrderNotEditableError | InvalidQuantityError | InvalidMoneyError, void> {

        if (this.props.status !== 'draft') return left(OrderNotEditableError.create(this.id.id));

        const qtyOrError = Quantity.create(quantity);
        if (qtyOrError.isLeft()) return left(qtyOrError.value);

        // La moneda la fija el pedido: no puede haber líneas en otra moneda
        const priceOrError = Money.create({ cents: unitPriceCents, currency: this.props.currency });
        if (priceOrError.isLeft()) return left(priceOrError.value);

        const existing = this.props.lines.find(l => l.productId === productId);
        if (existing) {
            const merged = existing.quantity.plus(qtyOrError.value);
            if (merged.isLeft()) return left(merged.value);
            existing.replaceQuantity(merged.value);
        } else {
            this.props.lines.push(OrderLine.create({
                productId, quantity: qtyOrError.value, unitPrice: priceOrError.value,
            }));
        }
        return right(undefined);
    }

    removeLine(productId: string): Either<OrderNotEditableError, void> {
        if (this.props.status !== 'draft') return left(OrderNotEditableError.create(this.id.id));
        this.props.lines = this.props.lines.filter(l => l.productId !== productId);
        return right(undefined);
    }

    confirm(): Either<OrderNotEditableError | EmptyOrderError, void> {
        if (this.props.status !== 'draft') return left(OrderNotEditableError.create(this.id.id));
        if (this.props.lines.length === 0) return left(EmptyOrderError.create(this.id.id));

        this.props.status = 'confirmed';
        const total = this.total();
        this.addDomainEvent(new OrderConfirmedEvent(this.id.id, total.cents, total.currency));
        return right(undefined);
    }

    total(): Money {
        return this.props.lines.reduce((acc, l) => acc.add(l.subtotal()), Money.zero(this.props.currency));
    }
}
```

---

## 3. Repositorio de `Order`

```ts
// src/orders/domain/OrderRepository.ts
import { Repository } from '../../shared/domain/Repository';
import { Order } from './Order';

export abstract class OrderRepository extends Repository<Order> {
    // Consulta con lenguaje del negocio
    abstract findOpenByCustomer(customerId: string): Promise<Order[]>;
}
```

```ts
// src/orders/infrastructure/persistence/MemoryOrderRepository.ts
import { OrderRepository } from '../../domain/OrderRepository';
import { Order } from '../../domain/Order';
import { Either, left, right } from '../../../shared/domain/Either';
import { EntityNotFoundError } from '../../../shared/domain/DomainError';

export class MemoryOrderRepository extends OrderRepository {
    private orders: Map<string, Order> = new Map();

    async save(obj: Order): Promise<void> {
        this.orders.set(obj.id.id, obj);
    }

    async findById(id: string): Promise<Either<EntityNotFoundError, Order>> {
        const order = this.orders.get(id);
        if (order === undefined) return left(EntityNotFoundError.create('Order', id));
        return right(order);
    }

    async findOpenByCustomer(customerId: string): Promise<Order[]> {
        return [...this.orders.values()].filter(o => o.customerId === customerId && o.status === 'draft');
    }

    async remove(id: string): Promise<boolean> {
        return this.orders.delete(id);
    }
}
```

### TypeORM

```ts
// src/orders/infrastructure/persistence/OrderEntity.ts
// Entidades PERSISTENTES (TypeORM). No son las entidades del dominio.
import { Entity, PrimaryColumn, Column, OneToMany, ManyToOne } from 'typeorm';

@Entity('orders')
export class OrderEntity {
    @PrimaryColumn('varchar')
    id!: string;

    @Column('varchar')
    customerId!: string;

    @Column('varchar')
    currency!: string;

    @Column('varchar')
    status!: string;

    // cascade: guarda las líneas con el pedido; eager: las carga siempre (el agregado va entero)
    @OneToMany(() => OrderLineEntity, line => line.order, { cascade: true, eager: true })
    lines!: OrderLineEntity[];
}

@Entity('order_lines')
export class OrderLineEntity {
    @PrimaryColumn('varchar')
    id!: string;

    @Column('varchar')
    productId!: string;

    @Column('integer')
    quantity!: number;

    @Column('integer')
    unitPriceCents!: number;

    // orphanedRowAction: borra las líneas que el agregado ha eliminado
    @ManyToOne(() => OrderEntity, order => order.lines, { onDelete: 'CASCADE', orphanedRowAction: 'delete' })
    order!: OrderEntity;
}
```

```ts
// src/orders/infrastructure/persistence/OrderRepositoryTypeORM.ts
import { DataSource } from 'typeorm';
import { UniqueEntityID } from '../../../shared/domain/UniqueEntityID';
import { Either, left, right } from '../../../shared/domain/Either';
import { EntityNotFoundError } from '../../../shared/domain/DomainError';
import { OrderRepository } from '../../domain/OrderRepository';
import { Order, OrderStatus } from '../../domain/Order';
import { OrderLine } from '../../domain/OrderLine';
import { Currency, Money } from '../../domain/Money';
import { Quantity } from '../../domain/Quantity';
import { OrderEntity, OrderLineEntity } from './OrderEntity';

export class OrderRepositoryTypeORM extends OrderRepository {

    // DataSource inyectado: en tests se usa SQLite en memoria
    constructor(private readonly dataSource: DataSource) {
        super();
    }

    // dominio -> objetos persistentes
    private toPersistence(order: Order): OrderEntity {
        const entity = new OrderEntity();
        entity.id = order.id.id;
        entity.customerId = order.customerId;
        entity.currency = order.currency;
        entity.status = order.status;
        entity.lines = order.lines.map(line => {
            const row = new OrderLineEntity();
            row.id = line.id.id;
            row.productId = line.productId;
            row.quantity = line.quantity.value;
            row.unitPriceCents = line.unitPrice.cents;
            return row;
        });
        return entity;
    }

    // objetos persistentes -> agregado (restore: sin eventos)
    private fromPersistence(entity: OrderEntity): Order {
        const currency = entity.currency as Currency;
        const lines = entity.lines.map(row => {
            const quantity = Quantity.create(row.quantity);
            const price = Money.create({ cents: row.unitPriceCents, currency });
            // Datos ya validados al guardarlos: un left aquí es corrupción, no un caso de negocio
            if (quantity.isLeft() || price.isLeft())
                throw new Error('Corrupted order line ' + row.id);
            return OrderLine.create(
                { productId: row.productId, quantity: quantity.value, unitPrice: price.value },
                UniqueEntityID.create(row.id),
            );
        });
        return Order.restore(
            { customerId: entity.customerId, currency, status: entity.status as OrderStatus, lines },
            UniqueEntityID.create(entity.id),
        );
    }

    async save(obj: Order): Promise<void> {
        await this.dataSource.getRepository(OrderEntity).save(this.toPersistence(obj));
    }

    async findById(id: string): Promise<Either<EntityNotFoundError, Order>> {
        const entity = await this.dataSource.getRepository(OrderEntity).findOneBy({ id });
        if (entity == null) return left(EntityNotFoundError.create('Order', id));
        return right(this.fromPersistence(entity));
    }

    async findOpenByCustomer(customerId: string): Promise<Order[]> {
        const entities = await this.dataSource.getRepository(OrderEntity).findBy({ customerId, status: 'draft' });
        return entities.map(e => this.fromPersistence(e));
    }

    async remove(id: string): Promise<boolean> {
        const result = await this.dataSource.getRepository(OrderEntity).delete({ id });
        return result.affected == 1;
    }
}
```

```ts
// src/shared/infrastructure/data-source.ts
import 'reflect-metadata';
import { DataSource } from 'typeorm';
import { OrderEntity, OrderLineEntity } from '../../orders/infrastructure/persistence/OrderEntity';

export const AppDataSource = new DataSource({
    type: 'better-sqlite3',
    database: 'database.sqlite',
    synchronize: true,           // solo en desarrollo; en producción, migraciones
    logging: false,
    entities: [OrderEntity, OrderLineEntity],   // añade aquí cada entidad persistente
});
```

---

## 4. Casos de uso

```ts
// src/orders/application/CreateOrder.ts
import { Either, left, right } from '../../shared/domain/Either';
import { UnexpectedError } from '../../shared/domain/DomainError';
import { DomainEvents } from '../../shared/domain/DomainEvents';
import { Order } from '../domain/Order';
import { OrderRepository } from '../domain/OrderRepository';
import { Currency } from '../domain/Money';

export class CreateOrder {
    constructor(private readonly orders: OrderRepository) {}

    async execute(dto: { customerId: string; currency: Currency }):
        Promise<Either<UnexpectedError, { orderId: string }>> {
        try {
            const order = Order.create(dto);
            await this.orders.save(order);
            DomainEvents.dispatch(order.pullDomainEvents());
            return right({ orderId: order.id.id });
        } catch (err) {
            return left(UnexpectedError.create(err));
        }
    }
}
```

```ts
// src/orders/application/AddOrderLine.ts
import { Either, left, right } from '../../shared/domain/Either';
import { EntityNotFoundError, UnexpectedError } from '../../shared/domain/DomainError';
import { OrderRepository } from '../domain/OrderRepository';
import { InvalidMoneyError, InvalidQuantityError, OrderNotEditableError } from '../domain/OrderErrors';

export type AddOrderLineErrors =
    | EntityNotFoundError
    | OrderNotEditableError
    | InvalidQuantityError
    | InvalidMoneyError
    | UnexpectedError;

export class AddOrderLine {
    constructor(private readonly orders: OrderRepository) {}

    async execute(dto: { orderId: string; productId: string; quantity: number; unitPriceCents: number }):
        Promise<Either<AddOrderLineErrors, void>> {
        try {
            const orderOrError = await this.orders.findById(dto.orderId);
            if (orderOrError.isLeft()) return left(orderOrError.value);
            const order = orderOrError.value;

            const result = order.addLine(dto.productId, dto.quantity, dto.unitPriceCents);
            if (result.isLeft()) return left(result.value);

            await this.orders.save(order);
            return right(undefined);
        } catch (err) {
            return left(UnexpectedError.create(err));
        }
    }
}
```

```ts
// src/orders/application/ConfirmOrder.ts
import { Either, left, right } from '../../shared/domain/Either';
import { EntityNotFoundError, UnexpectedError } from '../../shared/domain/DomainError';
import { DomainEvents } from '../../shared/domain/DomainEvents';
import { OrderRepository } from '../domain/OrderRepository';
import { EmptyOrderError, OrderNotEditableError } from '../domain/OrderErrors';

export class ConfirmOrder {
    constructor(private readonly orders: OrderRepository) {}

    async execute(dto: { orderId: string }):
        Promise<Either<EntityNotFoundError | OrderNotEditableError | EmptyOrderError | UnexpectedError, void>> {
        try {
            const orderOrError = await this.orders.findById(dto.orderId);
            if (orderOrError.isLeft()) return left(orderOrError.value);
            const order = orderOrError.value;

            const result = order.confirm();                  // la regla está en el agregado
            if (result.isLeft()) return left(result.value);

            await this.orders.save(order);                   // primero persistir...
            DomainEvents.dispatch(order.pullDomainEvents()); // ...luego notificar
            return right(undefined);
        } catch (err) {
            return left(UnexpectedError.create(err));
        }
    }
}
```

---

## 5. Adaptador HTTP (opcional, Express)

El controlador traduce errores del dominio a códigos HTTP con `instanceof`; el dominio no sabe nada de HTTP.

```ts
// src/orders/infrastructure/http/confirmOrderController.ts
import { Request, Response } from 'express';
import { ConfirmOrder } from '../../application/ConfirmOrder';
import { EntityNotFoundError } from '../../../shared/domain/DomainError';
import { EmptyOrderError, OrderNotEditableError } from '../../domain/OrderErrors';

export const confirmOrderController = (useCase: ConfirmOrder) => async (req: Request, res: Response) => {
    const result = await useCase.execute({ orderId: String(req.params.id) });
    if (result.isRight()) return res.status(204).send();

    const error = result.value;
    if (error instanceof EntityNotFoundError) return res.status(404).json({ error: error.errorMessage });
    if (error instanceof OrderNotEditableError) return res.status(409).json({ error: error.errorMessage });
    if (error instanceof EmptyOrderError) return res.status(422).json({ error: error.errorMessage });
    return res.status(500).json({ error: error.errorMessage });
};
```

---

## 6. Arranque

```ts
// src/index.ts
import { AppDataSource } from './shared/infrastructure/data-source';
import { DomainEvents } from './shared/domain/DomainEvents';
import { OrderConfirmedEvent } from './orders/domain/OrderEvents';
import { OrderRepositoryTypeORM } from './orders/infrastructure/persistence/OrderRepositoryTypeORM';
import { CreateOrder } from './orders/application/CreateOrder';
import { AddOrderLine } from './orders/application/AddOrderLine';
import { ConfirmOrder } from './orders/application/ConfirmOrder';

AppDataSource.initialize().then(async () => {
    DomainEvents.register<OrderConfirmedEvent>(OrderConfirmedEvent.name, (e) =>
        console.log(`Pedido ${e.orderId} confirmado: ${e.totalCents / 100} ${e.currency}`));

    const orders = new OrderRepositoryTypeORM(AppDataSource);
    const created = await new CreateOrder(orders).execute({
        customerId: '0b8f3c6e-7d2a-4f1b-9c3e-5a6d7e8f9a0b', currency: 'EUR',
    });
    if (created.isLeft()) return console.log(created.value.errorMessage);

    const { orderId } = created.value;
    await new AddOrderLine(orders).execute({ orderId, productId: 'book-ddd', quantity: 2, unitPriceCents: 4500 });

    const confirmed = await new ConfirmOrder(orders).execute({ orderId });
    if (confirmed.isLeft()) console.log('error:', confirmed.value.errorMessage);
}).catch(error => console.log(error));
```

---

## 7. Tests (Jest o Vitest, misma sintaxis)

```ts
// src/orders/domain/Order.test.ts
import { Order } from './Order';
import { OrderConfirmedEvent, OrderCreatedEvent } from './OrderEvents';
import { EmptyOrderError, InvalidQuantityError, OrderNotEditableError } from './OrderErrors';

const newOrder = () => Order.create({ customerId: 'c-1', currency: 'EUR' });

describe('Order', () => {
    it('records OrderCreatedEvent when created', () => {
        expect(newOrder().pullDomainEvents()[0]).toBeInstanceOf(OrderCreatedEvent);
    });

    it('cannot be confirmed without lines', () => {
        const result = newOrder().confirm();
        expect(result.isLeft()).toBe(true);
        expect(result.value).toBeInstanceOf(EmptyOrderError);
    });

    it('merges lines for the same product and computes the total', () => {
        const order = newOrder();
        order.addLine('p-1', 1, 500);
        order.addLine('p-1', 3, 500);
        order.addLine('p-2', 1, 250);
        expect(order.lines).toHaveLength(2);
        expect(order.total().cents).toBe(2250);
    });

    it('rejects a quantity above the maximum, even when merging', () => {
        const order = newOrder();
        order.addLine('p-1', 98, 100);
        const result = order.addLine('p-1', 2, 100);
        expect(result.value).toBeInstanceOf(InvalidQuantityError);
    });

    it('records OrderConfirmedEvent and becomes read-only', () => {
        const order = newOrder();
        order.addLine('p-1', 2, 1000);
        order.pullDomainEvents();

        expect(order.confirm().isRight()).toBe(true);
        const [event] = order.pullDomainEvents();
        expect(event).toBeInstanceOf(OrderConfirmedEvent);
        expect((event as OrderConfirmedEvent).totalCents).toBe(2000);
        expect(order.addLine('p-2', 1, 100).value).toBeInstanceOf(OrderNotEditableError);
    });
});
```

```ts
// src/orders/application/ConfirmOrder.test.ts
import { ConfirmOrder } from './ConfirmOrder';
import { MemoryOrderRepository } from '../infrastructure/persistence/MemoryOrderRepository';
import { Order } from '../domain/Order';
import { OrderConfirmedEvent } from '../domain/OrderEvents';
import { EntityNotFoundError } from '../../shared/domain/DomainError';
import { DomainEvents } from '../../shared/domain/DomainEvents';

describe('ConfirmOrder', () => {
    beforeEach(() => DomainEvents.clearHandlers());

    it('confirms, saves and dispatches OrderConfirmedEvent', async () => {
        const repo = new MemoryOrderRepository();
        const order = Order.create({ customerId: 'c-1', currency: 'EUR' });
        order.addLine('p-1', 1, 999);
        order.pullDomainEvents();
        await repo.save(order);

        const received: OrderConfirmedEvent[] = [];
        DomainEvents.register<OrderConfirmedEvent>(OrderConfirmedEvent.name, e => received.push(e));

        const result = await new ConfirmOrder(repo).execute({ orderId: order.id.id });

        expect(result.isRight()).toBe(true);
        const stored = await repo.findById(order.id.id);
        expect(stored.isRight() && stored.value.status).toBe('confirmed');
        expect(received.map(e => e.totalCents)).toEqual([999]);
    });

    it('returns EntityNotFoundError for an unknown order', async () => {
        const result = await new ConfirmOrder(new MemoryOrderRepository())
            .execute({ orderId: '6f1c2a7e-3b4d-4e5f-8a9b-0c1d2e3f4a5b' });
        expect(result.value).toBeInstanceOf(EntityNotFoundError);
    });
});
```

```ts
// src/orders/infrastructure/persistence/OrderRepositoryTypeORM.test.ts
import 'reflect-metadata';
import { DataSource } from 'typeorm';
import { OrderEntity, OrderLineEntity } from './OrderEntity';
import { OrderRepositoryTypeORM } from './OrderRepositoryTypeORM';
import { Order } from '../../domain/Order';

describe('OrderRepositoryTypeORM (SQLite en memoria)', () => {
    let dataSource: DataSource;

    beforeEach(async () => {
        dataSource = new DataSource({
            type: 'better-sqlite3', database: ':memory:', synchronize: true,
            entities: [OrderEntity, OrderLineEntity],
        });
        await dataSource.initialize();
    });
    afterEach(() => dataSource.destroy());

    it('round-trips the whole aggregate without re-emitting events', async () => {
        const repo = new OrderRepositoryTypeORM(dataSource);
        const order = Order.create({ customerId: 'c-1', currency: 'EUR' });
        order.addLine('p-1', 2, 1500);
        order.addLine('p-2', 1, 300);
        await repo.save(order);

        const loaded = await repo.findById(order.id.id);
        if (loaded.isLeft()) throw new Error('should exist');
        expect(loaded.value.equals(order)).toBe(true);
        expect(loaded.value.total().cents).toBe(3300);
        expect(loaded.value.pullDomainEvents()).toHaveLength(0);
    });

    it('deletes lines removed from the aggregate', async () => {
        const repo = new OrderRepositoryTypeORM(dataSource);
        const order = Order.create({ customerId: 'c-1', currency: 'EUR' });
        order.addLine('p-1', 1, 100);
        order.addLine('p-2', 1, 100);
        await repo.save(order);

        order.removeLine('p-1');
        await repo.save(order);

        expect(await dataSource.getRepository(OrderLineEntity).count()).toBe(1);
    });
});
```

Prioriza los tests del dominio (sin base de datos) y de casos de uso con el repositorio en memoria, y deja unos pocos de integración por adaptador.
