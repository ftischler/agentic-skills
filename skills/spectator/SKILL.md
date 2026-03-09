---
name: spectator
description: Comprehensive reference for Spectator (`@ngneat/spectator/jest`), the Angular test-fixture library used in this project. Covers all factory functions (`createComponentFactory`, `createHostFactory`, `createRoutingFactory`, `createServiceFactory`, `createPipeFactory`, `createDirectiveFactory`, `createHttpFactory`), DOM querying (`byTestId`, `byText`, `byRole`, `query`, `queryAll`), event simulation (`click`, `typeInElement`, `triggerEventHandler`, `keyboard.pressEnter`, `dispatchMouseEvent`), custom matchers (`toExist`, `toHaveText`, `toHaveClass`, `toBeVisible`, `toBeDisabled`), `mockProvider`, `selectOption`, route param helpers, and `overrideComponents`. Use this skill whenever writing, modifying, or debugging Angular tests that use Spectator — even if the user doesn't say "spectator" explicitly but mentions any of these APIs, asks how to query elements in tests, simulate events, write component/service/pipe/routing tests, or use data-testid selectors.
---

# Spectator Testing

Spectator (`@ngneat/spectator/jest`) is the standard test-fixture library in this project. It wraps Angular's TestBed with a cleaner API for creating components, querying the DOM, and triggering events.

Always import from `@ngneat/spectator/jest` (not the base `@ngneat/spectator` package) to get Jest-compatible spy implementations.

## Factory Functions

Pick the right factory for your test scenario:

| Factory                  | Use Case                                                               |
| ------------------------ | ---------------------------------------------------------------------- |
| `createComponentFactory` | Standard component tests                                               |
| `createHostFactory`      | Test via a wrapper host template (for `input()`/`output()` components) |
| `createRoutingFactory`   | Components with `Router`, `ActivatedRoute`, route params               |
| `createServiceFactory`   | Injectable services and stores                                         |
| `createPipeFactory`      | Pure/impure pipes                                                      |
| `createDirectiveFactory` | Attribute/structural directive tests                                   |
| `createHttpFactory`      | Service + HttpTestingController combined                               |

### Common Factory Options

All factory functions accept these shared options:

```typescript
{
  imports?: any[],
  providers?: Provider[],
  declarations?: any[],
  mocks?: Type[],                       // auto-mock these services
  detectChanges?: boolean,              // default true — set false to control CD
  disableAnimations?: boolean,
}
```

Component-specific options:

```typescript
{
  componentProviders?: Provider[],      // providers at component level
  componentViewProviders?: Provider[],
  componentMocks?: Type[],              // mock at component injector level
  componentImports?: [any, any][],      // [[OriginalImport, MockImport]]
  overrideComponents?: [Type, {}][],    // override standalone imports
  overrideDirectives?: [Type, {}][],
  overridePipes?: [Type, {}][],
  overrideModules?: [any, any][],
  declareComponent?: boolean,           // false to prevent re-declaration
  shallow?: boolean,                    // shallow rendering
  deferBlockBehavior?: DeferBlockBehavior,
}
```

### createComponentFactory

```typescript
import { createComponentFactory, Spectator } from '@ngneat/spectator/jest';

const createComponent = createComponentFactory({
  component: MyComponent,
  detectChanges: false,
  providers: [...],
});

let spectator: Spectator<MyComponent>;

beforeEach(() => {
  spectator = createComponent();
  // set up mocks before first CD
  spectator.detectChanges();
});
```

Per-test overrides:

```typescript
spectator = createComponent({
  props: { name: 'Test' },
  providers: [{ provide: MyService, useValue: mockService }],
  detectChanges: false,
});
```

### createHostFactory

Use when you need full control over how inputs are passed and outputs are bound:

```typescript
import { createHostFactory, SpectatorHost } from '@ngneat/spectator/jest';

const createComponent = createHostFactory({
  component: UserSurveyComponent,
  providers: [...],
});

const spectator = createComponent(
  `<my-component [vote]="vote" (submitted)="onSubmit()" />`,
  {
    hostProps: {
      vote: mockVote,
      onSubmit: jest.fn(),
    },
  },
);
```

Custom host component:

```typescript
@Component({ selector: 'custom-host', template: '' })
class CustomHostComponent {
  items = [];
}

const createComponent = createHostFactory({
  component: MyComponent,
  host: CustomHostComponent,
});
```

### createRoutingFactory

For components that depend on route params or navigation:

```typescript
import { createRoutingFactory, SpectatorRouting } from '@ngneat/spectator/jest';

const createComponent = createRoutingFactory({
  component: AdminEditComponent,
  providers: [provideRouter([...])],
  detectChanges: false,
  // Initial route state:
  params: { id: '123' },
  queryParams: { filter: 'active' },
  data: { title: 'Edit' },
  fragment: 'section1',
});

const spectator = createComponent({
  props: { action: 'edit' },
});

// Update route state during test:
spectator.setRouteParam('id', '456');
spectator.setRouteQueryParam('filter', 'inactive');
spectator.setRouteData('title', 'View');
spectator.setRouteFragment('section2');
spectator.triggerNavigation();
```

For integration testing with real routing, set `stubsEnabled: false` and provide `routes`.

### createServiceFactory

```typescript
import { createServiceFactory, SpectatorService } from '@ngneat/spectator/jest';

const createService = createServiceFactory({
  service: MyService,
  mocks: [HttpClient, AuthService],
});

let spectator: SpectatorService<MyService>;

beforeEach(() => {
  spectator = createService();
});

it('should call dependency', () => {
  spectator.service.doSomething();
  expect(spectator.inject(AuthService).check).toHaveBeenCalled();
});
```

### createPipeFactory

```typescript
import { createPipeFactory, SpectatorPipe } from '@ngneat/spectator/jest';

const createPipe = createPipeFactory({
  pipe: MyPipe,
  providers: [...],
});

it('should transform value', () => {
  const spectator = createPipe(`{{ 'hello' | myPipe }}`);
  expect(spectator.element).toHaveText('HELLO');
});

it('should transform with host props', () => {
  const spectator = createPipe(`{{ value | myPipe:format }}`, {
    hostProps: { value: 'hello', format: 'upper' },
  });
  expect(spectator.element).toHaveText('HELLO');
});
```

### createDirectiveFactory

```typescript
import { createDirectiveFactory, SpectatorDirective } from '@ngneat/spectator/jest';

const createDirective = createDirectiveFactory({
  directive: HighlightDirective,
});

it('should highlight on hover', () => {
  const spectator = createDirective(`<div appHighlight>Test</div>`);
  spectator.dispatchMouseEvent(spectator.element, 'mouseover');
  expect(spectator.element).toHaveStyle({ backgroundColor: 'yellow' });
});
```

### createHttpFactory

Combines service testing with `HttpTestingController`:

```typescript
import { createHttpFactory, SpectatorHttp, HttpMethod } from '@ngneat/spectator/jest';

const createService = createHttpFactory(MyApiService);

it('should fetch data', () => {
  const spectator = createService();
  spectator.service.getData().subscribe();

  spectator.expectOne('/api/data', HttpMethod.GET);
  // or: const req = spectator.controller.expectOne('/api/data');
  // req.flush({ result: 'ok' });
});
```

## DOM Querying

### byTestId — preferred selector

Always use `data-testid` attributes in templates and `byTestId()` in tests:

```typescript
import { byTestId } from '@ngneat/spectator/jest';

// Template: <button data-testid="submit">Submit</button>
spectator.query(byTestId('submit')); // single element
spectator.queryAll(byTestId('item')); // multiple elements
spectator.click(byTestId('submit')); // click by testid
```

### Query methods

```typescript
spectator.query('css-selector'); // CSS selector -> Element | null
spectator.query(ChildComponent); // component type -> component instance | null
spectator.queryAll('.item'); // all matching -> Element[]
spectator.queryLast('.item'); // last matching -> Element | null
spectator.queryHost('css-selector'); // query host element
spectator.queryHostAll('css-selector'); // query all in host
```

### Query options

```typescript
// Read a specific token from matched elements
spectator.query('.item', { read: ElementRef });
spectator.query(ChildComponent, { read: ChildComponent });

// Query from document root (not scoped to component)
spectator.query('.modal', { root: true });

// Query within a parent
spectator.query('.item', { parentSelector: '.container' });
```

### DOM Selector helpers

```typescript
import {
  byTestId,
  byPlaceholder,
  byValue,
  byTitle,
  byAltText,
  byLabel,
  byText,
  byTextContent,
  byRole,
} from '@ngneat/spectator/jest';

spectator.query(byPlaceholder('Enter name'));
spectator.query(byValue('option-1'));
spectator.query(byTitle('Close'));
spectator.query(byAltText('Logo'));
spectator.query(byLabel('Email'));
spectator.query(byText('Submit')); // exact text match
spectator.query(byText(/submit/i)); // regex
spectator.query(byText('Submit', { selector: 'button' })); // scoped to element type
spectator.query(byTextContent('Full text content')); // matches innerText
spectator.query(byRole('button'));
spectator.query(byRole('checkbox', { checked: true }));
```

## Event Triggering

### Click, blur, focus

```typescript
spectator.click(byTestId('submit'));
spectator.click('button.primary');
spectator.click(nativeElement);

spectator.blur(byTestId('input'));
spectator.focus(byTestId('input'));
```

### Keyboard events

```typescript
spectator.dispatchKeyboardEvent(element, 'keydown', 'Enter');
spectator.dispatchKeyboardEvent(element, 'keyup', { key: 'A', keyCode: 65 });

// Keyboard helpers
spectator.keyboard.pressEnter();
spectator.keyboard.pressEscape();
spectator.keyboard.pressTab();
spectator.keyboard.pressBackspace();
spectator.keyboard.pressKey('a');
spectator.keyboard.pressKey('ctrl.a');
spectator.keyboard.pressKey('ctrl.shift.a');
```

### Mouse events

```typescript
spectator.dispatchMouseEvent(element, 'mouseenter');
spectator.dispatchMouseEvent(element, 'click', x, y);

spectator.mouse.contextmenu('.target');
spectator.mouse.dblclick('.target');
```

### Touch events

```typescript
spectator.dispatchTouchEvent(element, 'touchstart', x, y);
```

### Text input

```typescript
spectator.typeInElement('hello', byTestId('name-input'));
spectator.typeInElement('hello', 'input[formControlName="name"]');
```

### triggerEventHandler — child component outputs

Simulate child component outputs without accessing component internals:

```typescript
// For output() — use component type selector
spectator.triggerEventHandler(ChildComponent, 'dataLoaded', mockData);

// For model() — use string CSS selector (ModelSignal isn't typed as EventEmitter/OutputRef)
spectator.triggerEventHandler('child-selector', 'modelChange', mockData);

// Query from root (useful for overlay/portal components)
spectator.triggerEventHandler(DialogComponent, 'closed', true, { root: true });

spectator.detectChanges();
```

**Caveat**: Spectator's `KeysMatching` only matches `EventEmitter` and `OutputRef`, not `ModelSignal`. When a child uses `model()` (two-way binding), the component type selector gives a TypeScript error. Use the string element selector instead.

### Event creators

For creating raw event objects:

```typescript
import {
  createKeyboardEvent,
  createMouseEvent,
  createTouchEvent,
  createFakeEvent,
} from '@ngneat/spectator/jest';

const event = createKeyboardEvent('keydown', 'Enter', targetElement);
const mouseEvent = createMouseEvent('click');
```

## Select Element Testing

```typescript
import { selectOption } from '@ngneat/spectator/jest';

// Select by value or text
selectOption(spectator.query('select')!, 'Option 1');
selectOption(spectator.query('select')!, ['Option 1', 'Option 2']); // multi-select
selectOption(spectator.query('select')!, optionElement);

// Suppress events
selectOption(spectator.query('select')!, 'Option 1', { emitEvents: false });
```

## Input/Output Management

```typescript
// Set component inputs programmatically (alternative to host template bindings)
spectator.setInput('name', 'Test');
spectator.setInput({ name: 'Test', count: 5 });

// Set host inputs (when using createHostFactory)
spectator.setHostInput('title', 'New Title');

// Subscribe to component output
spectator.output('clicked').subscribe(spy);
```

## Custom Matchers

### DOM existence and structure

```typescript
expect('.item').toExist();
expect('.item').toHaveLength(3);
expect(element).toHaveId('main');
expect(element).toHaveDescendant('.child');
expect(element).toHaveDescendantWithText({ selector: '.child', text: 'Hello' });
expect(element).toBeEmpty();
```

### Attributes and properties

```typescript
expect(element).toHaveAttribute('role', 'button');
expect(element).toHaveAttribute({ role: 'button', tabindex: '0' });
expect(element).toHaveProperty('disabled', true);
expect(element).toContainProperty({ disabled: true });
```

### Classes and styles

```typescript
expect(element).toHaveClass('active');
expect(element).toHaveClass(['active', 'highlighted']);
expect(element).toHaveClass('active', { strict: true }); // no other classes
expect(element).toHaveStyle({ backgroundColor: 'red' });
```

### Text content

```typescript
expect('.title').toHaveText('Hello');
expect('.title').toHaveText(['Hello', 'World']); // array of items
expect('.title').toHaveText((text) => text.includes('He'));
expect('.title').toContainText('ello');
expect('.title').toHaveExactText('Hello');
expect('.title').toHaveExactTrimmedText('Hello');
```

### Form controls

```typescript
expect('input').toHaveValue('test');
expect('select').toHaveValue(['opt1', 'opt2']);
expect('input').toContainValue('tes');
expect('input[type=checkbox]').toBeChecked();
expect('input[type=checkbox]').toBeIndeterminate();
expect('button').toBeDisabled();
expect('option').toBeSelected();
expect('select').toHaveSelectedOptions('Option 1');
expect('select').toHaveSelectedOptions(['Option 1', 'Option 2']);
```

### Visibility and focus

```typescript
expect(element).toBeVisible();
expect(element).toBeHidden();
expect(element).toBeFocused();
expect(element).toBeMatchedBy('.some-selector');
```

### Data attributes

```typescript
expect(element).toHaveData({ data: 'role', val: 'admin' });
```

### Partial object matching

```typescript
expect(myObject).toBePartial({ name: 'Test' });
```

## Dependency Injection

```typescript
// Inject from root injector
const service = spectator.inject(MyService);

// Inject from component's own injector (for component-level providers)
const store = spectator.fixture.debugElement.injector.get(MyStore);

// Run code in injection context
spectator.runInInjectionContext(() => {
  const service = inject(MyService);
});
```

## Mocking with mockProvider

```typescript
import { mockProvider } from '@ngneat/spectator/jest';

const createComponent = createComponentFactory({
  component: MyComponent,
  providers: [
    mockProvider(MyService), // all methods become jest.fn()
    mockProvider(MyService, { getData: () => of([]) }), // with overrides
  ],
  mocks: [OtherService], // shorthand for mockProvider
  componentMocks: [ComponentLevelService], // mock at component injector
});
```

## overrideComponents

Replace standalone child component imports with mocks:

```typescript
import { MockComponents } from 'ng-mocks';

const childComponents = [ChildA, ChildB];

const createComponent = createComponentFactory({
  component: ParentComponent,
  overrideComponents: [
    [
      ParentComponent,
      {
        remove: { imports: childComponents },
        add: { imports: MockComponents(...childComponents) },
      },
    ],
  ],
});
```

Also available: `overrideDirectives`, `overridePipes`, `overrideModules`.

Alternative for imports: `componentImports` replaces an import directly:

```typescript
componentImports: [[RealComponent, MockComponent]],
```

## Defer Block Testing

```typescript
import { DeferBlockBehavior, DeferBlockState } from '@angular/core/testing';

const createComponent = createComponentFactory({
  component: MyComponent,
  deferBlockBehavior: DeferBlockBehavior.Manual,
});

it('should render deferred content', async () => {
  const spectator = createComponent();
  spectator.detectChanges();

  // Via fixture API
  const deferBlocks = await spectator.fixture.getDeferBlocks();
  await deferBlocks[0].render(DeferBlockState.Complete);

  expect(spectator.query('loaded-component')).toBeTruthy();
});
```

## Change Detection

```typescript
spectator.detectChanges(); // trigger CD on component + host
spectator.detectComponentChanges(); // CD only on component (no host)
spectator.flushEffects(); // flush signal effects
await spectator.fixture.whenStable(); // wait for async operations
spectator.tick(500); // advance fakeAsync timer
```

## Global Configuration

Set default providers/imports for all tests in `test-setup.ts`:

```typescript
import { defineGlobalsInjections } from '@ngneat/spectator';

defineGlobalsInjections({
  providers: [provideAnimations()],
  imports: [SharedModule],
});
```
