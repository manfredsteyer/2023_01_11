# Patterns for Custom Standalone APIs in Angular

- Actually, they are not patterns but idioms, as it is Angular-specific
- We stick with the term pattern anyway, because it's a more common term
- The patterns are primarily for reusable libs.
- The patterns are taken from the implementations of `@angular/common/http`, `@angular/router`, `@ngrx/store`, and `@ngrx/effects`.
- Golden Rule: Whenever possible, use `@Injectable({providedIn: 'root'})`

## Example

- The inferred patterns are presented using a simple logger lib
- It is as simple as possible but as complex as needed to show the patterns

```mermaid
[LoggerService]-->[LoggerConfig]
[LoggerService]-->[LogFormatter]
[LoggerService]-->*[LogAppender]
```

```typescript
export enum LogLevel {
  DEBUG = 0,
  INFO = 1,
  ERROR = 2,
}
```

```typescript
export abstract class LoggerConfig {
  abstract level: LogLevel;
  abstract formatter: Type<LogFormatter>;
  abstract appenders: Type<LogAppender>[];
}

export const defaultConfig: LoggerConfig = {
  level: LogLevel.DEBUG,
  formatter: DefaultLogFormatter,
  appenders: [DefaultLogAppender],
};
```

```typescript
export abstract class LogFormatter {
  abstract format(level: LogLevel, category: string, msg: string): string;
}

export class DefaultLogFormatter implements LogFormatter {
  format(level: LogLevel, category: string, msg: string): string {
    const levelString = LogLevel[level].padEnd(5);
    return `[${levelString}] ${category.toUpperCase()} ${msg}`;
  }
}
```

```typescript
export abstract class LogAppender {
  abstract append(level: LogLevel, category: string, msg: string): void;
}

export class DefaultLogAppender implements LogAppender {
  append(level: LogLevel, category: string, msg: string): void {
    console.log(msg);
  }
}

export const LOG_APPENDERS = new InjectionToken<LogAppender[]>("LOG_APPENDERS");
```

```typescript
@Injectable()
export class LoggerService {
  private appenders = inject(LOG_APPENDERS);
  private formatter = inject(LogFormatter);
  private config = inject(LoggerConfig);

  log(level: LogLevel, category: string, msg: string): void {
    if (level < this.config.level) {
      return;
    }
    const formatted = this.formatter.format(level, category, msg);
    for (const a of this.appenders) {
      a.append(level, category, formatted);
    }
  }

  error(category: string, msg: string): void {
    this.log(LogLevel.ERROR, category, msg);
  }

  info(category: string, msg: string): void {
    this.log(LogLevel.INFO, category, msg);
  }

  debug(category: string, msg: string): void {
    this.log(LogLevel.DEBUG, category, msg);
  }
}
```

Remark: While the "golden rule" tells us to use @Injectable(providedIn: 'root') where possible, the Logger goes with traditional providers instead. This allows to register it once per environment scope (see patterns `provide-Function` and `Service Chain`, below)

## Pattern: Provider Factory

### Intention

- Providing services for a reusable lib
- Configuring a reusable lib
- Exchanging defined implementation details

### Description

Function that returns a `Provider`-Array cross-casted into the `EnvironmentProviders`-Type. This makes sure, the providers can also be used in an environment scope, which is the root scope or the scope introduced with a lazy routing configuration.

Angular and NGRX put this function into a file `provider.ts`.

### Example

```typescript
export function provideLogger(
  config: Partial<LoggerConfig>
): EnvironmentProviders {
  // using default values for missing properties
  const merged = { ...defaultConfig, ...config };

  return makeEnvironmentProviders([
    {
      provide: LoggerConfig,
      useValue: merged,
    },
    {
      provide: LogFormatter,
      useClass: merged.formatter,
    },
    merged.appenders.map((a) => ({
      provide: LOG_APPENDERS,
      useClass: a,
      multi: true,
    })),
  ]);
}
```

Remarks: The use of multi providers allow other libs and/ or the application to add additional `LogAppenders`.

### Findings and Variations

- This usual pattern is used in all examined libraries
- The `Router` and `HttpClient` have another parameter that takes additional features (see Pattern Features).
- Instead of passing in the concreate service implementation, e. g. LogFormatter, NGRX allows to take either a token or the concreate object for reducers.
- The HttpClient takes an array with functional interceptors via a with function (see pattern feature, below). These functions are also registered as services.

## Pattern: Feature

### Intention

- Activating and configuring optional features
- Making these features tree-shakable
- Providing the underlying services in the current environment scope

### Description

The provide function takes an array with feature object. Each feature object has an identifier called `kind` and a providers array. The `kind` property allows to validate the combination of passed features. For instance, there might be features that are mutual exclusive like configuring XSRF token handling and disabling XSRF token handling for the `HttpClient`.

### Example

```typescript
export type LoggerFeatureKind = "COLOR" | "OTHER-FEATURE";

export interface LoggerFeature {
  kind: LoggerFeatureKind;
  providers: Provider[];
}

export function withColor(config?: Partial<ColorConfig>): LoggerFeature {
  const internal = { ...defaultColorConfig, ...config };

  return {
    kind: "COLOR",
    providers: [
      {
        provide: ColorConfig,
        useValue: internal,
      },
      {
        provide: ColorService,
        useClass: DefaultColorService,
      },
    ],
  };
}
```

```typescript
export function provideLogger(
  config: Partial<LoggerConfig>,
  ...features: LoggerFeature[]
): EnvironmentProviders {
  const merged = { ...defaultConfig, ...config };

  // Inspecting passed features
  const colorFeatures =
    features?.filter((f) => f.kind === "COLOR")?.length ?? 0;

  // Validating passed features
  if (colorFeatures > 1) {
    throw new Error("Only one color feature allowed for logger!");
  }

  return makeEnvironmentProviders([
    {
      provide: LoggerConfig,
      useValue: merged,
    },
    {
      provide: LogFormatter,
      useClass: merged.formatter,
    },
    merged.appenders.map((a) => ({
      provide: LOG_APPENDERS,
      useClass: a,
      multi: true,
    })),

    // Providing services for the features
    features?.map((f) => f.providers),
  ]);
}
```

As features are optional, we need to pass `optional: true` to inject. Otherwise, we would get an exception if the feature is not applied. Also, we need to check for null values:

```typescript
export class DefaultLogAppender implements LogAppender {
  colorService = inject(ColorService, { optional: true });

  append(level: LogLevel, category: string, msg: string): void {
    if (this.colorService) {
      msg = this.colorService.apply(level, msg);
    }
    console.log(msg);
  }
}
```

### Findings and Variations

- The router uses it, e. g. for configuring preloading or for activating debug tracing.
- The HttpClient uses it, e. g. for providing interceptors, for configuring JSONP, and for configuring/ disabling the XSRF token handling.
- Some implementations just inject the current Injector and use it to find out which features have been configured. This is an imperative alternative to using `optional: true`.

## Pattern: Configuration Provider

### Intention

- Configuring existing services
- Providing additional services to be used by an existing one
- Extending the behavior of a service from a nested environment scope

### Description

Configuration Providers are provide-Function extending the behavior of an existing service. The may provide additional services. They use an ENVIRONMENT_INITIALIZER to get hold of instances of such additional services and of the service to extend. The former is registered with the latter one.

### Example

Let's assume an extended version of our LoggerService that allows to define an additional `LogAppender` for each category:

```typescript
@Injectable()
export class LoggerService {

    private appenders = inject(LOG_APPENDERS);
    private formatter = inject(LogFormatter);
    private config = inject(LoggerConfig);
  [...]

    log(level: LogLevel, category: string, msg: string): void {
        if (level < this.config.level) {
            return;
        }
        const formatted = this.formatter.format(level, category, msg);

        // Lookup appender for this very category and use
        // it, if there is one.
        const catAppender = this.categories[category];

        if (catAppender) {
            catAppender.append(level, category, formatted);
        }

        for (const a of this.appenders) {
            a.append(level, category, formatted);
        }
    }

    [...]
}
```

```typescript
export function provideCategory(
  category: string,
  appender: Type<LogAppender>
): EnvironmentProviders {
  // TODO: Is this local token a good idea ?!
  const appenderToken = new InjectionToken<LogAppender>("APPENDER_" + category);

  return makeEnvironmentProviders([
    {
      provide: appenderToken,
      useClass: appender,
    },
    {
      provide: ENVIRONMENT_INITIALIZER,
      multi: true,
      useValue: () => {
        const appender = inject(appenderToken);
        const logger = inject(LoggerService);

        logger.categories[category] = appender;
      },
    },
  ]);
}
```

```typescript
export const FLIGHT_BOOKING_ROUTES: Routes = [
  {
    path: '',
    component: FlightBookingComponent,

    // Providers for this route and child routes
    // Using the providers array sets up a new
    // environment injector for this part of the
    // application.
    providers: [
      // Setting up an NGRX feature slice
      provideState(bookingFeature),
      provideEffects([BookingEffects]),

      // Provide LogAppender for logger category
      provideCategory('booking', DefaultLogAppender),
    ],
    children: [
      {
        path: 'flight-search',
        component: FlightSearchComponent,
      },
      [...]
    ],
  },
];
```

### Findings and Variations

- `@ngrx/store` uses this pattern to register a feature slice.
- `@ngrx/effects` uses this pattern, to wire-up effects provided by a feature.

## Pattern: NgModule Bridge

```typescript
@NgModule({
  imports: [],
  exports: [],
  declarations: [],
  providers: [],
})
export class LoggerModule {
  static forRoot(config = defaultConfig): ModuleWithProviders<LoggerModule> {
    return {
      ngModule: LoggerModule,
      providers: [provideLogger(config)],
    };
  }

  static forCategory(
    category: string,
    appender: Type<LogAppender>
  ): ModuleWithProviders<LoggerModule> {
    return {
      ngModule: LoggerModule,
      providers: [provideCategory(category, appender)],
    };
  }
}
```

## Pattern: Service Chain

```typescript
export const FLIGHT_BOOKING_ROUTES: Routes = [
  {
    path: '',
    component: FlightBookingComponent,
    canActivate: [() => inject(AuthService).isAuthenticated()],
    providers: [
      // NGRX
      provideState(bookingFeature),
      provideEffects([BookingEffects]),

      // Providing **another** logger for this part of the app:
      provideLogger(
        {
          level: LogLevel.DEBUG,
          bubbleUp: true,
          appenders: [DefaultLogAppender],
        },
        withColor({
          debug: 42,
          error: 43,
          info: 46,
        })
      ),

    ],
    children: [
      {
        path: 'flight-search',
        component: FlightSearchComponent,
      },
      [...]
    ],
  },
];
```

```typescript
@Injectable()
export class LoggerService {
  private appenders = inject(LOG_APPENDERS);
  private formatter = inject(LogFormatter);
  private config = inject(LoggerConfig);

  private parentLogger = inject(LoggerService, {
    optional: true,
    skipSelf: true,
  });
  [...]

  log(level: LogLevel, category: string, msg: string): void {
    if (level < this.config.level) {
      return;
    }
    const formatted = this.formatter.format(level, category, msg);

    const catAppender = this.categories[category];

    if (catAppender) {
      catAppender.append(level, category, formatted);
    }

    for (const a of this.appenders) {
      a.append(level, category, formatted);
    }

    if (this.parentLogger) {
        this.parentLogger.log(level, category, msg);
    }
  }
  [...]
}
```

## Pattern: Provider Aggregate

```typescript
type InternalEnvironmentProviders = {
  ɵproviders: Provider[];
};

export function combine(
  ep1: EnvironmentProviders,
  ep2: EnvironmentProviders
): EnvironmentProviders {
  const internal1 = ep1 as unknown as InternalEnvironmentProviders;
  const internal2 = ep2 as unknown as InternalEnvironmentProviders;

  return makeEnvironmentProviders([
    ...internal1.ɵproviders,
    ...internal2.ɵproviders,
  ]);
}

export function provideDiagnostics(): EnvironmentProviders {
  return combineAll(provideLogger({}, withColor()), provideTimer());
}
```

```typescript
export function combineAll(
  ...ep: EnvironmentProviders[]
): EnvironmentProviders {
  const internal = ep as unknown as InternalEnvironmentProviders[];

  return makeEnvironmentProviders([
    internal.reduce(
      (acc: Provider[], p: InternalEnvironmentProviders) => [
        ...acc,
        p.ɵproviders,
      ],
      []
    ),
  ]);
}
```

```typescript
bootstrapApplication(AppComponent, {
  providers: [
    [...]
    provideDiagnostics(),
  ]
}
```

## Pattern: Functional Service

```typescript
bootstrapApplication(AppComponent, {
  providers: [
    provideLogger(
      {
        level: LogLevel.DEBUG,
        appenders: [DefaultLogAppender],

        // Functional CSV-Formatter
        formatter: (level, cat, msg) => [level, cat, msg].join(';'),
      },
      withColor({
        debug: 3,
      })
    ),
  ]
});
```

```typescript
export type LogFormatFn = (
  level: LogLevel,
  category: string,
  msg: string
) => string;

export const LOG_FORMATTER = new InjectionToken<LogFormatter | LogFormatFn>(
  "LOG_FORMATTER"
);
```

```typescript
export function provideLogger(config: Partial<LoggerConfig>, ...features: LoggerFeature[]): EnvironmentProviders {

    const merged = { ...defaultConfig, ...config};

    [...]

    return makeEnvironmentProviders([
        LoggerService,
        {
            provide: LoggerConfig,
            useValue: merged
        },

        // Register LogFormatter
        //  - Functional LogFormatter:  useValue
        //  - Class-based LogFormatters: useClass
        (typeof merged.formatter === 'function' ) ? {
            provide: LOG_FORMATTER,
            useValue: merged.formatter
        } : {
            provide: LOG_FORMATTER,
            useClass: merged.formatter
        },

        merged.appenders.map(a => ({
            provide: LOG_APPENDERS,
            useClass: a,
            multi: true
        })),
        [...]
    ]);
}
```

```typescript
@Injectable()
export class LoggerService {
  private appenders = inject(LOG_APPENDERS);
  private formatter = inject(LOG_FORMATTER);
  private config = inject(LoggerConfig);

  [...]

  private format(level: LogLevel, category: string, msg: string): string {
    if (typeof this.formatter === 'function') {
        return this.formatter(level, category, msg);
    }
    else {
        return this.formatter.format(level, category, msg);
    }
  }

  log(level: LogLevel, category: string, msg: string): void {
    if (level < this.config.level) {
      return;
    }

    const formatted = this.format(level, category, msg);

    [...]
  }
  [...]
}
```
