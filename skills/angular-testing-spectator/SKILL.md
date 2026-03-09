# Angular Testing Conventions

## Philosophy: Strict Blackbox Testing

Components, Directives, and Pipes are **always tested via the template**. This is a hard rule:

- **NEVER** access component properties/methods directly (e.g. `spectator.component.someMethod()`)
- **NEVER** use type casts (`as any`, `as unknown as X`, `as TestableX`, `Record<string, any>`) to bypass `protected`/`private` access
- **NEVER** call `formGroup.setValue()` / `formGroup.patchValue()` directly -- interact through UI inputs
- **ALWAYS** trigger actions through browser events, clicks, typing, Material test harnesses, and `triggerEventHandler`

### Testing child component interactions

Mock child components with `MockComponents` from ng-mocks and emit outputs via `spectator.triggerEventHandler()`:

```typescript
// WRONG - accessing protected method directly
spectator.component.addSubvote(mockData);

// WRONG - manual stub classes (unnecessary boilerplate)
@Component({ selector: 'child-stub', template: '', standalone: true })
class ChildStubComponent {
  dataLoaded = output<MyType>();
}

// RIGHT - MockComponents + triggerEventHandler
import { MockComponents } from 'ng-mocks';

const createComponent = createComponentFactory({
  component: ParentComponent,
  overrideComponents: [
    [
      ParentComponent,
      {
        remove: { imports: [ChildComponent] },
        add: { imports: MockComponents(ChildComponent) },
      },
    ],
  ],
});

// output() -> use component type selector
spectator.triggerEventHandler(ChildComponent, 'dataLoaded', mockData);

// model() -> use string selector (ModelSignal isn't typed as EventEmitter/OutputRef)
spectator.triggerEventHandler('child-selector', 'modelChange', mockData);

spectator.detectChanges();
```

### Testing outputs

Test component outputs through the host template spy, not by subscribing to the output directly:

```typescript
// WRONG
spectator.component.submitted.subscribe(spy);

// RIGHT - use host template binding
const onSubmittedSpy = jest.fn();
const spectator = createComponent('<my-comp (submitted)="onSubmit()" />', {
  hostProps: { onSubmit: onSubmittedSpy },
});
spectator.click(byTestId('submit'));
expect(onSubmittedSpy).toHaveBeenCalled();
```

## Element Selectors: data-testid + byTestId

Always use `data-testid` attributes in templates for test-targeted elements instead of `id`. In tests, use `byTestId()` from Spectator to query them:

```typescript
// In template:
<button data-testid="submit" mat-stroked-button type="submit">Submit</button>

// In test:
import { byTestId } from '@ngneat/spectator/jest';

spectator.click(byTestId('submit'));
expect(spectator.query(byTestId('create-open'))).toBeTruthy();
```

**Why**: `id` attributes have semantic meaning in HTML and can cause conflicts. `data-testid` is explicitly for testing and won't clash with CSS or ARIA.

## Spectator

Always use `@ngneat/spectator/jest` for all test types:

- **Components**: `createComponentFactory` (set `detectChanges: false`, call `spectator.detectChanges()` after mocking)
- **Host wrappers**: `createHostFactory`
- **Routed components**: `createRoutingFactory`
- **Services**: `createServiceFactory`
- **Pipes**: `createPipeFactory`

```typescript
import { createComponentFactory, Spectator } from '@ngneat/spectator/jest';

const createComponent = createComponentFactory({
  component: MyComponent,
  detectChanges: false,
  // ... providers, imports, mocks
});

let spectator: Spectator<MyComponent>;

beforeEach(() => {
  spectator = createComponent();
  // set up mocks here
  spectator.detectChanges();
});
```

## Mocking Child Components with ng-mocks

Use `MockComponents` from `ng-mocks` to mock standalone child components. This replaces manual stub classes and auto-generates all inputs/outputs:

```typescript
import { MockComponents } from 'ng-mocks';

const childComponents = [ChildA, ChildB, ChildC];

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

For **pipes**, use `mockStandalonePipes` (which wraps `MockPipe` from ng-mocks):

```typescript
import { mockStandalonePipes } from '../../test-helpers/mock-standalone';

overrideComponents: [
  // ... MockComponents override
  mockStandalonePipes(MyComponent, {
    pipes: [{ pipe: PermissionPipe, transform: () => true }],
  }),
],
```

## Clean Console Output

Tests must produce **zero** `console.warn` or `console.error` output. The CI test output must be clean.

- **Mock missing methods on MockComponents**: Use `MockInstance` from ng-mocks to provide methods/signals that the parent's `computed`/`effect` accesses on mocked children:

  ```typescript
  import { MockInstance } from 'ng-mocks';
  MockInstance(VoteChoiceComponent, 'hasErrors', signal(false));
  MockInstance(VoteChoiceComponent, 'scrollToFirstError', jest.fn());
  ```

- **Suppress jsdom-only warnings** (like `cdkFocusInitial is not focusable`) with a targeted `console.warn` spy that only filters the specific message:

  ```typescript
  beforeAll(() => {
    jest.spyOn(console, 'warn').mockImplementation((...args: unknown[]) => {
      if (typeof args[0] === 'string' && args[0].includes('cdkFocusInitial')) return;
      console.info(...args);
    });
  });
  ```

- **Fix root causes** whenever possible (e.g. typos in ngrx state property names) rather than suppressing.

## Snapshots

Use `expect(spectator.fixture).toMatchSnapshot()` to verify rendered output. Use `toMatchInlineSnapshot()` for specific text assertions.

**Never** use `expect(spectator.component).toBeTruthy()` -- it provides zero value. Replace with a snapshot or a meaningful assertion.

## fakeIt Helper

See the dedicated [test-helper skill](../test-helper/SKILL.md) for full documentation.

Use `fakeIt` from `@nemovote/shared/util/test` **inline at call sites** where TypeScript infers the type from context. **Never** pass an explicit generic type parameter.

```typescript
import { fakeIt } from '@nemovote/shared/util/test';

// CORRECT: inline at call site -- type inferred from parameter/return context
myFunction(fakeIt({ id: '1', name: 'Test' }));
mockSubject.next([fakeIt({ id: 'vl-1', name: 'Default' })]);

// CORRECT: in factory functions with return type annotation
const createMock = (overrides: Partial<Vote> = {}): Vote => fakeIt({ id: '1', name: 'Test', ...overrides });

// WRONG: explicit generic -- defeats the purpose of type inference
const mock = fakeIt<Vote>({ id: '1', name: 'Test' }); // Don't do this!
```

## Facade Mocks

Use `provideFacadeMocks` to provide mock facades:

```typescript
const createComponent = createComponentFactory({
  component: MyComponent,
  providers: [provideFacadeMocks([AppFacadeMock, AdminFacadeMock])],
});
```

## Material Test Harnesses

Use Angular Material test harnesses for **all** Material component interactions:

```typescript
import { TestbedHarnessEnvironment } from '@angular/cdk/testing/testbed';
import { MatButtonHarness } from '@angular/material/button/testing';
import { MatSelectHarness } from '@angular/material/select/testing';

it('should submit form', async () => {
  const loader = TestbedHarnessEnvironment.loader(spectator.fixture);
  const submitButton = await loader.getHarness(MatButtonHarness.with({ text: 'Submit' }));
  await submitButton.click();
});
```

## Form Testing

Always interact with forms through the UI -- type into inputs, click buttons, use harnesses. **Never** call `formGroup.setValue()` or `formGroup.patchValue()` directly in tests. This ensures no testing gap between template bindings and component class logic.

## Service Testing

Test with `createServiceFactory`. Access only public methods. Mock all external dependencies:

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
```

## Angular Defer Block Testing

Test deferred content using `DeferBlockBehavior.Manual` and `getDeferBlocks()`:

```typescript
import { DeferBlockBehavior, DeferBlockState } from '@angular/core/testing';

const createComponent = createComponentFactory({
  component: MyComponent,
  deferBlockBehavior: DeferBlockBehavior.Manual,
});

it('should render deferred content', async () => {
  const spectator = createComponent();
  spectator.detectChanges();

  expect(spectator.fixture.nativeElement.innerHTML).toContain('Placeholder');

  const deferBlockFixture = (await spectator.fixture.getDeferBlocks())[0];
  await deferBlockFixture.render(DeferBlockState.Complete);

  expect(spectator.fixture.nativeElement.innerHTML).toContain('loaded content');
});
```

## Transloco (i18n)

Use `provideTranslocoTesting()` for translations in tests:

```typescript
import { provideTranslocoTesting } from '@nemovote/nemovote-client-shared/util/test-helpers';

const createComponent = createComponentFactory({
  component: MyComponent,
  providers: [provideTranslocoTesting()],
});
```

## HTTP Testing

Use the Angular HTTP testing utilities:

```typescript
import { provideHttpClient } from '@angular/common/http';
import { provideHttpClientTesting, HttpTestingController } from '@angular/common/http/testing';

const createService = createServiceFactory({
  service: MyApiService,
  providers: [provideHttpClient(), provideHttpClientTesting()],
});

it('should fetch data', () => {
  const httpController = spectator.inject(HttpTestingController);
  spectator.service.getData().subscribe();

  const req = httpController.expectOne('/api/data');
  req.flush({ result: 'ok' });

  httpController.verify();
});
```
