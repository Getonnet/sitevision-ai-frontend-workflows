# Guide: Navigating the Sitevision Editor for AI Agents

## 1. Orientation — the main regions

The editor has three persistent regions:

The **top toolbar** (`id="toolbar"`) runs across the top and holds the global editing actions: the gear/**Edit** menu (`#toolbar-btn-properties`), the **Show** eye menu (`#toolbar-btn-preview`), quick **Add text module** (`#toolbar-btn-addTextPortlet`) and **Add image module** (`#toolbar-btn-addImagePortlet`) buttons, the **Modules** dropdown (`#toolbar-btn-portlet`, `title="Modules"`), and **Done**.

The **left sidebar** contains two stacked panels: a **Pages** tree at the top (the site's page hierarchy) and a **Content** panel below it. The Content panel is the **layers panel** — a Fancytree showing the structure of the currently open page (layouts, grids, rows, columns, and modules nested as parent → child).

The **canvas** in the center is a live preview of the page rendered inside a content iframe (`#content-frame`). Selecting a node in the layers panel highlights the corresponding element on the canvas.

## 2. Working with the layers panel

The Content/layers panel mirrors the page DOM. Each entry is a layer or module; a triangle toggles its children. Clicking an entry selects it (it becomes the active node and is highlighted on the canvas). This is how you target a specific container before editing it.

**Adding a new layer.** Right-click any parent node in the layers panel to open its context menu, then choose **Add layout ▸** and pick the orientation: **Vertical**, **Horizontal**, or **Linked**. A vertical layout stacks its children top-to-bottom; horizontal places them side-by-side. The same menu also offers **Add grid**, **Add grid row**, **Add column**, and **Add in layout** for other structural containers. The new layer is created as a child of the node you right-clicked, so right-click the intended parent.

**Organizing and moving layers.** Drag and drop entries within the layers panel to re-parent or reorder them. Dragging a node onto another node moves it (and all its children) inside that target; dropping between nodes reorders siblings. This is the primary way to restructure a page — move a module into a different column, nest a layout inside another, or change the order of sections.

## 3. Adding modules from the top toolbar

Modules (text, images, and richer components) are added from the toolbar, not the layers panel. For the two most common ones there are dedicated buttons: **Add text module** and **Add image module** insert a text or image block directly.

For everything else, open the **Modules** button (`title="Modules"`, `#toolbar-btn-portlet`). Its dropdown has a **"Find module"** search field, a **"Most recently added modules"** shortcut list, and an **"Available modules"** section grouped into categories — *Image and media, Integration, Interactive, Other, Template related,* and *Social media* — plus a **"Get more modules…"** link. Pick a module to insert it. New modules are placed relative to the currently selected layer, so select the destination container in the layers panel first.

## 4. Responsive (mobile/tablet) preview

The **eye icon** in the top toolbar (the **Show** menu, `#toolbar-btn-preview`) opens viewing modes including **Responsive mode**. Responsive mode renders the current page inside a resizable mobile/tablet frame so you can check how layouts reflow at small breakpoints — without leaving the Sitevision editor or publishing. Use it to confirm things like stacked buttons, hidden elements, or breakpoint-specific styling. Close the responsive frame (the **×** at its top-left) to return to normal editing.

## 5. Adding a background image to a layer

To add a background image to a layer, you need to use the layer's **Properties** dialog: right-click the layer in the layers panel and choose **Properties** (this opens the **Appearance** panel). Expand the **Background** section. There you'll find a **Background color** selector and a **"Use background image"** checkbox. Tick **"Use background image"** to reveal the image controls:

The **Image** field with a **"Select image"** picker (opens the media browser to choose the image), a **Size** dropdown (e.g. *Auto*, cover, contain), an **Alignment** dropdown (e.g. *Top - Left*), and a **Repeat** dropdown (e.g. *Horizontal and vertical*). Set the image and these options, then click **OK** to apply.

## 6. Saving vs. publishing (important for agents)

All edits — new layers, moved children, added modules, background images — are saved to the page's **working copy**, not the live site. The editor's in-memory model can be stale after API-level changes, so **reload the editor** to see them reflected. To make changes live, **publish** separately: right-click the page in the Pages tree and choose **Update**. Treat publishing as a deliberate, separate step rather than an automatic consequence of editing.

## Special Notes

- In the layer settings panel, if a theme is selected, the background option is set to hidden by sitevision.
- when creating styles, if the required css is really small and dont need responsive adjust ments, the styles can be added directly inside: `layer propertis > styles > Custom style` input box.
