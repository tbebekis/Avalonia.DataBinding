# Avalonia Data Binding

## Binding Tabular Data in Avalonia

Avalonia has a powerful binding engine, but tabular runtime data is a demanding use case. This repository is a study and proposal for binding Avalonia controls to data structures such as `DataTable`, `DataView`, and `DataRowView` through a small abstraction layer.

The goal is not to replace Avalonia binding. The goal is to provide a stable binding surface for scenarios where the data model is not a normal CLR object graph with compile-time properties.

ADO.NET tabular objects are still useful in many business applications. They support dynamic schemas, database-shaped data, filtering, sorting, and row-level editing. The problem appears when those dynamic structures are used directly as Avalonia binding sources.

## The Problem

`DataRowView` exposes values through column names, but those column names are not normal CLR properties. In WPF, many applications relied on binding paths that worked well enough against `DataRowView` and `DataView`. In Avalonia, the same approach is not always reliable.

Typical paths such as `[Code]`, `Code`, or `Row[Code]` may read an initial value in some cases, but they do not provide a complete and dependable editing surface. A `DataGrid` bound directly to a `DataView` can also auto-generate columns from the CLR properties of `DataRowView`, instead of the actual columns of the underlying table.

There is a second problem: not all changes come from the control that is currently bound to a field. In real applications, a lookup, locator, command, or service may update several fields directly on the underlying row. The UI still has to refresh correctly when those changes bypass the bound editor.

Snapshot templates can display values by manually reading from `DataRowView`, but that moves the problem elsewhere. Displaying a value once is easy. Keeping grids, editors, current row state, external changes, and master-detail views synchronized is the part that needs a clearer model.

## What This Repository Explores

This repository explores a small `DataSource` layer between Avalonia controls and the underlying data object.

The proposed model is:

- `DataSource` acts as the controller and binding orchestrator.
- `DataSourceRow` acts as the UI-facing row wrapper.
- `IDataProvider` hides the differences between `DataTable`, `DataView`, `DataRowView`, and POCO lists.
- `Avalonia.DataBinding` contains the data-side abstractions.
- `Avalonia.DataBinding.Desktop` contains Avalonia-specific binding helpers.

The important design choice is that Avalonia binds to a stable wrapper object, not directly to raw ADO.NET row objects. The wrapper can raise field notifications consistently, while the provider listens to the underlying source and reports external changes.

## Why a Wrapper Helps

Avalonia binding works best when the binding source exposes stable properties and standard notification behavior. A tabular row is different. Its fields are runtime-defined, and the available names come from table metadata rather than from CLR properties.

`DataSourceRow` gives the UI one predictable object per visible row. It reads and writes values through its provider and exposes field access through an indexer. When a value changes, it raises notifications in one place, whether the change started from the UI or from the underlying data source.

That makes the grid and editor story simpler:

- Editors bind to the current row.
- Grids bind to the visible `DataSourceRow` collection.
- Grid columns bind to fields on `DataSourceRow`.
- Provider notifications are translated into row notifications.
- External row changes can refresh the UI without per-cell manual subscriptions.

## Lookup and Locator Controls

Small reference tables are usually handled well by lookup controls. Examples include countries, currencies, units of measure, tax categories, and other datasets with a limited number of rows. In those cases, the control can load the available values, let the user choose one, and write the selected key back to the bound field.

The same approach does not scale well for large datasets. Customers, products, documents, and similar tables may contain thousands or tens of thousands of rows. Loading all possible values into a lookup is not practical, and the user usually needs a search-oriented UI instead of a simple dropdown.

In this study, a locator is the mechanism used for that larger scenario. A locator presents a search UI, finds one record, and returns the selected key and display values to the bound row.

A locator may appear as a simple editor or as grid columns:

- As a simple editor, it may show one combined control with two or more searchable text boxes.
- For a `CustomerId` field, the locator may display and search by `Customer.Code` and `Customer.Name`.
- In a grid with a hidden `ProductId` field, the locator may display visible columns such as `ProductCode` and `ProductName`.
- When the user selects a record, the locator writes the required id and display fields back to the current row.

This is one of the cases where direct field binding is not enough. A single user action may update more than one underlying field, and those changes may be applied by the locator rather than by the editor currently bound to each field. The binding layer must therefore detect and publish row changes consistently.

## Architecture

The core namespace is `Avalonia.DataBinding`.

It contains:

- `IDataProvider`
- `DataTableProvider`
- `DataViewProvider`
- `ListProvider<T>`
- `DataSource`
- `DataSourceList`
- `DataSourceRow`
- `DataSourceRelation`
- `CascadeDeleteRule`

The Avalonia desktop namespace is `Avalonia.DataBinding.Desktop`.

It contains:

- `DataSourceBinding`
- `DataSourceBindingExtensions`

The split is intentional. The data layer should remain usable and testable without Avalonia controls. The desktop layer can then focus on control binding, grid columns, binding metadata, and disposable binding subscriptions.

## DataSource

`DataSource` owns an `IDataProvider` and exposes the state needed by the UI.

It provides:

- Visible rows through `Rows`.
- All wrapped rows through `AllRows`.
- Current row through `Current`.
- Position and count state.
- Navigation methods such as `MoveFirst()`, `MovePrevious()`, `MoveNext()`, and `MoveLast()`.
- Row creation methods such as `NewRow()`, `AddRow()`, `AppendRow()`, and `AddNew()`.
- Delete methods such as `DeleteCurrent()` and `DeleteRow()`.
- Master-detail relation support.
- Simple field/value filtering.
- Change and position events.

The current implementation also supports detail activation control, generated relation names, string filtering with case-insensitive contains, `prefix*` matching, and cascade delete behavior.

Full expression filtering similar to `DataView.RowFilter` is intentionally left as a later design step.

## DataSourceRow

`DataSourceRow` wraps one underlying item.

It provides:

- Field access by name.
- Read and write operations through the owning provider.
- `INotifyPropertyChanged` support.
- Typed helpers such as `AsString()`, `AsInteger()`, `AsInt32()`, `AsDecimal()`, `AsBoolean()`, and `AsDateTime()`.
- Field notifications for both internal edits and provider-detected external changes.

The wrapper is deliberately small. It is not meant to become a full domain model. It is a binding surface for runtime fields.

## Providers

Providers keep source-specific logic out of `DataSource`.

The current provider set includes:

- `DataTableProvider` for `DataTable`, `DataView`, `DataRowView`, and `DataRow`.
- `DataViewProvider` for explicit `DataView` sources.
- `ListProvider<T>` for POCO lists where `T` implements `INotifyPropertyChanged`.

Provider responsibility is limited to schema discovery, value read/write, item creation, item deletion, and external change notification.

## Avalonia Binding Layer

The desktop layer exposes extension methods on `DataSource`.

The intended usage model is:

- Simple controls bind to `DataSource.Current[FieldName]`.
- Grids bind to `DataSource.Rows`.
- Generated grid columns bind to `DataSourceRow[FieldName]`.
- Binding methods return `DataSourceBinding` objects.
- `DataSourceBinding` owns binding metadata and disposable subscriptions.
- `BindingComplete()` performs the initial current-row synchronization.

This keeps the application code close to the shape of the data while avoiding direct dependency on raw `DataRowView` binding behavior.

## Lifecycle Notes

Grid cells are often virtualized and recycled. A template that manually subscribes to row or provider events must unsubscribe when the visual is detached, otherwise old rows may stay alive longer than expected.

The preferred direction is to avoid per-cell manual subscriptions where native binding can be used. Row notifications should be centralized through `DataSource` and `DataSourceRow`, while `DataSourceBinding` owns the subscriptions it creates for controls and grids.

## Tested Behavior

The current implementation covers:

- Loading and navigation from `DataTable`.
- Loading from filtered and sorted `DataView`.
- Automatic `DataSource.Name` from `DataTable.TableName` and `DataView.Table.TableName`.
- Editing values through `DataSourceRow`.
- Detecting external changes through `DataRowView`, `DataRow`, and POCO notifications.
- Adding and deleting rows.
- Master-detail filtering.
- Generated `DataSourceRelation.Name` from master and detail `DataSource` names.
- `DetailsActive` activation and deactivation.
- Field/value filtering through `SetFilter()` and `CancelFilter()`.
- POCO list filtering, including refresh after POCO property changes.
- `DataSourceList` name lookup, duplicate protection, and owner propagation.
- Restrict and cascade delete behavior.
- Position and change events, including cancellation.
- Desktop binding metadata for controls and grids.
- `BindingComplete()` selecting the first row when current is empty and preserving an existing current row.

## Status

This repository is a study and proposal, not a finished framework claim.

The current direction is promising because it keeps the public model small:

- `DataSource` controls state and navigation.
- `DataSourceRow` is the binding surface.
- Providers isolate source-specific behavior.
- Avalonia-specific helpers stay in `Avalonia.DataBinding.Desktop`.

The next useful step is to keep the sample application focused on the binding problem itself: a grid, a few editors, external row updates, and optionally a small master-detail or lookup scenario if it helps demonstrate why direct `DataRowView` binding is not enough.
