---
name: gameventsubscription-code-provider
description: Capabilities and quirks for idempotent initialization of GAMEventSubscription. EO is source of truth for properties and methods
---

# GAMEventSubscription Code Provider
**EO**: `GAMEventSubscription` (`EventSubscription` in `GeneXusSecurity.Tenants.Common` on the modular naming). Locate by content header, not by an assumed file-name suffix (see [global-constraints.md § Where GAM's EOs live](../../../global-constraints.md#where-gams-eos-live))
Derive the SDT using the [SDT Generation Workflow](../../../global-constraints.md#sdt-generation-workflow--derive-the-sdt-11-from-the-eo)
The program that ATTENDS the event has its own contract, see [GAM Events Subscription: Handler Contract](../../../events/domain-event-subscription.md)

---

## What you can use
- **Idempotency strategy**: B, filter lookup then `Load(Id)` or `new()`. There is no static ID getter on this EO, so a new subscription has nothing to load; resolve the id first via `GAMRepository.GetEventSubscriptions` (see below)
- **Capabilities**: `Load(Id)`, `Save()`, `Delete()`, `Success()`, `Fail()`, `GetErrors()`; read the EO for exact signatures
- **Activation**: `GAMRepository.SubscribeEvent(EventId, out Errors)`, static, returns `GAMBoolean`. Must be called AFTER `Save()` and `Commit` (see API quirks below)
- **Read/Write properties**: `Id`, `Description`, `Status`, `Event`, `FileName`, `ClassName`, `MethodName`, the four audit fields, and `Properties`. Open the EO for the complete list before writing the SDT

## Known facts (not derivable from the EO)
- The identifier (`BasedOn = 'GAMGUID, GeneXusSecurityCommon'`) is assigned by GAM on `Save()`, so never set or hardcode it. **Its casing depends on the layout:** `GAMEventSubscription.Id` on release, `EventSubscription.ID` on the modular naming; read the EO, see [global-constraints.md § Member casing changed with the layout](../../../global-constraints.md#member-casing-changed-with-the-layout)
- `Event` must be a value of the `GAMEvents` Domain. Read `ref/GeneXusSecurityCommon/#domains/GAMEvents.gx` and use dot notation (`GeneXusSecurityCommon.GAMEvents.User_Insert`), never a string literal and never a remembered list
- `FileName`, `ClassName` and `MethodName` follow a per-generator convention, and a wrong `ClassName` fails silently, see [the handler contract](../../../events/domain-event-subscription.md#registration-filename--classname--methodname)
- **GAM does NOT pre-create subscriptions.** A fresh repository has none, so the flow starts at `new()`; the filter lookup exists only for idempotency across re-runs

## Idempotency: filter lookup then Load or new
Lookup uses `GAMEventSubscriptionFilter`, whose properties are `Id`, `Descripction`, `Status`, `Event`, `FileName`, `ClassName`, `MethodName`

> `Descripction` is misspelled in the EO itself; write it exactly as declared, because "correcting" it to `Description` does not compile

```genexus
&EventSubscriptionFilter = new()
&EventSubscriptionFilter.Event = &EventSubscriptionDefinition.Event
&EventSubscriptionCollection = GAMRepository.GetEventSubscriptions(&EventSubscriptionFilter, &GAMErrors)
```

Then match the collection on `Event` + `ClassName` + `MethodName` to get `Id`, and branch:
- match found → `&EventSubscription.Load(&EventSubscriptionId)`; if `Success()` fails, fall back to `new()`
- no match → `&EventSubscription = new()`

Those three fields together are what GAM treats as unique per repository: matching on fewer of them makes the init non-idempotent and the second run fails with "event subscription already exists"

## SDT scope: documented deviation from the 1:1 rule
The definition SDT carries 5 members: `Description`, `Event`, `FileName`, `ClassName`, `MethodName`

The EO declares 12 `Read/Write` properties, and the trimmed ones fall under the [documented exception](../../../global-constraints.md#sdt-generation-workflow--derive-the-sdt-11-from-the-eo): `Id` is GAM-assigned, the four audit fields (`DateCreated`, `UserCreated`, `DateUpdated`, `UserUpdated`) are GAM-written, `Status` is set by `SubscribeEvent` rather than by the caller, and `Properties` (a key/value collection) is not used by this initialization; add it back if the handler needs custom properties

## API quirks and order-of-operations
- **Two-step operation**: `Save()` and `Commit` write the subscription record, and `GAMRepository.SubscribeEvent(&EventSubscription.Id, &GAMErrors)` activates it. Both are required; without `SubscribeEvent` the handler is defined but never triggered
- **Scope is the repository.** The EO description says *"Subscribes the current application to the specified repository event"*, but subscriptions are stored and resolved per repository, so every application in the repository sees the same active handlers. `UnsubscribeEvent(EventId, out Errors)` is the inverse
- Assigning `Status = GAMEventSubscriptionStatus.Subscribed` is NOT an alternative to `SubscribeEvent`, because `Save()` ignores `Status`. `SubscribeEvent` and `UnsubscribeEvent` are the only way to change it, and a subscription that is not `Subscribed` never runs
- `SubscribeEvent` returns `GAMBoolean`, so it needs an assignment target and the result must be checked before reporting success
- Call `SubscribeEvent` only inside the `If &EventSubscription.Success()` branch, after `Commit`
- **`&GAMErrors` belongs to this file, not to the handler.** `GetEventSubscriptions`, `SubscribeEvent` and `UnsubscribeEvent` declare an `out Errors` parameter and the init procedure must iterate it. The handler program does the opposite: it never builds or consumes a `GAMErrorCollection`, because GAM owns error handling for the operation that fired the event and the handler communicates only through `&JsonOut`

---

## Cross-references
- EO `GAMEventSubscription` and filter EO `GAMEventSubscriptionFilter`
- [GAM Events Subscription: Handler Contract](../../../events/domain-event-subscription.md)
- [SDT Generation Workflow](../../../global-constraints.md#sdt-generation-workflow--derive-the-sdt-11-from-the-eo)
- [GAM Entity Initialization — Declarative Pattern](../entity-initialization.md)
- [GAM Entity Initialization — Consolidated Pattern](../entity-initialization-consolidated.md) (default pattern)
- [Backoffice: sessions and events](../../../backoffice/sessions-and-events.md)
