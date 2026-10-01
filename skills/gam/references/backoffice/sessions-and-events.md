---
name: backoffice-sessions-and-events
description: GAM Backoffice — Connections, Sessions, and Event Subscriptions sections
---

# GAM Backoffice — Connections / Sessions / Event Subscriptions
These three Backoffice sections handle runtime state and integration wiring

---

## Connections (GAM_Connections)
Screen (object name as shipped in `GAM_Web-Administration`): `GAMExampleWWConnections`

Connection.gam configuration — see [GAM Connection & Configuration Encyclopedia](../multi-tenant/domain-connection-config.md)

---

## Sessions (GAM_Sessions)
Screen (object name as shipped in `GAM_Web-Administration`): `GAMExampleWWSessions`

Active session list. Allows:
- Viewing sessions by user
- Invalidating sessions
- Viewing statistics

---

## Event Subscriptions (GAM_Event_Susbcriptions)
Screen (object name as shipped in `GAM_Web-Administration`): `GAMExampleWWEventSubscriptions`

Fields: Description, Event Type, FileName, ClassName, MethodName

Subscriptions are NOT pre-created by GAM: a fresh repository has none, and rows are added here or by code. Adding the row registers the handler; it only runs once its status is `Subscribed` (`GAMRepository.SubscribeEvent` by code). Subscriptions are per repository: every application in the repository sees the same active handlers

The handler program contract covers the `parm` signature, mandatory properties, payload per event, the `FileName`/`ClassName`/`MethodName` convention per generator, and the silent-failure trap when the handler sits inside a `Module`; it lives in [GAM Events Subscription: Handler Contract](../events/domain-event-subscription.md). Registration by code: [GAMEventSubscription Code Provider](../kb-setup/init/code-provider/gameventsubscription-code-provider.md)

Note on module name: the module identifier in the Backoffice export is `GAM_Event_Susbcriptions` (typo preserved from source). The user-facing menu label is "Event Subscriptions"
