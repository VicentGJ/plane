# Plane Light: Basic Kanban/Scrum Backend Design

This document captures a trimmed-down backend/domain design inspired by Plane, focused only on the basics needed for a lightweight task/project management product with Kanban and Scrum flows.

## Goal

Build a small "Plane Light" that supports:

- Projects
- Basic user roles
- Kanban states/columns
- Issues/tasks
- Assignees
- Basic prioritization and due dates
- Scrum sprints (equivalent to Plane's Cycle)
- Adding/removing issues from sprints
- Board/list filtering

Out of scope for the first version:

- Pages
- Modules
- Intake/triage inbox
- Analytics dashboards
- Notifications
- Integrations
- Realtime collaboration
- Rich text editor internals
- Attachments
- Webhooks
- Saved views
- Favorites/recent visits
- Public spaces
- Imports/exports
- Time tracking
- Issue relations/blockers

---

## Core Domain Model

Plane Light is a self-hosted, single-tenant internal tool. Do not carry Plane's workspace multi-tenancy into the MVP domain. The project is the top-level work container.

The minimum persistence/domain model is:

```text
Project -> State -> Issue
        -> Sprint
        -> Label
Issue   -> Comment
User    -> issues through assignees / authorship
```

Two relationships need many-to-many join tables. These should remain persistence details, not first-class domain entities, unless the join gains its own meaningful fields.

```text
Table             Columns                    Constraints
issue_assignees   issue_id, user_id          unique(issue_id, user_id), index(user_id)
issue_labels      issue_id, label_id          unique(issue_id, label_id), index(label_id)
```

Rules:
- Pairs must be unique (enforced by composite primary key).
- Do not create standalone domain objects for these joins until the relationship gains extra fields like `assigned_by_id`, `applied_by_id`, or `applied_at`.

The many-to-many relationships are listed in the relevant model sections below.

Optional but useful later:

```text
Issue -> Sub-issues via parent_id
```

## Persistence Model Definitions

Use conventional relational naming unless there is a strong reason not to:

- Tables use plural `snake_case`: `projects`, `states`, `issues`.
- Columns use `snake_case`: `project_id`, `created_by_id`, `completed_at`.
- Models use singular PascalCase: `Project`, `State`, `Issue`.
- Prefer explicit enum-like types for closed sets such as roles, state groups, priorities, and sprint status.
- Prefer soft/archive timestamps such as `archived_at` over hard deletes for project-level work records.
- Do not add `workspace_id` or workspace-scoped uniqueness in the MVP; this is a single-tenant app.

### Enums

```ts
type Role = "admin" | "member" | "viewer";

type StateGroup =
  | "backlog"
  | "unstarted"
  | "started"
  | "completed"
  | "cancelled";

type IssuePriority = "urgent" | "high" | "medium" | "low" | "none";

type SprintStatus = "planned" | "active" | "completed";
```

### `users`

Represents an authenticated person. Authorization is handled per-project via `ProjectMember` (see below).

Important columns:

```text
id
name
email
is_active boolean default true
avatar_url?
created_at
updated_at
```

Relationships:

```text
one-to-many Project as ledProjects via lead_id
one-to-many ProjectMember as memberships
many-to-many Issue as assignedIssues via issue_assignees
one-to-many Issue as createdIssues via created_by_id
one-to-many Issue as updatedIssues via updated_by_id
one-to-many Comment as authoredComments via author_id
```

Rules:

- `is_active = false` disables access without deleting historical authorship/assignment references.
- Authorization role is determined by `ProjectMember.role` (see below).

### `projects`

Top-level work container.

Important columns:

```text
id
name
identifier // e.g. WEB, API, OPS
description?: text
lead_id?: FK -> users.id
default_state_id?: FK -> states.id
archived_at?: timestamp
created_at
updated_at
```

Relationships:

```text
many-to-one User as lead
many-to-one State as defaultState
one-to-many ProjectMember
one-to-many State
one-to-many Issue
one-to-many Sprint
one-to-many Label
one-to-many Comment
```

Indexes/constraints:

```text
unique(identifier)
unique(name) // optional; identifier uniqueness is the important one
index(archived_at)
```

Rules:

- `identifier` is user-chosen, persisted, globally unique, and normalized to uppercase.
- `identifier + sequence_id` creates stable work item keys like `WEB-1`.
- Creating a project creates default states.

Default states:

```text
Backlog      -> backlog
Todo         -> unstarted
In Progress  -> started
Done         -> completed
Cancelled    -> cancelled
```

### `project_members`

Links users to projects with a role.

Important columns:

```text
id
project_id FK -> projects.id
user_id FK -> users.id
role Role default member
is_active boolean default true
created_at
updated_at
```

Relationships:

```text
many-to-one Project
many-to-one User
```

Indexes/constraints:

```text
unique(project_id, user_id)
index(project_id, role)
index(user_id)
```

Rules:

- A user can be added to a project once.
- Admins can manage project settings, workflow states, labels, and members.
- Members can create/update issues and comments.
- Viewers can read only.
- `is_active = false` suspends access without removing history.
- Deleting a project member reassigns their issues to the project lead or unassigns them.

### `states`

Kanban column/workflow state.

Important columns:

```text
id
project_id FK -> projects.id
name
slug
color
sequence integer
group StateGroup
is_default boolean default false
created_at
updated_at
```

Relationships:

```text
many-to-one Project
one-to-many Issue
```

Indexes/constraints:

```text
unique(project_id, name)
unique(project_id, slug)
index(project_id, sequence)
index(project_id, group)
```

Rules:

- States are ordered by `sequence`.
- One state can be marked default per project.
- Default state cannot be deleted.
- A state cannot be deleted while issues are in it.
- Issue completion is derived from the state group.

### `issues`

Central task/work item.

Important columns:

```text
id
project_id FK -> projects.id
state_id FK -> states.id
sprint_id?: FK -> sprints.id
parent_id?: FK -> issues.id
sequence_id integer
sort_order numeric
title
description?: text
priority IssuePriority default none
start_date?: date
due_date?: date
completed_at?: timestamp
archived_at?: timestamp
created_by_id FK -> users.id
updated_by_id?: FK -> users.id
created_at
updated_at
```

Relationships:

```text
many-to-one Project
many-to-one State
many-to-one Sprint optional
many-to-one Issue as parent
one-to-many Issue as children via parent_id
many-to-one User as creator via created_by_id
many-to-one User as updater via updated_by_id
many-to-many User as assignees via issue_assignees
many-to-many Label via issue_labels
one-to-many Comment
```

Indexes/constraints:

```text
unique(project_id, sequence_id)
index(project_id, state_id, sort_order)
index(project_id, sprint_id)
index(project_id, priority)
index(project_id, archived_at)
index(created_by_id)
```

Rules:

- If no state is provided, use the project default state.
- `sequence_id` is unique per project and generates issue keys like `WEB-1`.
- `sort_order` controls board ordering inside a state/column.
- Moving to a `completed` state sets `completed_at`.
- Moving out of a `completed` state clears `completed_at`.
- Assignees must be active users.

### `sprints`

Scrum sprint/timebox. Equivalent to Plane's Cycle concept.

Important columns:

```text
id
project_id FK -> projects.id
name
description?: text
start_date?: date
end_date?: date
owner_id FK -> users.id // sprint lead; defaults to creator
status SprintStatus default planned
archived_at?: timestamp
created_at
updated_at
```

Relationships:

```text
many-to-one Project
many-to-one User as owner
one-to-many Issue
```

Indexes/constraints:

```text
index(project_id, status)
index(project_id, start_date, end_date)
```

Rules:

- Sprint belongs to one project.
- Optional rule: only one active sprint per project.
- Completed sprints should be mostly read-only.
- Sprint can only be deleted by project admins or its owner.

### Sprint assignment

For Plane Light, model sprint membership directly on `issues.sprint_id` instead of using a join table.

This keeps the relationship simple:

```text
Sprint one-to-many Issue
Issue many-to-one Sprint optional
```

Rules:

- An issue can belong to zero or one sprint at a time.
- Issue and sprint must belong to the same project.
- Moving an issue between sprints is a normal issue update: change `sprint_id`.
- Use a join table only if the product later needs sprint history, spillover tracking, or many-to-many sprint planning metadata.

### `labels`

Project-scoped reusable tags.

Important columns:

```text
id
project_id FK -> projects.id
name
color
created_at
updated_at
```

Relationships:

```text
many-to-one Project
many-to-many Issue via issue_labels
```

Indexes/constraints:

```text
unique(project_id, name)
```

### `comments`

Important columns:

```text
id
project_id FK -> projects.id
issue_id FK -> issues.id
author_id FK -> users.id
body text
created_at
updated_at
```

Relationships:

```text
many-to-one Project
many-to-one Issue
many-to-one User as author
```

### Minimal Domain UML

```mermaid
classDiagram
    class User
    class Project
    class ProjectMember
    class State
    class Issue
    class Sprint
    class Label
    class Comment

    User "0..1" -- "0..*" Project : leads
    User "1" -- "0..*" ProjectMember : member_of
    Project "1" *-- "0..*" ProjectMember : has

    Project "1" *-- "1..*" State : workflow
    Project "1" -- "0..1" State : default
    Project "1" *-- "0..*" Issue : contains
    State "1" -- "0..*" Issue : current state
    Issue "0..1" -- "0..*" Issue : parent of
    User "1" -- "0..*" Issue : creates
    User "0..1" -- "0..*" Issue : updates

    Issue "0..*" -- "0..*" User : assignees

    Project "1" *-- "0..*" Sprint : contains
    User "1" -- "0..*" Sprint : owns
    Sprint "0..1" -- "0..*" Issue : includes

    Project "1" *-- "0..*" Label : contains
    Issue "0..*" -- "0..*" Label : labels

    Project "1" *-- "0..*" Comment : contains
    Issue "1" *-- "0..*" Comment : comments
    User "1" -- "0..*" Comment : authors
```

---

## Backend Application Design

Use a conventional server-rendered or API-backed web application. Keep the design close to the defaults of the chosen stack to reduce boilerplate:

- **Entities/models**: first-class domain records for projects, states, issues, sprints, labels, comments, users, and project membership.
- **Controllers/handlers**: thin request handlers for resource-oriented CRUD and domain actions.
- **Validation/authorization**: explicit request validation plus project-scoped authorization checks.
- **Serialization/view models**: a clear response boundary for shaping data sent to the UI.
- **UI**: pages/views organized around product screens, not around database tables.
- **Workflow logic**: small service/action functions only when controller code would become unclear.

### Structure Approach

Do not prescribe a custom folder structure up front. Start from the structure generated by the chosen framework or application bootstrap path.

Add application-specific classes/modules only where they make the design clearer:

- Domain entities/models for first-class records.
- Resource-oriented controllers/handlers for CRUD-like resources.
- Request validators/schemas when validation grows beyond simple cases.
- Authorization rules for role-level and model-scoped decisions.
- Response serializers/view models when UI/API shape needs to be explicit.
- Small action/service functions only when a workflow becomes too large for a controller/handler method.

The goal is to lean on generated framework structure, not lock the application into a hand-designed directory layout before implementation.

### Routing Approach

Use resource-oriented routes for normal CRUD and add named custom routes only for domain actions.

```text
/projects
  -> ProjectController resource

/projects/{project}/members
  -> ProjectMemberController resource

/projects/{project}/states
  -> StateController resource
  + POST states/reorder
  + POST states/{state}/default

/projects/{project}/issues
  -> IssueController resource
  + POST issues/{issue}/move
  + PUT issues/{issue}/assignees

/projects/{project}/board
  -> BoardController read endpoint/page

/projects/{project}/sprints
  -> SprintController resource
  + POST sprints/{sprint}/start
  + POST sprints/{sprint}/complete
  + PUT sprints/{sprint}/issues

/projects/{project}/labels
  -> LabelController resource

/projects/{project}/issues/{issue}/comments
  -> CommentController resource
```

Resolve `State`, `Issue`, `Sprint`, `Label`, and `Comment` inside the current project context so nested resources cannot leak across projects.

### Controller Responsibilities

| Controller | Role |
| --- | --- |
| `ProjectController` | Project CRUD; create default states on project creation. |
| `ProjectMemberController` | Project member CRUD and role management. |
| `StateController` | Workflow column CRUD, default state, and state ordering. |
| `IssueController` | Issue CRUD, filters, assignment sync, and issue movement. |
| `BoardController` | Read-only board page/read model grouped by state. |
| `SprintController` | Sprint CRUD, start/complete lifecycle, and issue membership sync. |
| `LabelController` | Project-scoped label CRUD. |
| `CommentController` | Issue comment CRUD. |

Mutation controllers/handlers should usually validate input, authorize the current user's project role, perform the write, and return either a redirect or a serialized response depending on the chosen UI architecture.

### Response Serialization

Use a clear serialization boundary for both JSON responses and server-rendered page props.

Recommended response/view models:

```text
UserView
ProjectView
StateView
IssueView
BoardView
SprintView
LabelView
CommentView
```

Guidelines:

- Shape lists consistently at the response boundary.
- Include related records intentionally (`assignees`, `labels`, `states`, `issues`) instead of relying on accidental lazy loading.
- Include counts like comments, issue totals, or sprint progress when they are part of the screen contract.
- Compute presentation fields like issue key (`WEB-123`) in the response/view model.
- Do not put authorization, validation, or mutation logic in response serializers.

---

## Core Business Logic to Preserve from Plane

### 1. Project scoping everywhere

The persistence model definitions make `project_id` the main work boundary. Preserve that boundary in every query, authorization check, route parameter resolution, and mutation so states, issues, sprints, labels, and comments cannot leak across projects.

### 2. Project-local issue keys

Plane's `projects.identifier + issues.sequence_id` pattern is worth keeping.

```text
WEB-1
WEB-2
WEB-3
```

This is much more user-friendly than exposing UUIDs.

### 3. State groups, not just columns

Separate custom workflow names from semantic meaning.

Example:

```text
Backlog      group backlog
Todo         group unstarted
Doing        group started
Code Review  group started
Done         group completed
Won't Do     group cancelled
```

This lets users customize workflows without breaking completion/progress logic.

### 4. Completion derived from state group

Do not store independent `isDone` unless necessary.

```text
if issue.state.group == completed:
  completed_at = now
else:
  completed_at = null
```

### 5. Sparse sort ordering

Use sparse numbers for drag/drop ordering.

Initial examples:

```text
10000
20000
30000
```

Insert between two cards:

```text
new_sort_order = (before.sort_order + after.sort_order) / 2
```

Periodically normalize if numbers get too dense.

### 6. Safe workflow mutations

Rules:

```text
Cannot delete default state
Cannot delete state with issues
Cannot delete project without admin rights
Cannot assign inactive user to issue
Cannot add issue to sprint from another project
```

### 7. Keep filters simple

Initial filters:

```text
state_id
assignee_id
priority
sprint_id
search
created_by_id
due_date
```

Avoid saved views/advanced filter DSL until the basic product is solid.

---

## Minimal Route Surface

The concrete route shape should follow the routing approach above rather than maintaining a second exhaustive API map. The minimum route surface is:

```text
Auth/session routes from the chosen application framework
Project resource routes
Nested project member resource routes
Nested state resource routes plus default/reorder actions
Nested issue resource routes plus move/assignee-sync actions
Project board route
Nested sprint resource routes plus start/complete actions
Label resource routes
Nested comment resource routes
```

---

## Minimal UI Views

If translating this into frontend/product screens:

### Home

```text
Project list
User/account menu
```

### Project

```text
Project settings
Label settings
```

### Kanban

```text
Project board
Backlog/list
Issue detail drawer/page
```

### Scrum

```text
Sprint list
Sprint planning/backlog
Sprint board
Sprint summary
```

### Settings

```text
Workflow/state settings
Labels
```

---

## Suggested Build Order

### Phase 1: Kanban core

Domain models:

```text
User
Project
ProjectMember
State
Issue
Label
Comment
```

Relationship tables:

```text
issue_assignees
issue_labels
```

Features:

```text
Create project
Create default states
Create issue
Move issue across board
Assign issue
Label issue
Comment on issue
Filter board
```

### Phase 2: Scrum

Domain models:

```text
Sprint
```

Features:

```text
Create sprint
Assign issues to sprint
Sprint board
Start/complete sprint
Basic progress count
```

### Phase 3: Collaboration polish

Models/features:

```text
Activity log
```

### Phase 4: Product niceties

Models/features:

```text
Saved views
Notifications
Attachments
Subtasks
Issue relations
```

---

## Summary

The essence of Plane for basic Kanban/Scrum is:

```text
Project
ProjectMember
State
Issue
Label
Comment
Sprint
```

With these persistence-only join tables:

```text
issue_assignees
issue_labels
```

The most valuable Plane ideas to replicate are:

1. Project scoping
2. Custom states grouped by semantic state groups
3. Project-local issue sequence IDs
4. Sparse sort ordering for drag/drop
5. Sprints (equivalent to Plane's Cycles)
6. Issue completion derived from state group
7. Simple role-based permissions

This gives a small, coherent backend domain while preserving the parts of Plane that make basic task/project workflows work.
