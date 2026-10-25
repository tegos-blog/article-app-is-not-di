# app() Is Not Dependency Injection: 18 Calls Gone Before the Rule

![app() Is Not Dependency Injection: 18 Calls Gone Before the Rule](assets/poster.jpg)

I keep hearing "we use DI everywhere" from Laravel devs whose code is full of `app(SomeService::class)`. That's Service Locator, and I've said so in code reviews for years. Still, the git history of the project I work on had 18 of these calls at the peak.

Knowing it's an anti-pattern and not writing it are two different things. This is how those 18 calls went to zero, with no linter, and the written rule came only at the end.

## What's Inside

- Service Locator vs dependency injection: the difference is who knows about whom, not who creates the object
- Four years of git history: from a peak of 18 calls to 0 while `app/` grew from 371 to 1166 files
- The last call: a model that built its own primary key, and the unit test that pushed it out
- Form request `after()` accepts method injection, even though the docs don't mention it (verified on Laravel 12.43.1)
- Where `app()` is still fine, and a three-line rule of thumb

## 📎 Read Full

[app() Is Not Dependency Injection: 18 Calls Gone Before the Rule](https://dev.to/tegos/PLACEHOLDER)
