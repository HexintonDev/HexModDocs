# Trainer Controls and UI Bindings

Status: current public technical guide.

Lua functions become usable trainer features through package declarations. The runtime defines the
operation; the hosted trainer view defines how that operation is presented to a user.

## Actions, Queries, and Services

- Use an `action` for a button, toggle write, patch, or explicit refresh.
- Use a `query` for a one-shot search or read requested by the user.
- Use a `service` for changing state that should be displayed continuously.

An action result means the operation reached a terminal result. It does not automatically mean that
a later state feed has updated. Keep command completion and authoritative state separate.

## Hosted View Models

Declare a service-backed view model in `frontend.hosted`:

```json
{
  "frontend": {
    "hosted": {
      "schemaVersion": 1,
      "viewModels": [
        { "id": "health", "service": "watch-health", "autoStart": true }
      ],
      "groups": []
    }
  }
}
```

The service should publish a small initial JSON payload and later publish only meaningful changes.
The host adds package scope and revision metadata. Do not use a query as a replacement for a live
service.

## Binding an Action

```json
"bindings": {
  "health.current": {
    "type": "viewModel",
    "viewModel": "health"
  },
  "health.set": {
    "type": "command",
    "action": "set-health",
    "executionPolicy": "write"
  }
}
```

The binding alias is local to the surface. The command's arguments are validated against the action
`parameterSchema` in the package manifest.

## Queries and Feeds

```json
"inventory.search": {
  "type": "query",
  "query": "search-inventory"
},
"inventory.state": {
  "type": "feed",
  "stateFeed": "resources"
}
```

Queries are appropriate for search and pagination. A feed connects a surface to a service-backed
state feed and carries a scoped value plus a monotonically increasing revision.

## Custom Surface Declaration

```json
{
  "frontend": {
    "surfaces": [
      {
        "id": "health-panel",
        "kind": "web-inline",
        "label": "Health",
        "entry": "frontend/health/index.html",
        "presentation": { "mode": "panel", "size": "large" },
        "bindings": {
          "health": { "type": "viewModel", "viewModel": "health" },
          "setHealth": { "type": "command", "action": "set-health", "executionPolicy": "write" }
        }
      }
    ]
  }
}
```

Supported placements are `web-inline` with `compact` or `panel` mode, and `web-app` with `app` mode.
The current host loads one package-relative HTML document and enforces a 1 MiB document limit.

Surface code runs in a sandbox. It receives only declared bindings and must use the provided surface
SDK. It must not call the raw WebView bridge, access credentials, or assume unrestricted filesystem,
network, or native access.

## Toggle with a Value (implementation notes)

Status: current implementation on HexModClient `feature/toggle-with-value` (2026-10-26). The public
contract and complete example are in the public engine wiki `trainer-controls.md#toggle-with-a-value`.

- Manifest: a `toggle` with `write.valueParameter`, `valueType` `number`/`integer`, `constraints` and
  `toggleValue { default, control? }`. The desktop compiler (`HostedTrainerViewCompiler.ToggleValue.cs`)
  fills a missing `constraints.step`, puts `defaultValue` and `variant` (`slider`/`input`) into the
  compiled presentation, and rejects an on action whose `parameterSchema.required` lists the parameter.
- State: `HostedTrainerWidget.Value` stays the on/off boolean. The held number is `Setting`, projected
  as `values[packageId]["<localId>#value"]` (`#` is outside the widget ID alphabet). It survives detach
  and recompiles of the same package version; Save Mods stores an on toggle as its number.
- Commands: `true`/`false` turn it on/off (the on action gets the setting, read inside the session
  transaction); a number changes the setting, re-running the on action only while the toggle is on.
  Desktop, overlay and phone remote send the same `trainer.runAction` value.
- Compatibility decision: graceful fallback instead of a minimum app version. Older apps ignore
  `toggleValue`/`write.valueParameter` and show a plain toggle that calls the on action with `{}`, so
  packages default the parameter in Lua. The default is not stored in `presentation.defaultValue`
  because older compilers reject that key on toggles. Older phone pages accept the projection
  (unknown values keys are ignored; `variant` was already allowed).
- Tests: Studio/engine-kit `toggle` steps accept an optional numeric `value` on `"enabled": true`.

## UI Design Rules

Use a toggle for a reversible enable/disable feature, a command button for a one-time action, a
query for transient results, and a read-only view model value for authoritative state. Keep local
draft input separate from committed service state.