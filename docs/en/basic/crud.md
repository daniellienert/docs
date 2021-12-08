# Create, Update and Delete entities
Any persistence operation with entity or entities has to be done using the `Cycle\ORM\EntityManager` object.

> Read how to [describe your entity here](/docs/en/annotated/entity.md).

## Create Entity
In order to create an entity simply pass its instance to the transaction object and invoke the method `run`:

```php
$user = new User();
$user->setName("Antony");

$manager = new \Cycle\ORM\EntityManager($orm);
$manager->persist($user);

$state = $manager->run();
```

Метод `run` возвращает объект `\Cycle\ORM\Transaction\StateInterface` содержащий информацию о статусе выполнения транзакции. 
С помощью него вы можете проверить статус выполнения транзакции и при необходимости перезапустить.

```php
$user = new User();
$user->setName("Antony");

$manager = new \Cycle\ORM\EntityManager($orm);
$manager->persist($user);

$totalTries = 5;

$logger = new MyLogger(...);
$state = $manager->run();

try {
    while (!$state->isSuccess() && $totalTries > 0) {
        $logger->error($state->getError());
        $state = $state->retry();
        $totalTries--;
    } 
} catch (\Cycle\ORM\Exception\SuccessTransactionRetryException $e) {
    // OK
}
```

In order to process persistent errors make sure to handle exceptions produced by the `run` method:

```php
try {
    $state = $manager->run();
    if ($state->getError()) {
        throw $state->getError()
    }
} catch (\Throwable $e) {
   print_r($e);
}
```

One of the most important types of exception you must handle is `Cycle\Database\Exception\DatabaseException`. This exception branches
into multiple types for each of the error types:

```php
use Cycle\Database\Exception\StatementException;

try {
    $state = $manager->run();
    if ($state->getError()) {
        throw $state->getError()
    }
} catch (StatementException\ConnectionException $e) {
   print_r("database has gone away");
} catch (StatementException\ConstrainException $e) {
   print_r("database constrain not met, nullable field?");
}
```

## Update the Entity
In order to update the entity you must first obtain the loaded entity object:

```php
$user = $orm->getRepository(User::class)->findByPK(1);
```

Simply change desired entity fields and register it in the transaction using `persist` method:

```php
$user->setName("John");

$manager = new \Cycle\ORM\EntityManager($orm);
$manager->persist($user);
$state = $manager->run();
```

Note, by default, ORM will update only changed entity fields (a.k.a. dirty state), given code would produce
SQL code similar to:

```sql
UPDATE `users` SET `name` = "John" WHERE `id` = 1
```

## Delete Entity
Any entity can be deleted using the transaction method `delete`:

```php
$user = $orm->getRepository(User::class)->findByPK(1);

$manager = new \Cycle\ORM\EntityManager($orm);
$manager->delete($user);
$state = $manager->run();
```

Please note, the ORM will not automatically trigger the delete operation for related entities and will rely on foreign key rules set in the database.

## Persisting Related Entities
Persisting an entity will also persist all related entities within it.

```php
$user = new User();
$user->setAddress(new Address());
$user->getAddress()->setCountry("USA");

$manager = new \Cycle\ORM\EntityManager($orm);
$manager->persist($user);
$state = $manager->run();

print_r($user->getAddress()->getID());
```

This behavior is enabled by default by persisting the entity with the `cascade: true` flag.
Code above can be equally rewritten as:

```php
use Cycle\ORM\EntityManager;

$manager = new EntityManager($orm);
$manager->persist($user, cascade: true);
$state = $manager->run();
```

Pass the `cascade: false` flag to disable cascade persisting of related entities:

```php
use Cycle\ORM\EntityManager;

$manager = new EntityManager($orm);
$manager->persist($user, cascade: false);
$state = $manager->run();
```

> The `cascade: false` flag can be used while creating or updating the entity.

You can also turn off cascading on the relation level by setting `cascade` flag to `false`.