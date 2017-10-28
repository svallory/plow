# The building blocks

{% method %}
## Domain Events

Events represent change. Specifically a change that has already happened. They are immutable. Events are used inside Entities
to apply the change to it's internal state and are propagated throughout the system to notify interested parties that something happened.

Here's is a simple event implementation that represents the fact that a task was completed:

{% sample lang="ts" %}
```ts
import { Guid } from '@cashfarm/lang';
import { DomainEvent } from '@cashfarm/plow';

@DomainEvent
class TaskCompleted {
    constructor(
        public readonly taskId: Guid,
        public readonly done: boolean
    ) {}
}
```

{% sample lang="js" %}
```js
import { Guid } from '@cashfarm/lang';
import { DomainEvent } from '@cashfarm/plow';

@DomainEvent
class TaskCompleted {
    constructor(
        public readonly taskId: Guid,
        public readonly done: boolean
    ) {}
}
```
{% endmethod %}

!INCLUDE "blocks/entities.md"

{% method %}
## Repositories

Repositories are how we call the classes that permanently store aggregate roots.
If you are using [GetEventStore](http://geteventstore.com) to store your aggregates, all you
need to do is extend the class `EventSourcedRepositoryOf<TAggregate, TAggtId>`;

If you are using Plow's container, you can simply inject `IEventStore` and `IEventPublisher` as
they are already configured in the inversify container.

{% sample lang="ts" %}
```ts
export class TaskRepository extends EventSourcedRepositoryOf<Task, Guid> {
  constructor(
    @inject(IEventStore) protected storage: IEventStore,
    @inject(IEventPublisher) eventPublisher: IEventPublisher
  ) {
    super(storage, Campaign, eventPublisher);
  }
}
```

{% sample lang="js" %}
```js
export class TaskRepository extends EventSourcedRepositoryOf<Task, Guid> {
  constructor(
    @inject(IEventStore) protected storage: IEventStore,
    @inject(IEventPublisher) eventPublisher: IEventPublisher
  ) {
    super(storage, Campaign, eventPublisher);
  }
}
```

{% common %}
You can then use the rpository methods `save(aggregate: TAggregate): Promise<DomainEvent[]>` and
`getById(id: TId): Promise<TAggregate>` to store and retrieve aggregates.

> **[info] Note:**
> Trying to save an aggregates whith no events to be persisted will throw an error.

Plow will propagate the persisted events through the event bus.
{% endmethod %}

{% method %}
## Projections

A projection class implements methods that handle events with the intent of projecting the data to
a read model. The read model is a denormalized databsed optimized for reading.

Here's an example of a projection that sums up the number of completed tasks

{% sample lang="ts" %}

```ts
import { Handle } from '@cashfarm/plow';

import { TaskCompleted } from '../events';

@Projection(TaskCompleted) // This decorator will register the projection in the event bus
export class TaskProjections {
  constructor(
      @inject(TaskStore) private tasks: TaskStore,
      @inject(UserStore) private users: UserStore) {}

  public [Handle(TaskCompleted)](event: TaskCompleted): void {
    debug('Running projection for TaskCompleted');
    const task = this.tasks.findById(event.taskId);
    const user = this.users.findById(task.userId);

    user.completedTasks += 1;

    this.users.update(user);
  }
}
```

{% sample lang="js" %}
```js
import { Handle } from '@cashfarm/plow';

import { TaskCompleted } from '../events';

@Projection(TaskCompleted) // This decorator will register the projection in the event bus
export class TaskProjections {
  constructor(
      @inject(TaskStore) private tasks: TaskStore,
      @inject(UserStore) private users: UserStore) {}

  public [Handle(TaskCompleted)](event: TaskCompleted): void {
    debug('Running projection for TaskCompleted');
    const task = this.tasks.findById(event.taskId);
    const user = this.users.findById(task.userId);

    user.completedTasks += 1;

    this.users.update(user);
  }
}
```
{% endmethod %}