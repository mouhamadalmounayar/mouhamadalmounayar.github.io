---
layout: "post"
author: "Al Mounayar Mouhamad"
date: 2024-08-04
title: "Prosemirror x Angular : My experience with prosemirror"
tags: "frontend"
---

I've always wanted to build my own writing collaboration tool, similar to GitBook, Notion, or Confluence. To achieve this, I needed to create a powerful text editor.

After some research, I discovered ProseMirror. In this article, I share the knowledge I gained after days of grappling with technical documentation.

# Setup

I am a big fan of angular so I decided to leverage angular components and prosemirror to build my text-editor.
I started by setting up a standard angular project with the Angular CLI and I installed all needed dependencies

```sh
npm i prosemirror-model prosemirror-state prosemirror-view prosemirror-test-builder
```

Then, I created the component that is actually going to hold the editor :

```
ng g c traak-editor --skip-tests
```

### Prosemirror Data flow

Prosemirror leverages three of its essential modules, in order to allow editing to happen.

- The `EditorView` dispatches `transactions` that are usually caused by user interaction with the DOM.
- `transactions` are then used to create a new document `state` by updating the old one.
- The view is then updated with this new `state`. Thus, creating a unidirectional data flow.

### Design with angular

I wanted external components to alter the view and dispatch transactions. This could be useful in many scenarios, but the main purpose is to be able to achieve a menu-like functionality.
Thus, the `TraakEditorComponent` needs to pass a reference to the view to other components.

In angular, communication between components can be implemented in many ways (Services and Subjects, Inputs, Outputs, Ngrx etc). I decided to keep it simple by using angular's `Input` and `Output` decorators.
The `TraakEditorComponent` creates the view, passes a reference to it to a parent component called `WrapperComponent`. The `WrapperComponent` then passes the view to other child components.

![Alt text](../../../assets/components.svg)

Angular's change detection ensures that child component inputs are updated whenever the parent component's view attribute is modified

Hence, I created the `WrapperComponent` :

```shell
ng g c wrapper --skip-tests
```

### Defining a schema for my editor

The beauty of ProseMirror is that it defines its own data structure for the document, giving you full control over what is allowed in your editor. This is done by defining a schema for your document.
I decided that a document in my editor should consist of a title and multiple lines (essentially paragraphs). A schema also defines behaviors for parsing from the DOM and rendering to the DOM. To read more about schemas and how they work, you can check the [prosemirror-docs](https://prosemirror.net/docs/guide/).

```ts
// traakSchema.ts
import { Schema } from "prosemirror-model";
export const schema = new Schema({
  nodes: {
    doc_title: {
      content: "text*",
      toDOM() {
        return ["h1", 0];
      },
      parseDOM: [{ tag: "h1" }],
    },
    text: {},
    line: {
      content: "text*",
      toDOM() {
        return ["p", 0];
      },
      parseDOM: [{ tag: "p" }],
    },
    doc: {
      content: "doc_title (line)*",
    },
  },
  marks: {
    bold: {
      toDOM() {
        return ["strong", 0];
      },
      parseDOM: [{ tag: "strong" }],
    },
    italic: {
      toDOM() {
        return ["i", 0];
      },
      parseDOM: [{ tag: "i" }],
    },
    strikethrough: {
      toDOM() {
        return ["s", 0];
      },
      parseDOM: [{ tag: "s" }],
    },
    code: {
      toDOM() {
        return ["code", 0];
      },
      parseDOM: [{ tag: "code" }],
    },
    link: {
      attrs: {
        href: {},
      },
      inclusive: false,
      toDOM(node) {
        const { href } = node.attrs;
        return ["a", { href }, 0];
      },
      parseDOM: [
        {
          tag: "a[href]",
          getAttrs(dom) {
            return { href: dom.getAttribute("href") };
          },
        },
      ],
    },
  },
});
```

This is just an example. A schema of a full-featured editor should have nodes for lists, code blocks, headings etc.

I also wanted to add a starter document to be rendered when the editor initializes.

```ts
// traakStarter.ts
export const initialDoc = {
  type: "doc",
  content: [
    {
      type: "doc_title",
      content: [{ type: "text", text: "Page Title" }],
    },
    {
      type: "line",
      content: [{ type: "text", text: "Hello from traak" }],
    },
  ],
};
```

# Implementation

Now that I have justified my design choices, let's delve into the implementation.

`traak-editor.component.*` :
To start, I added a DOM element in the component's template to serve as the anchor for the editor.

```typescript
@Component({
  selector: 'lib-traak-editor',
  standalone: true,
  imports: [],
  template: ` <div #editor test-id="editor"></div> `,
  styles: '',
})
```

Next, I added the `Output` and `editor` attributes.

```ts
@ViewChild('editor') editor?: ElementRef;

@Output viewEvent : EventEmitter<EditorView> = new EventEmitter<EditorView>();
```

With that, I implemented the `initializeEditor()` method. This method will create an `EditorView` instance with my custom schema and document starter. It will also emit the view to the parent component.

```ts
initializeEditor(): void {
    const schema = traakSchema;
    if (schema) {
      const state = EditorState.create({
        doc: Node.fromJSON(schema, traakStarter),
      });
      const view = new EditorView(this.editor?.nativeElement, {
        state: state,
        dispatchTransaction: (tr) => {
          const newState = view.state.apply(tr);
          view.updateState(newState);
          this.viewEvent.emit(view);
        },
      });
      this.viewEvent.emit(view);
    }
  }
```

Since this method needs to be called after the view is rendered, I used the `AfterViewInit` interface.

```ts
  ngAfterViewInit() {
    this.initializeEditor();
  }
```

At this point, I had an editor with absolutely zero functionality. Which felt awesome.

`wrapper.component.*` :

The `WrapperComponent` is a parent to the `TraakEditorComponent`.

```html
<!-->wrapper.component.html<-->
<lib-traak-editor (viewEvent)="handleViewEvent($event)"></lib-traak-editor>
```

I then implemented the above handler to initialize a view attribute:

```ts
//wrapper.component.ts
view : EditorView;

handleViewEvent($event){
    this.view = $event;
}
```

The wrapper component now holds a reference to the editor view, allowing it to pass this view to other child components. For example, it can pass the view to a `MenuComponent`, which can handle the implementation of ProseMirror commands on click events.
To see this setup, you can check the source code [here](https://github.com/mouhamadalmounayar/traak).
