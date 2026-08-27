# Access Model

The access model has three layers:

1. the graph substrate, which stores knowledge in one shared graph;
2. the Project access plane, which decides which source coordinates an actor receives;
3. the admin plane, which decides who may manage the system.

This is the canonical shape after [ADR-20](../decisions/0020-project-access-and-wheel-elevation.md).
ADR-16 remains the substrate: identity is verified, scopes and capabilities are derived by the
kernel, and reads are default-deny.

## Graph Substrate

The kernel keeps one graph. It does not create a graph per user, project, tenant, or connector.
Isolation is carried by two independent coordinates on graph rows:

| Coordinate | Meaning | Examples |
| --- | --- | --- |
| `scope` | Which source/security coordinate produced or owns the row | `public`, `src:<source_uuid>` |
| `owner` | Which user owns a private row | `NULL` for shared rows, a user UUID for owned rows |

The visibility rule is one predicate:

```text
visible ⇔ scope ∈ actor.effective_scopes
       AND (owner IS NULL OR owner = actor.id)
```

The kernel applies this rule at the single access gate, default-deny, and on both endpoints of an
edge. An empty effective-scope set discloses nothing. User-facing queries never run in root
context.

## Projects

A Project is the data-access and sharing container. Users share Projects, not raw source scopes.

A Project owns Sources. A Source is the security-bearing ingest coordinate behind one connector or
source instance. Its stable identifier, not its human label, becomes the graph scope:

```text
Project -> Source -> Connector -> graph rows with scope = src:<source_uuid>
```

Human names such as `wiki`, `confluence`, or `ldap` are labels and routing hints. They are not
security keys; two different Confluence connectors must have different source scopes.

Effective data access is derived like this:

```text
actor -> user/group memberships
      -> Project memberships
      -> Project Sources
      -> effective source scopes
      -> graph gate
```

Groups do not grant source visibility directly. A group can be a Project member; users in that
group then inherit that Project's source scopes through the Project.

Direct user membership in a Project is allowed, but it is explicit audited Project membership, not
a hidden per-user grant outside the Project model.

## Project Visibility Words

`personal`, `shared`, and `public` are product classifications for Projects, not a second access
system:

| Term | Meaning |
| --- | --- |
| `personal` | A Project used by one user. Its private rows may carry `owner = actor`. |
| `shared` | A Project shared with users or groups through explicit membership. Project-shared source data usually has `owner = NULL`. |
| `public` | A Project or source class whose rows enter the `public` baseline. This is intentional public material, not an accidental result of membership count. |
| `own` | Not a visibility class. It means `owner = actor` on the owner axis. |

Project membership controls source scopes. The owner axis controls private user rows inside any
Project shape.

## Groups And Roles

The fixed groups are:

| Group | Purpose |
| --- | --- |
| `Wheel` | Local-only break-glass cohort. Members may elevate after fresh re-authentication. |
| `Admins` | Operators who can manage ordinary users, Projects, and connectors within the admin capability boundary. |
| `Staff` | Default internal cohort. Confers no role by itself; receives data access only through Project membership. |

Roles and data visibility are separate:

- roles confer capabilities;
- Project membership confers source visibility;
- no role makes private or out-of-project data visible by itself.

`Admins` cannot modify `Wheel`, role bindings, auth/elevation/audit controls, or publicness. Those
operations require an active elevation.

## Elevation

There is no standing `superadmin` group role. `superadmin` is an active elevation session:

- only local `Wheel` members may request it;
- fresh re-authentication is required;
- a reason is required;
- the session is time-boxed;
- an audit row is written before the elevated capability takes effect.

Data break-glass is still per-operation: the target, reason, request id, effective lens, and
decision are audited before data is returned. Elevation is not an ambient root read path.

## Guests

A guest is a normal user with `external = true`. A guest is not a member of `Staff` by default and
receives no internal source visibility unless explicitly added to a Project.

Guests may eventually attach their own connectors. If a connector is private to the guest, its
rows carry `owner = guest`. If the connector is shared through a Project, its shared rows use the
Project source scope and normally `owner = NULL`.

## Service Context

System tasks such as ingestion, enrichment, consolidation, and maintenance can run with broader
service authority than a user lens. Their outputs must inherit the source and owner coordinates of
their inputs. Scope widening is never implicit. Cross-source or cross-project merges are refused
unless an audited reclassification explicitly makes the result safe.

## Superseded Patterns

ADR-18's per-source scope substrate remains. Its direct `group -> src:*` grant mechanism is
superseded by Project-derived source access.

ADR-19's group names and standing group-derived `superadmin` mechanism are superseded by
`Wheel` / `Admins` / `Staff` and time-boxed elevation.
