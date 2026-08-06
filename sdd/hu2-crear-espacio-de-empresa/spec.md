# Espacio CRUD Specification

## Purpose

Manage empresa spaces (espacios) that allow superadmin users to organize work by client. Each espacio has a name and a color identifier for visual distinction.

## Requirements

### Requirement: List espacios

The system SHALL return a list of all active espacios ordered by creation date (newest first).

#### Scenario: List espacios when some exist

- GIVEN there are 2 espacios previously created
- WHEN the user requests GET /api/espacios
- THEN the response contains exactly 2 espacio objects with their id, nombre, color, and activo=true

#### Scenario: List espacios when none exist

- GIVEN no espacios have been created
- WHEN the user requests GET /api/espacios
- THEN the response is an empty array

### Requirement: Create espacio

The system SHALL create a new espacio when provided with a valid nombre and color.

#### Scenario: Create espacio successfully

- GIVEN the user provides "Acme Corp" as nombre and "#FF5733" as color
- WHEN the user submits POST /api/espacios with {nombre, color}
- THEN the response contains the new espacio with id assigned and activo=true
- AND the espacio appears in the listing

#### Scenario: Create espacio without nombre fails validation

- GIVEN the user provides empty string as nombre and "#FF5733" as color
- WHEN the user submits POST /api/espacios with {nombre, color}
- THEN the response is HTTP 400 with a validation error
- AND no espacio is created

#### Scenario: Create espacio with missing color uses default

- GIVEN the user provides "Acme Corp" as nombre with no color
- WHEN the user submits POST /api/espacios with {nombre}
- THEN the response is HTTP 400 with a validation error requiring color
- AND no espacio is created

### Requirement: Edit espacio

The system SHALL update the nombre and/or color of an existing espacio.

#### Scenario: Edit espacio nombre

- GIVEN an espacio with id "abc123" exists with nombre="Old Name"
- WHEN the user submits PUT /api/espacios/abc123 with {nombre: "New Name"}
- THEN the espacio's nombre is updated to "New Name"
- AND the response contains the updated espacio

#### Scenario: Edit espacio color

- GIVEN an espacio with id "abc123" exists with color="#FF5733"
- WHEN the user submits PUT /api/espacios/abc123 with {color: "#33FF57"}
- THEN the espacio's color is updated to "#33FF57"
- AND the response contains the updated espacio

### Requirement: Delete espacio (soft delete)

The system SHALL mark an espacio as inactive rather than removing it from the database.

#### Scenario: Delete espacio sets activo to false

- GIVEN an espacio with id "abc123" exists with activo=true
- WHEN the user submits DELETE /api/espacios/abc123
- THEN the espacio's activo is set to false
- AND the response is HTTP 200
- AND the espacio no longer appears in the listing

#### Scenario: Delete non-existent espacio returns 404

- GIVEN no espacio with id "nonexistent" exists
- WHEN the user submits DELETE /api/espacios/nonexistent
- THEN the response is HTTP 404