# SOLID for OSGi/DS bundles — review rules

How each principle concretely manifests (and is violated) in a bnd/DS workspace like the fennec projects. Findings must cite the class/component, not just the principle.

## S — Single Responsibility

- **Component level:** one DS component = one reason to change. Smells: a component that both serves REST and talks to storage; parsing + persistence + eventing in one class; `activate()` doing unrelated setups. Fix direction: split components, wire via references.
- **Bundle level:** one concern per bundle. fennec already separates `*.api` vs impl bundles and per-backend `management.*` bundles — flag NEW mixing (e.g. webhook parsing inside a storage bundle), not the existing layout.
- Metric hints (not findings by themselves): class > ~500 lines, component with > ~6 `@Reference`s, method > ~60 lines — read those first.

## O — Open/Closed

- Extension in OSGi = **new service implementations**, not editing a switch. Smells: `if/else` or `switch` chains on a type/format/backend name where a whiteboard pattern (service + target filter/service property) exists or should; hardcoded lists of supported backends/formats that each new backend must edit.
- Good existing pattern to compare against: pluggable storage via `storageService.target=(storage.type=...)` and media-type codec tracking. Flag code that bypasses these with hardcoded backend knowledge.

## L — Liskov Substitution

- Every implementation of a service API must honor the interface's documented contract: same exception discipline, same null/Promise semantics, no surprise preconditions.
- Smells: an impl throwing `UnsupportedOperationException` for an operation the API does not document as optional (a **documented** read-only backend is fine — the fennec git backend is the canonical allowed case); overrides that silently narrow accepted inputs; returning null where siblings return empty collections; Promise-based APIs where one impl throws synchronously instead of failing the Promise.
- Check sibling impls of the same interface together (file/apicurio/git storage; the REST format endpoints) — contract drift between them is exactly an LSP finding.

## I — Interface Segregation

- Service APIs should be role-based and small. Smells: interfaces forcing impls into no-op/UOE methods; "god" service interfaces consumed by clients that use 2 of 15 methods.
- Good existing pattern: `ReadOnlyScopeService` vs `WritableScopeService` split. Flag fat interfaces that should follow it.

## D — Dependency Inversion

- Depend on **service interfaces via `@Reference`**; never import another bundle's implementation package. Findings, in decreasing severity:
  - compile-time import of a foreign `*Impl`/internal package (check `Import-Package` in generated manifests and java imports) → blocker;
  - `new`-ing a class that is available as a service (bypasses substitution/config) → major;
  - `Class.forName`/reflection on impl class names → major;
  - static singletons/global registries used instead of services (`EPackage.Registry.INSTANCE` mirroring is a known legacy exception — flag new additions, note existing ones as info).
- Constructor/field injection via DS is the norm; component code should be testable without a framework (plain unit-testable logic separated from lifecycle glue).

## DS/OSGi lifecycle correctness (reviewed alongside SOLID; category `osgi-ds`)

- `deactivate()` must undo `activate()`: unregister listeners/registrations, close resources, remove entries pushed into global/shared registries (a real past bug class in this workspace: EMFFileWatcher leaking EPackages into the global registry).
- No blocking I/O or long work in `activate()` — fail fast or go async (Promises); no swallowed `InterruptedException`.
- Mutable component state must be safe under DS dynamics (references can rebind; components can reactivate). Static mutable fields in components → major.
- Dynamic/optional references handled for the unbind case; greedy/static chosen deliberately.
- Configuration through ConfigAdmin properties (factory/pid), not system properties or hardcoded paths, unless there is an env→prop→default chain by design (the runtime config bundles do this deliberately — don't flag those).
