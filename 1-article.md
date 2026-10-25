I keep hearing "we use DI everywhere" from Laravel devs whose code is full of `app(SomeService::class)`. The container builds the object, so it feels like DI. It's not. It's Service Locator, and I've said so in code reviews for years.

And still, when I went through the git history of the project I work on, I counted 18 of these calls at the peak. Knowing it's an anti-pattern and not writing it are two different things. This is how that gap closed.

## "But We Use DI Everywhere"

Here's what I mean. Both of these get their dependency from the container. The first one is from our codebase in 2024:

```php
// app/Models/User.php, 2024: Service Locator, the class asks for it
public function getCart(): Cart
{
    /** @var CartService $cartService */
    $cartService = app(CartService::class);

    return $cartService->getCartEntity($this);
}
```

The second one is how a cart action looks on our main branch today:

```php
// app/Actions/Cart/CartIndexAction.php, today: dependency injection, the class receives it
final readonly class CartIndexAction implements Actionable
{
    public function __construct(
        private CartItemDtoFactory $cartItemDtoFactory
    )
    {
    }

    public function handle(CartItemParamDTO $cartItemParamDTO, int $userId, ?int $managerId = null): Collection
    {
        $cartItemDTOs = $this->cartItemDtoFactory->fetchForDisplay($userId, $managerId, $cartItemParamDTO->itemUuids);

        return $cartItemDTOs->values();
    }
}
```

The container creates the dependency in both cases. What changes is **who knows about whom**. In the first one, `User` knows about the container and hides its dependency inside a method body. In the second one, the class has no idea where `CartItemDtoFactory` comes from, and the constructor tells you everything it needs.

That's the whole cost of Service Locator. The class signature lies about its dependencies, a missing binding fails at runtime instead of at construction, and a constructor that grows too long is a useful warning that `app()` simply hides.

The Laravel docs lean the same way. They show `app()` and `App::make()` for code that has no `$app` variable at hand, and then, about constructor type-hints:

> In practice, this is how most of your objects should be resolved by the container.

One scope note before the numbers. Facades and helpers like `config()` or `Cache::` pull from the container too, and some people call them Service Locator as well. I didn't count them. That's a different debate. This post is only about explicit `app(Foo::class)`-style resolution of the project's own classes.

## What Four Years of Git History Show

I counted calls to `app(Foo::class)`, `app()->make()`, `App::make()` and `resolve(Foo::class)` in `app/` on the main branch at a few points in time:

| Date | Service Locator calls | PHP files in `app/` |
|---|---|---|
| 2022-07 | 5 | 117 |
| 2023-01 | 12 | 235 |
| 2023-07 | 18 | 371 |
| 2024-01 | 18 | 428 |
| 2024-07 | 16 | 548 |
| 2025-01 | 11 | 856 |
| 2025-07 | 3 | 965 |
| 2026-01 | 1 | 1052 |
| 2026-05 | 0 | 1058 |
| 2026-10 | 0 | 1166 |

From the 18-call peak down to 0, while `app/` grew from 371 to over 1,000 files.

**All four spellings lived in the same codebase.** Nobody ever decided "we use a Service Locator here". It just happened.

The worst method I found, in a search service from 2024, resolved four services in a row:

```php
// app/Services/Search/SearchResultService.php, 2024
/** @var PriceService $priceService */
$priceService = App::make(PriceService::class);
/** @var CurrencyService $currencyService */
$currencyService = App::make(CurrencyService::class);
// ...
/** @var CartService $cartService */
$cartService = App::make(CartService::class);
/** @var WarehouseService $warehouseService */
$warehouseService = App::make(WarehouseService::class);
```

Four dependencies, and the constructor mentioned none of them.

A good part of the drop was not refactoring at all. Of the 18, 10 went away only because the whole file was deleted. `User::getCart()` from the example above was dead too. The cart actions had switched to an injected cart repository long before, and only other dead code still called the model method. It went out with a "Cleanup" commit, together with a few similar calls. The last one needed a real refactoring. Let's be honest: most of these calls lived in old code, and old code gets deleted.

## The Last One: A Model That Built Its Own Primary Key

The last call survived until April 2026. It lived in the `Order` model, which generated its own 8-digit id with a year prefix:

```php
// app/Models/Order.php, until April 2026
public function newUniqueId(): string
{
    $now = Date::now();
    /** @var PrimaryKeyIntegerGenerator $primaryKeyIntegerGenerator */
    $primaryKeyIntegerGenerator = app(PrimaryKeyIntegerGenerator::class);
    $primaryKeyIntegerGenerator
        ->setModel($this)
        ->setLockSeconds(config('limits.order.min_processing_time_lock'))
        ->setPrefix($now->format('y'))
        ->setBuilderAggregate(Order::query()->whereYear('created_at', $now));

    return strval($primaryKeyIntegerGenerator->generate());
}
```

Eloquent creates models with `new`, not through the container, so you can't inject anything into a model constructor. And `newUniqueId()` is a hook Eloquent calls itself, with no arguments. Inside a hook like that, you can only ask for a service: `app()` or a facade. So **if a model needs `app()`, the logic is probably in the wrong place**.

To be fair, `app()` itself wasn't what blocked testing. In Laravel you can swap a binding with `$this->instance()` or `$this->mock()`. The real problem was the order-specific logic (prefix, lock, year scope) living in a model hook. When I wanted to test the edge cases (a locked id, too many attempts), I had to pull that logic out first:

```php
// app/Actions/Order/OrderCreateAction.php, today
final readonly class OrderCreateAction implements Actionable
{
    public function __construct(
        // ...
        private OrderPrimaryKeyGenerator $orderPrimaryKeyGenerator,
        // ...
    )
    {
    }

    public function handle(OrderCreateDTO $orderCreateDTO, int $userId, ?int $managerId = null): Order
    {
        // ...
        $id = $this->orderPrimaryKeyGenerator->generate();

        $orderData['id'] = $id;
        $orderData['number'] = $id;
        // ...
    }
}
```

The trade-off: the model has `$incrementing = false`, so an `Order::create()` that skips the action gets no id from Laravel. In our case both order create actions inject the generator, and the factory sets its own id.

And now the generator gets its own test, no order creation needed:

```php
public function test_order_pk_skips_locked_id_and_tries_next(): void
{
    // Arrange
    $year = Date::now()->format('y');
    $lockedId = intval($year . '000001');
    Cache::lock(Order::class . ':' . $lockedId, 60)->get();

    // Act
    $id = (new OrderPrimaryKeyGenerator())->generate();

    // Assert
    $this->assertSame(intval($year . '000002'), $id);
}
```

Yes, the generator still uses the `Cache` and `Date` facades, `config()`, and `Order::query()` inside. So the test still boots the app and touches the database. That's the scope note from above: facades stayed, the hidden `app()` lookup of our own class went.

The commit that removed the last `app()` call was titled "Test: Add failing unit tests for OrderPrimaryKeyGenerator". A test pushed it out.

## No Lint, No Blocking Review

There was no PHPStan rule, no architecture test, no blocking review.

Over the years I mentioned it to a colleague in code review a few times. More like a note than a rule. At some point I even admitted I use `app()` myself sometimes and told him to do it his way. My own new calls kept coming for months after that. By `git blame`, I added more of these calls than he did. So this isn't a story about fixing someone else's code. We both wrote these calls and we both deleted them.

A written rule did appear eventually. "Avoid `app()`, prefer constructor injection" went into our project guidelines in April 2026, when exactly one call was left. Two weeks later, that one was gone too.

## Where You Don't Need app() (And Where It's Fine)

Before reaching for `app()`, check if Laravel already calls your method through the container. It does in more places than most people think:

- `handle()` in commands and queued jobs
- `rules()` and `authorize()` in form requests (documented)
- `after()` in form requests (not documented, but it goes through `container->call()` too)

We once had this in a few form requests (December 2024):

```php
// app/Http/Requests/.../CartItemImportRequest.php, 2024
public function after(): array
{
    /** @var CartValidateImportFileAction $cartValidateImportFileAction */
    $cartValidateImportFileAction = App::make(CartValidateImportFileAction::class);

    return [
        function (Validator $validator) use ($cartValidateImportFileAction): void {
            $cartValidateImportFileAction->handle($validator, $this->file('import_file'));
        },
    ];
}
```

We ended up dropping that hook for other reasons, so there is no real "after" version. But it never needed `App::make()`. This is how it could have looked:

```php
// how it could have looked: method injection
public function after(CartValidateImportFileAction $validateImportFile): array
{
    return [
        function (Validator $validator) use ($validateImportFile): void {
            $validateImportFile->handle($validator, $this->file('import_file'));
        },
    ];
}
```

Heads up: the docs mention dependency injection for `rules()` and `authorize()`, not for `after()`. I checked it on Laravel 12.43.1: the injected class is resolved and the hook runs. I didn't check older versions. One more gotcha: Laravel passes the validator to `after()` by name, so don't call your injected parameter `$validator`.

Where `app()` is OK: service providers, obviously. Code that Laravel builds without the container, like models, when you really can't move the logic out. Lazy resolution of something heavy you only need in one branch. That's a short list. Most of the rest, in my experience, should have been a constructor parameter.

## My Rule of Thumb

Not a rule for the team, just what the history taught me:

- If a class calls `app()` for one of your own services, that service belongs in the constructor.
- If a model needs `app()`, move the logic to a service or an action.
- If you can't control the constructor, try method injection first. `app()` comes last.

## TL;DR

- `app(Foo::class)` is resolved by the container, but it's still the Service Locator pattern. DI means the dependency arrives through the signature.
- In one real project, it went from a peak of 18 calls to 0 while the code roughly tripled. Many calls simply died with old code.
- The last one was in a model hook, and writing a test for it pushed it out. The written rule landed when only that one was left.
- Form request `after()` accepts method injection too (undocumented, verified on Laravel 12.43.1).

---

- [Laravel docs: Automatic Injection](https://laravel.com/docs/12.x/container#automatic-injection)
- [Laravel docs: Performing Additional Validation on Form Requests](https://laravel.com/docs/12.x/validation#performing-additional-validation-on-form-requests)
- [Mark Seemann: Service Locator is an Anti-Pattern](https://blog.ploeh.dk/2010/02/03/ServiceLocatorisanAnti-Pattern/)
- [Jimmy Bogard: Service Locator is not an Anti-Pattern](https://www.jimmybogard.com/service-locator-is-not-an-anti-pattern/) (a fair counterpoint)
- [Adel Faizrakhmanov: Architecture of Complex Web Applications, Dependency Injection chapter](https://github.com/adelf/acwa_book_en/blob/master/manuscript/2-di.md)
- [Laravel Actions and Services](https://dev.to/tegos/laravel-actions-and-services-360d) (my earlier post on where this logic goes)

## Author's Note

Thanks for sticking around!
Find me on [dev.to](https://dev.to/tegos), [linkedin](https://www.linkedin.com/in/ivan-mykhavko/), or you can check out my work on [github](https://github.com/tegos).

**Laravel, after the happy path.**
