# Backoffice Configuration

## Purpose

Provide practical, repeatable patterns for configuring SAP Commerce Backoffice (cockpitng/ZK) without forking platform code.
This is a reference for common configuration tasks: editor area layout, list view columns, search fields, actions, wizards,
custom editors/renderers, labels, and deep links.

## Key Concepts

- Backoffice is built on the cockpitng framework (ZK-based widgets).
- Most customization is declarative via XML under `resources/backoffice/` and `resources/cockpitng/`.
- Configurations are merged across extensions; later/high-priority configs override earlier ones.
- Prefer configuration over custom widgets; introduce custom Java/ZUL only when a declarative option does not exist.

## Where Configuration Lives

Typical locations inside a custom extension:

```
<extension>/resources/
  backoffice/
    labels/
      labels_en.properties
    <extension>-backoffice-config.xml
  cockpitng/
    ... (cockpitng config fragments if needed)
  <extension>-backoffice-spring.xml
```

Common file roles:

- `*-backoffice-config.xml`: contexts for editor area, list view, search, wizards, actions.
- `backoffice/labels/*.properties`: UI labels (tabs/sections/action titles/validation messages).
- `*-backoffice-spring.xml`: Spring beans for custom renderers/editors/action handlers.

## Configuration Merge and Override

Backoffice configuration is merged from platform + all loaded extensions.
When multiple configs target the same context, the effective result depends on merge priority/order.

Practical guidance:

- Keep your customization in a dedicated extension (e.g., `mycompanybackoffice`).
- Avoid copying whole platform contexts; override the smallest fragment needed.
- Name labels with a stable prefix (e.g., `mycompany.backoffice.*`) to reduce collisions.

## Editor Area Customization

Editor area configuration uses the `editorArea` component.

### Example: Tabs, Sections, Attributes

```xml
<?xml version="1.0" encoding="UTF-8"?>
<config xmlns="http://www.hybris.com/cockpit/config"
        xmlns:y="http://www.hybris.com/cockpit/config/hybris">

	<context type="Product" component="editor-area">
		<editorArea:editorArea xmlns:editorArea="http://www.hybris.com/cockpitng/component/editorArea">
			<editorArea:tab name="mycompany.backoffice.product.tab.general" position="0">
				<editorArea:section name="mycompany.backoffice.product.section.essential">
					<editorArea:attribute qualifier="code" readonly="true"/>
					<editorArea:attribute qualifier="name"/>
					<editorArea:attribute qualifier="description" editor="com.hybris.cockpitng.editor.localized"/>
					<editorArea:attribute qualifier="approvalStatus"/>
					<editorArea:attribute qualifier="catalogVersion" readonly="true"/>
				</editorArea:section>
			</editorArea:tab>

			<editorArea:tab name="mycompany.backoffice.product.tab.classification" position="1">
				<editorArea:section name="mycompany.backoffice.product.section.classification">
					<editorArea:attribute qualifier="classificationClasses"/>
				</editorArea:section>
			</editorArea:tab>
		</editorArea:editorArea>
	</context>

</config>
```

### Hide or Make Readonly

```xml
<editorArea:attribute qualifier="internalNotes" visible="false"/>
<editorArea:attribute qualifier="creationtime" readonly="true"/>
```

Guidelines:

- Mark system fields readonly (e.g., `code`, `creationtime`, `modifiedtime`) if users should not change them.
- Prefer hiding fields over removing them from type system when the attribute is still part of the API.

## Search Configuration

Backoffice commonly uses simple (quick) search and advanced search contexts.

### Simple Search (Quick Search)

```xml
<context type="Product" component="simple-search">
	<simpleSearch:simpleSearch xmlns:simpleSearch="http://www.hybris.com/cockpitng/config/simpleSearch">
		<simpleSearch:field name="code" selected="true"/>
		<simpleSearch:field name="name" selected="true"/>
		<simpleSearch:field name="ean"/>
		<simpleSearch:field name="catalogVersion" operator="equals" selected="true"/>
		<simpleSearch:sort name="name" ascending="true" default="true"/>
		<simpleSearch:sort name="code" ascending="true"/>
	</simpleSearch:simpleSearch>
</context>
```

### Advanced Search

```xml
<context type="Product" component="advanced-search">
	<simpleSearch:simpleSearch xmlns:simpleSearch="http://www.hybris.com/cockpitng/config/simpleSearch">
		<simpleSearch:field name="code" operator="contains" selected="true"/>
		<simpleSearch:field name="name" operator="contains" selected="true"/>
		<simpleSearch:field name="approvalStatus" operator="equals"/>
		<simpleSearch:field name="supercategories" operator="equals"/>
		<simpleSearch:field name="creationtime" operator="greater"/>
	</simpleSearch:simpleSearch>
</context>
```

Guidelines:

- Only include fields that are indexed or frequently used; advanced search with many joins can be slow.
- Prefer `contains` for free text fields and `equals` for enumerations/references.

## List View Customization

Configure which columns appear in the list view and how values are rendered.

```xml
<context type="Product" component="listview">
	<listview:listview xmlns:listview="http://www.hybris.com/cockpitng/component/listView">
		<listview:column qualifier="code" width="160" sortable="true"/>
		<listview:column qualifier="name" width="auto" sortable="true" spring-bean="localizedListCellRenderer"/>
		<listview:column qualifier="approvalStatus" width="140" sortable="true"/>
		<listview:column qualifier="catalogVersion" width="220"/>
		<listview:column qualifier="modifiedtime" width="170" sortable="true"/>
	</listview:listview>
</context>
```

Guidelines:

- Keep list columns minimal; too many columns increase rendering time and reduce usability.
- Use a custom renderer only when formatting cannot be achieved by existing renderers.

## Custom Editors and Renderers

Create custom editors/renderers for special UI needs (e.g., badges, rating widgets, composite fields).
Treat these as UI components: keep them small, deterministic, and easy to remove.

### Reference: Custom Editor Renderer (Java)

```java
package com.mycompany.backoffice.editors;

import com.hybris.cockpitng.editors.CockpitEditorRenderer;
import com.hybris.cockpitng.editors.EditorContext;
import com.hybris.cockpitng.editors.EditorListener;
import org.zkoss.zul.Intbox;

public class Rating0To5Editor implements CockpitEditorRenderer<Integer> {

	@Override
	public void render(final org.zkoss.zk.ui.Component parent, final EditorContext<Integer> context,
			final EditorListener<Integer> listener) {
		final Intbox intbox = new Intbox();
		intbox.setConstraint("min 0 max 5: Rating must be between 0 and 5");

		if (context.getInitialValue() != null) {
			intbox.setValue(context.getInitialValue());
		}

		intbox.addEventListener("onChange", event -> listener.onValueChanged(intbox.getValue()));
		intbox.setParent(parent);
	}
}
```

### Register the Editor in Spring

```xml
<bean id="rating0To5Editor" class="com.mycompany.backoffice.editors.Rating0To5Editor" scope="prototype"/>
```

### Use the Editor in Editor Area Config

```xml
<editorArea:attribute qualifier="customRating" editor="rating0To5Editor"/>
```

Guidelines:

- Prefer prototype scope for UI components.
- Keep any service calls out of render loops; fetch data in advance (or via a dedicated data provider).

## Custom Widgets

Use a custom widget only when the existing backoffice components cannot express the UI or flow.
Keep widgets focused: one responsibility, small surface area, easy to remove.

Typical structure:

```
<extension>/resources/
  widgets/
    mycompanywidget/
      definition.xml
      mycompanywidget.zul
```

Widget definition:

```xml
<widget-definition
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:noNamespaceSchemaLocation="http://www.hybris.com/schema/cockpitng/widget-definition.xsd"
	id="com.mycompany.backoffice.widget.sample"
	name="MyCompany Sample Widget">

	<controller class="com.mycompany.backoffice.widgets.SampleWidgetController"/>
	<view src="mycompanywidget.zul"/>

	<setting key="title" type="String" default-value="mycompany.backoffice.widget.sample.title"/>

	<socketEvent name="currentObject" direction="input"/>
	<socketEvent name="objectUpdated" direction="output"/>
</widget-definition>
```

Widget controller:

```java
package com.mycompany.backoffice.widgets;

import com.hybris.cockpitng.annotations.SocketEvent;
import com.hybris.cockpitng.util.DefaultWidgetController;
import org.zkoss.zk.ui.select.annotation.Wire;
import org.zkoss.zul.Label;

public class SampleWidgetController extends DefaultWidgetController {

	@Wire
	private Label title;

	@SocketEvent(socketId = "currentObject")
	public void onCurrentObjectChanged(final Object item) {
		title.setValue(item == null ? "" : String.valueOf(item));
	}
}
```

Guidelines:

- Put business logic in services; keep the widget controller as glue code.
- Keep widgets resilient to null inputs and partially loaded items.

## Actions and Wizards

Use actions for operations on selected items; use wizards for guided flows.

Wizard example (configurable flow):

```xml
<context type="Product" component="wizard">
	<simpleWizard:simpleWizard xmlns:simpleWizard="http://www.hybris.com/cockpitng/config/simpleWizard">
		<simpleWizard:step id="basic" label="mycompany.backoffice.wizard.product.basic">
			<simpleWizard:property qualifier="code" required="true"/>
			<simpleWizard:property qualifier="name" required="true"/>
			<simpleWizard:property qualifier="catalogVersion" required="true"/>
		</simpleWizard:step>
		<simpleWizard:step id="details" label="mycompany.backoffice.wizard.product.details">
			<simpleWizard:property qualifier="description"/>
			<simpleWizard:property qualifier="ean"/>
			<simpleWizard:property qualifier="approvalStatus"/>
		</simpleWizard:step>
	</simpleWizard:simpleWizard>
</context>
```

Guidelines:

- Keep wizard steps short; validate early and show user-friendly messages.
- Avoid embedding complex business logic in backoffice handlers; delegate to service layer.

## Labels and i18n

Backoffice UI uses label keys for tab/section names, wizard step titles, action names, etc.

`resources/backoffice/labels/labels_en.properties`:

```properties
mycompany.backoffice.product.tab.general=General
mycompany.backoffice.product.section.essential=Essential
mycompany.backoffice.product.tab.classification=Classification
mycompany.backoffice.product.section.classification=Classification Details

mycompany.backoffice.wizard.product.basic=Basics
mycompany.backoffice.wizard.product.details=Details
```

Guidelines:

- Never hardcode visible strings in XML when a label key is accepted.
- Use a consistent prefix and keep keys stable; treat them like API.

## Deep Links

Backoffice supports opening items by type and PK.

```
https://<host>:9002/backoffice/#/cxtype/<TypeCode>/<PK>
```

Example:

```
https://localhost:9002/backoffice/#/cxtype/Product/8796093054977
```

If you generate links programmatically, do not expose them publicly; treat PKs as internal identifiers.

## Best Practices

- Keep backoffice configuration isolated in a dedicated extension; avoid editing platform extensions.
- Prefer small overrides instead of copying large contexts.
- Make UI-only logic UI-only; call service layer for business logic.
- Validate inputs and fail with actionable messages (avoid stack traces in UI where possible).
- Keep performance in mind: small list views, indexed search fields, minimal heavy renderers.

## Common Pitfalls

- Configuration not picked up due to extension order or missing resource registration.
- Copy/paste of large platform contexts creates upgrade pain and subtle merge conflicts.
- Slow advanced search due to unindexed attributes or expensive joins.
- Custom renderers doing DB/service calls for every row in a list view.

## Troubleshooting Checklist

- If configuration changes do not appear: confirm your extension is loaded and your config file is on the classpath.
- If the UI is inconsistent: clear browser cache and restart the server (backoffice state is often cached).
- If a renderer/editor fails: check server logs for cockpitng/ZK exceptions; verify the Spring bean id matches the XML.

## Resources

- SAP Help Portal: Backoffice / cockpitng configuration reference
- Platform source examples: `platformbackoffice` and other backoffice-enabled extensions
