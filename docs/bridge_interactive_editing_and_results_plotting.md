# Implementation Approach and Contribution Report

## OsdagBridge: Interactive CAD Editing, Input Safety, and Transverse Result Plotting

## 1. Introduction

As part of the OsdagBridge internship work, we extended the existing bridge
design application in three areas:

1. Interactive editing of important bridge parameters from the 2D CAD canvas.
2. Safer validation and state management for the Additional Inputs dialog.
3. Post-analysis plotting of transverse bridge members.

The work was implemented by studying and extending the existing UI, bridge
input, result extraction, and plotting layers. The approach was to integrate
with the existing architecture and data flow instead of creating separate
parallel logic for the new features.

The main contribution was therefore not only adding visible controls, but also
connecting those controls to the existing input dictionaries, central key
definitions, bridge model, analysis dataset, and standard OsdagBridge dialogs.

## 2. Existing System Understanding

Before making changes, we traced the existing flow of a typical bridge design:

1. The Basic Inputs and Additional Inputs panels collect user values.
2. The values are stored using keys defined in
   `src/osdagbridge/core/utils/common.py`.
3. The template page passes those values to the bridge type implementation.
4. The bridge type creates the grillage/OpenSeesPy model and performs analysis.
5. The CAD widgets visualize the generated geometry.
6. The result-data and plotting modules read the analysis output and display
   forces, moments, and displacements in the Output Dock.

This investigation identified the appropriate extension points:

- CAD widget mouse events for direct parameter editing.
- The Additional Inputs dialog's existing Save and close lifecycle methods for
  validation and warning behavior.
- `results_data.py` for node/element interpretation.
- `plot_generator.py` for rendering result diagrams.
- `output_dock.py` for exposing the new member selection to users.

This kept the changes localized and allowed the new behavior to reuse existing
signals, dictionaries, validators, datasets, and message-box components.

## 3. Task 1: Interactive 2D CAD Parameter Editing

### 3.1 Existing limitation

The 2D CAD canvas displayed dimension labels such as Span Length, Carriageway
Width, Girder Spacing, and Deck Thickness, but those labels were read-only.
Users had to leave the CAD view and manually find the corresponding input field
in the Basic or Additional Inputs panel.

The CAD labels also included derived values such as Overhang, Overall Bridge
Width, and Bracing Spacing. These values are calculated from other parameters
and should not be edited independently.

### 3.2 Implementation approach

We implemented inline editing at the CAD-widget level. Each CAD view maintains
editable regions containing:

- The screen rectangle occupied by a label.
- The central input key associated with that label.
- The current displayed value.

The `mouseDoubleClickEvent` checks whether the pointer is inside one of these
registered editable regions. If it is, the widget creates a temporary
`QLineEdit` positioned over the label. This gives the user an editing control
without permanently changing the existing CAD layout.

The input editor is connected to the existing editing-finished behavior. When
the user presses Enter or moves focus outside the editor, the value is parsed
and validated. A valid value is written back through the existing update
signal/dictionary flow, and the CAD view redraws using the new geometry.

### 3.3 Validation and locked values

The editor accepts only positive numeric values. Empty text, non-numeric text,
zero, and negative values are rejected while keeping the editor available for
correction.

Only these parameters were registered as editable:

- Carriageway Width
- Girder Spacing
- Deck Thickness
- Span Length

Overhang, Overall Bridge Width, and Bracing Spacing were deliberately excluded
from the editable-region lists. This is an important design decision because
they are derived values. Keeping them read-only prevents the CAD interaction
from creating conflicting values in the bridge input model.

### 3.4 UI improvement

During testing, the first inline editor did not provide enough contrast against
the drawing. We improved its usability by adding:

- A white background.
- Dark text.
- A green border matching the application theme.
- Padding and a minimum width.
- Centered text and automatic selection of the current value.

This makes the active editing state obvious and ensures that the typed value is
visible even when the underlying CAD drawing is detailed.

### 3.5 Files modified

- `src/osdagbridge/desktop/ui/docks/cad_cross_section.py`
- `src/osdagbridge/desktop/ui/docks/cad_top_view.py`
- `src/osdagbridge/desktop/ui/docks/cad_dual_view.py`
- `src/osdagbridge/desktop/ui/template_page.py`

The implementation uses the existing central keys from
`src/osdagbridge/core/utils/common.py` rather than introducing duplicate
parameter names.

### 3.6 Result

The 2D CAD canvas became an active input surface. A user can now double-click a
supported dimension, change its value, and immediately see the updated value
propagated to the main UI and drawing.

## 4. Task 2: Additional Inputs Validation and State Management

### 4.1 Existing limitation

The Additional Inputs dialog supported several input modes, including Custom
values. Previously, a user could select Custom, leave its value field empty,
and attempt to save. The dialog also did not clearly protect staged edits when
the user closed it without saving.

This could result in incomplete input data or accidental loss of user work.

### 4.2 Implementation approach for Custom validation

We intercepted the existing Save-button flow rather than adding a second save
mechanism. Before the normal save operation continues, the dialog now scans all
enabled tabs.

The validation logic identifies mode controls through their existing central
key convention. For a mode/value pair, the mode control uses the `.mode`
suffix and the associated value control uses the `.value` suffix. When the mode
is `Custom`, the paired value is checked for non-empty content.

The implementation also handles adaptive controls whose visible page changes
based on the selected design mode. This ensures that both ordinary Custom
fields and dynamically displayed Custom fields receive the same validation.

When an empty Custom value is found:

1. The save operation is stopped.
2. The standard OsdagBridge message-box wrapper is opened.
3. The error message includes the human-readable parameter label.
4. Focus moves to the field that needs correction.

The labels are derived from `ADDITIONAL_INPUTS_SCHEMA`, while field identity is
derived from the central keys/object names. This avoids hardcoding a separate
list of parameter strings in the validation code.

### 4.3 Implementation approach for unsaved changes

We added an `is_modified` flag to the dialog. It is updated by connecting to
the existing change signals of:

- `QLineEdit`
- `QComboBox`
- `QCheckBox`

When the dialog is populated programmatically, a suppression flag prevents
those initialization operations from being treated as user edits.

The dialog's `closeEvent` was overridden. If no edits are pending, the normal
close behavior is preserved. If edits are pending, the standard warning dialog
asks whether the user wants to Discard or Cancel.

Choosing Cancel ignores the close event and keeps the dialog open. Choosing
Discard accepts the close and clears the modified state. A successful Save also
clears the modified state because the staged values have been committed.

### 4.4 Files modified

- `src/osdagbridge/desktop/ui/dialogs/additional_input/additional_inputs.py`

The implementation reuses the existing `CustomMessageBox`, message types,
schema definitions, and central key system.

### 4.5 Result

The Additional Inputs dialog now protects both data quality and user effort:

- Empty Custom values cannot be saved.
- Error messages identify the missing parameter.
- Unsaved changes are clearly reported before closing.
- Users can either return to the dialog or intentionally discard changes.

## 5. Task 3: Transverse Member Result Plotting

### 5.1 Existing limitation

The result plotting workflow was primarily organized around longitudinal
girder members. Although the analysis model contains transverse components such
as end diaphragms and intermediate cross members, they were not exposed as one
selectable member group in the plotting UI.

This limited the user's ability to inspect the behavior of the complete bridge
system after analysis.

### 5.2 Implementation approach

We extended the existing result path at four connected points:

1. Result-data extraction identifies transverse element tags.
2. The bridge wrapper obtains node and member information through the result
   data module.
3. The plotting module creates diagrams for the selected transverse elements.
4. The Output Dock exposes `Transverse Members` in the member dropdown.

This approach keeps the member-selection label shared between the UI and plot
logic, preventing a mismatch where the UI offers an option that the plotting
code does not understand.

### 5.3 Member classification

The element classification is implemented in
`src/osdagbridge/core/bridge_types/plate_girder/results_data.py`.

For every two-node element, the end-node coordinates are inspected:

- If the element's end nodes maintain the same transverse coordinates, it is
  classified as a longitudinal element.
- Otherwise, it is classified as a transverse element.

The classification is based on the live model's connectivity and coordinates,
not on a hardcoded element-number range. This is important because element
tags can change when the model layout or number of girders changes.

The shared label `Transverse Members` is defined in `results_data.py` and
imported by both the Output Dock and plotting code.

### 5.4 Plot generation

The plotting module now has a transverse-member path that:

1. Reads the selected load case from the analysis dataset.
2. Finds the requested result component.
3. Iterates over the classified transverse elements.
4. Maps each element's end nodes to its plotted coordinates.
5. Draws the selected result diagram.
6. Calculates and displays maximum and minimum values.

The supported components are:

- Forces: `Fx`, `Fy`, `Fz`
- Moments: `Mx`, `My`, `Mz`
- Displacements: `Dx`, `Dy`, `Dz`

No separate UI categories were introduced for chords, diagonals, or individual
cross-bracing subtypes. They are represented through the single transverse
member selection requested for this task.

### 5.5 Files modified

- `src/osdagbridge/core/bridge_types/plate_girder/results_data.py`
- `src/osdagbridge/core/bridge_types/plate_girder/plategirderbridge.py`
- `src/osdagbridge/core/bridge_types/plate_girder/plot_generator.py`
- `src/osdagbridge/desktop/ui/docks/output_dock.py`

### 5.6 Result

After analysis, the user can select `Transverse Members`, choose a load case,
and inspect force, moment, or displacement diagrams. The plot title and
summary identify the selected component and provide maximum/minimum values.

## 6. Integration and Design Decisions

### Reuse of existing architecture

The changes were designed around existing application boundaries rather than
duplicating behavior:

- CAD edits use the existing input-update and redraw flow.
- Additional Inputs uses existing schema definitions and message boxes.
- Result plotting uses the existing analysis dataset and plotting framework.
- The Output Dock remains the central place for result selection.

### Centralized keys and labels

The application already defines central keys for bridge inputs. We retained
that design and used schema-derived labels for error messages. For Task 3, the
new member-group label is defined once in `results_data.py` and reused by all
consumers.

### Protection of derived values

The CAD editor intentionally exposes only independent parameters. Derived
dimensions remain display-only, which avoids mathematical conflicts during
redraw and preserves the existing bridge geometry rules.

### Compatibility with variable bridge models

Transverse-member identification uses node coordinates and connectivity rather
than fixed element numbers. This allows the same logic to work across bridge
models with different spans, girder counts, and generated element tags.

## 7. Verification Performed

### Functional verification

The following behaviors were manually verified in the application:

- Editable CAD labels open a visible inline input box.
- Valid CAD values update the input display and redraw the geometry.
- Locked CAD labels do not open an editor.
- Empty Custom values stop the Additional Inputs save operation.
- The validation message names the affected parameter.
- Closing the Additional Inputs dialog with edits shows the warning dialog.
- Cancel preserves the dialog; Discard closes it.
- `Transverse Members` appears in the result member dropdown.
- A transverse `Fy` diagram is displayed for the selected `SW` load case.
- The transverse plot includes a max/min summary.

### Static verification

The modified Python modules were checked using:

```powershell
python -m compileall src/osdagbridge/core/bridge_types/plate_girder/results_data.py src/osdagbridge/core/bridge_types/plate_girder/plategirderbridge.py src/osdagbridge/core/bridge_types/plate_girder/plot_generator.py src/osdagbridge/desktop/ui/docks/output_dock.py src/osdagbridge/desktop/ui/dialogs/additional_input/additional_inputs.py
```

Runtime verification must use the project's configured environment because the
application depends on PySide6, OpenSeesPy, ospgrillage, and related bridge
analysis packages.

## 8. Summary of Contribution

The contribution extended OsdagBridge from a primarily panel-driven workflow
into a more interactive and analysis-aware application:

- The 2D CAD view now supports direct editing of independent bridge
  parameters.
- The input dialog now prevents incomplete Custom data and accidental data
  loss.
- The result viewer now includes transverse bridge-member behavior alongside
  longitudinal girder results.

These changes were integrated into the existing UI and analysis architecture,
with attention to central keys, reusable data extraction, derived-value
protection, and consistency with the application's existing user experience.

