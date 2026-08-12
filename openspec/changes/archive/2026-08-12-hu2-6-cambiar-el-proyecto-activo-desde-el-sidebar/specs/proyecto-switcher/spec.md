# ProyectoSwitcher Specification

## Purpose

Add a `ProyectoSwitcher` dropdown to the sidebar, enabling project context switching without full page reload. Follows the `EspacioSwitcher` pattern.

## Requirements

### Requirement: Multi-Project Dropdown Mode

When the user has more than one active project, the system SHALL display a dropdown switcher in the sidebar that shows all active projects with their name, espacio, and color.

#### Scenario: Open switcher with multiple projects

- GIVEN the user has 2 or more active projects
- WHEN the user clicks the ProyectoSwitcher button in the sidebar
- THEN a dropdown opens displaying each project with its name, associated espacio, and color indicator

#### Scenario: Select project from dropdown

- GIVEN the dropdown is open showing multiple projects
- WHEN the user clicks on a project entry
- THEN the system navigates to `/proyectos/{id}/casos` preserving dashboard context without full page reload

---

### Requirement: Single-Project Static Mode

When only one active project exists, the system SHALL display a static header instead of a dropdown.

#### Scenario: Single project shows static header

- GIVEN only one active project exists
- WHEN the sidebar renders
- THEN the ProyectoSwitcher displays as a static header showing the project name and color
- AND no dropdown button is rendered

---

### Requirement: Project Context Navigation

The system SHALL change all visible project-scoped UI (scope-bar, casos, ejecuciones) when a project is selected.

#### Scenario: Project context updates after selection

- GIVEN the user is viewing the dashboard in Project A context
- WHEN the user selects Project B from the switcher
- THEN the sidebar, scope-bar, and main content update to show Project B data
- AND the URL changes to `/proyectos/{B_id}/casos`

---

### Requirement: Active Project Detection

The system SHALL detect the active project from the URL pattern `proyectos/[id]/...`.

#### Scenario: Active project highlighted in dropdown

- GIVEN the user is at URL `/proyectos/project-123/casos`
- WHEN the dropdown renders
- THEN the project with id `project-123` is shown as selected/active in the list
