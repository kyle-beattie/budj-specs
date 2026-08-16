## ADDED Requirements

### Requirement: Components take values and closures, and know nothing about the server

A component in the design system layer SHALL receive everything it displays as
properties and SHALL communicate outwards through closures or bindings. It MUST
NOT read a model from the environment, perform a request, or import the
networking layer.

#### Scenario: A component compiles and previews without the app running

- **WHEN** any design system component is rendered in a preview
- **THEN** it displays fully from literal values, with no network, session, or
  model present

#### Scenario: The layer boundary is enforced by inspection

- **WHEN** the design system layer's imports are inspected
- **THEN** no file references the API client, the session store, or a feature
  model

### Requirement: One type per file, and no components defined as computed properties

Each view, modifier, and style SHALL live in its own file named after it. A view
body MUST NOT be decomposed into properties or functions returning `some View`
where the fragment is a nameable component with its own states.

#### Scenario: A file name identifies a component

- **WHEN** the source tree is listed
- **THEN** each component's name is a file name, and no file declares two view
  types

#### Scenario: A reusable fragment is extracted as a type

- **WHEN** a portion of a view has a name, states, and would benefit from a
  preview
- **THEN** it exists as its own `View` struct in its own file rather than as a
  computed property

### Requirement: Visual values come from named tokens, not literals

Colour, spacing, corner radius, typography, and motion SHALL be referenced
through named token members. Numeric and colour literals MUST NOT appear in
feature views.

#### Scenario: No raw values in feature code

- **WHEN** a feature view is inspected
- **THEN** it contains no hard-coded colour, no raw corner radius, and no padding
  value outside the spacing scale

#### Scenario: Colour resolves from the asset catalogue

- **WHEN** a colour token is resolved
- **THEN** it reads from the asset catalogue so that light and dark appearances
  are both defined in one place

### Requirement: Repeated visual treatments are modifiers, not copied blocks

A treatment applied to more than one component SHALL be expressed as a
`ViewModifier` exposed through a `View` extension — card surface, glass surface,
and press feedback are all such treatments.

#### Scenario: A surface treatment is applied by one modifier

- **WHEN** two components need the same card surface
- **THEN** both apply the same modifier and neither restates its background,
  border, or shadow

### Requirement: Onboarding screens share one scaffold

Every onboarding screen SHALL be built on a single shared scaffold component
providing the title, body slot, and action area, so that spacing, safe areas, and
action placement are defined once.

#### Scenario: A new step inherits the layout

- **WHEN** a further onboarding screen is added
- **THEN** it supplies a title, content, and actions to the scaffold and defines
  no layout of its own

### Requirement: Every component is previewable in every meaningful state

Each component SHALL have previews covering its states, and a gallery screen SHALL
render the whole component set in one place in debug builds.

#### Scenario: States are previewed, not just the happy one

- **WHEN** a component has loading, disabled, error, or selected states
- **THEN** each appears in a preview

#### Scenario: The gallery renders the full set

- **WHEN** the component gallery is opened in a debug build
- **THEN** every design system component is displayed with its states

#### Scenario: The gallery is not reachable in release builds

- **WHEN** a release build is inspected
- **THEN** no route reaches the gallery

### Requirement: Components meet the accessibility floor before they are considered finished

Every component SHALL support Dynamic Type without truncation or clipping at
accessibility sizes, carry text labels on all controls, and avoid conveying state
by colour alone.

#### Scenario: Accessibility type sizes do not break layout

- **WHEN** the gallery is viewed at the largest accessibility type size
- **THEN** no component clips, truncates a label, or overlaps another element

#### Scenario: An icon-only control still has a label

- **WHEN** a control is visually icon-only
- **THEN** it carries a text label for VoiceOver, hidden visually rather than
  omitted

#### Scenario: Tappable elements are buttons

- **WHEN** an element responds to a tap
- **THEN** it is a button rather than a view with a tap gesture

#### Scenario: State is not colour alone

- **WHEN** a component signals a state such as active, connected, or failed
- **THEN** a symbol, shape, or text difference accompanies the colour change

### Requirement: Motion is value-driven and respects Reduce Motion

Animations SHALL be attached to a specific value and SHALL degrade to opacity
changes when Reduce Motion is enabled.

#### Scenario: Animations name the value they respond to

- **WHEN** an animation is applied
- **THEN** it specifies the value driving it rather than animating implicitly

#### Scenario: Reduce Motion replaces movement with fading

- **WHEN** Reduce Motion is enabled
- **THEN** step transitions and surface presentations cross-fade instead of
  sliding or scaling
