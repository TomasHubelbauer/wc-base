# Web Components Base Class

I'm playing around with web components in plain JavaScript recently and one thing that I wished I could achieve
without using a bundler is the experience of defining a component and having its stylesheet associated with it
as well as having it be defined in the `customElements` registry automatically, as is provided by UI frameworks
and libraries which provide custom components built on top of the web components technology and achieved using
a bundler/compiler.

The ideal experience is as follows:

```js
// `MyComponent.css` is added to the shadow root
// `my-app-my-component` is defined in `customElements`
class MyComponent extends Component {

}
```

Initially I thought this would do the trick as it worked a charm in my single-file prototype:

```js
/**
 * Provides a base class for all custom HTML elements (web components) which
 * automatically registers the component under a tag name based on the derived
 * class' name and places a stylesheet `link` DOM element referencing a CSS file
 * by the same name as that of the component into the component's shadow root.
 */
export default class Component extends HTMLElement {
  constructor() {
    // Obtain derived class name (caller function name) to be able to evaluate it into the derived class constructor
    // NOTE: this.construct.name cannot be called before super and we need to call customElements.defined before super
    // NOTE: arguments.callee.caller.name cannot be used in strict mode which is implied in ESM
    const derivedClassName = (new Error()).stack.split('\n')[1].match(/^\w+/)[0];

    // Evaluate the derived class name into the derived class constructor needed for the custom element definition
    const derivedClass = eval(derivedClassName);

    // Derive HTML custom element name by splitting the class name words, prefixing with `paper` and joining with dashes
    const customElementTagName = 'paper' + derivedClassName.replace(/[A-Z]/g, match => '-' + match[0].toLowerCase());

    // Define the custom element before calling super so that the `super` call succeeds
    // NOTE: This needs to be called prior to `super` otherwise the `super` call will fail with *Illegal constructor*
    customElements.define(customElementTagName, derivedClass);

    // Call the base HTMLElement class which will now succeed since the derived class constructor has a defined element
    super();

    // Attach shadow in the closed mode to isolate the component's styles
    this._shadowRoot = this.attachShadow({ mode: 'closed' });

    // Import the component's styles by appending a `link` element pointing to a CSS file by the name of the component
    const link = document.createElement('link');
    link.rel = 'stylesheet';
    link.href = derivedClassName + '.css';
    this._shadowRoot.append(link);
  }
}
```

However, it stopped working the moment I attempted to bring it to my application! Why? Because the `eval` is scoped to
the context of the ESM module it is executed in so the derived class is not in the scope and therefore cannot be eval'd
into a constructor by its name. You cannot use static `import` in `eval` so we cannot ad-hoc import it by that name
either like we do with the stylesheet. We could use a dynamic `import`, but the constructor itself cannot be async, so
this is a no-go, too.

I experimented with "importing" the portion of the code which evaluates the derived class name into the derived class
constructor into the scope of the module where the derived class actually resides, so you would instead call something
like `super(hack())` in your component and it would evaluate the name there and provide the constructor to the base
class, but this didn't work either, because while the top of the call site is in the derived class module, the `eval`
itself is still in the base class module.

After mulling over this for a bit, I came up with something which deviates from the initial ideal call-site experience
a little, but it's not too bad:

```js
class MyComponent extends Component {
  constructor() {
    super(MyComponent);
  }
}
```

It is less elegant in that you always have to have the `constructor`, but I do have those in 100 % of my components, so
it is not that bad in practice, albeit a little less cool. You also have to pass in the symbol name for the class, not
just `this`, which is a shame and there is a risk of copy-paste errors, but I guard against those dynamically, so it's
something I guess:

```js
/**
 * Provides a base class for all custom HTML elements (web components) which
 * automatically registers the component under a tag name based on the derived
 * class' name and places a stylesheet `link` DOM element referencing a CSS file
 * by the same name as that of the component into the component's shadow root.
 */
export default class Component extends HTMLElement {
  constructor(/** @type {Component} */ constructor) {
    if (!constructor) {
      throw new Error('Constructor not defined! Pass the derived class name, e.g.: `super(MyComponent)` instead of `super()`!');
    }

    // Derive the component custom HTML element tag name from its constructor class name
    const name = 'paper' + constructor.name.replace(/[A-Z]/g, match => '-' + match[0].toLowerCase());

    // Define the custom element before calling super so that the `super` call succeeds
    if (!customElements.get(name)) {
      customElements.define(name, constructor);
    }

    // Call the base HTMLElement class which will succeed as the component class constructor has a defined element
    super();

    // Check that the provided constructor was correct to prevent against copy-paste errors and element misdefinitions
    if (constructor !== this.constructor) {
      throw new Error(`Incorrect constructor passed! Pass ${this.constructor.name} instead of ${constructor.name}!`);
    }

    // Attach shadow in the closed mode to isolate the component's styles
    this._shadowRoot = this.attachShadow({ mode: 'closed' });

    // Import the component's styles by appending a `link` element pointing to a CSS file by the name of the component
    const link = document.createElement('link');
    link.rel = 'stylesheet';
    link.href = constructor.name + '.css';
    this._shadowRoot.append(link);
  }
}
```

This works as expected and it's good enough for me to use in plain JavaScript to approximate the bundler/custom compiler
experience without having to actually use either and introduce that complexity to a personal project where I'm not
compensated for my time lost troubleshooting tooling and not my own bugs unlike at work, where I am happy to use tooling
such as a bundler or a custom compiler (TypeScript, Svelte). So the trade-off is still good with this design.

This also opens up opportunities for attaching performance hooks to the components automatically, which I'm excited about.
