# Backend Monitoring: Because console.log Isn't a Monitoring Strategy

## Integrating New Relic with NestJS

**Objective**: This document offers a step-by-step guide on integrating your NestJS applications with New Relic for enhanced monitoring and error reporting.

## Table of Contents

1. Prerequisites
2. Setting Up New Relic Account
3. Installing the New Relic Agent
4. NestJS Custom Logging Service
5. NestJS Exception Filter
6. Docker Integration

## 1. Prerequisites

- A New Relic account.
- A NestJS application.

## 2. Setting Up Your New Relic Account

1. [Sign up for New Relic](https://newrelic.com/signup) if you haven’t.
2. Login and select New Relic APM product.
3. Create a new application instance and note your license key.

## 3. Installing the New Relic Agent

### Using yarn

1. Navigate to your NestJS project directory and run:

   ```bash
   yarn add newrelic
   ```

2. In the project root directory, create a configuration file named `newrelic.js`. Populate this with the default configurations from `node_modules/newrelic/newrelic.js`.

3. Update the `license_key` and `app_name` fields in `newrelic.js`.

4. **Important**: Import the New Relic agent at the very top of your `main.ts` file, before any other imports. This ensures the agent can properly instrument your application:

   ```typescript
   // main.ts - This MUST be the first line
   import 'newrelic';

   import { NestFactory } from '@nestjs/core';
   import { AppModule } from './app.module';
   // ... rest of your imports
   ```

```javascript
"use strict";
/**
 * New Relic agent configuration.
 *
 * See lib/config/default.js in the agent distribution for a more complete
 * description of configuration variables and their potential values.
 */

exports.config = {
  /**
   * This symbol `||` provides a default value if the environment variable is not set.
   * Array of application names.
   */
  app_name: process.env.NEW_RELIC_APP_NAME || "ltss_backend",
  /**
   * Your New Relic license key.
   */
  license_key: process.env.NEW_RELIC_LICENSE_KEY || "XXXXXXXX",

  /**
   * This setting controls distributed tracing.
   * Distributed tracing lets you see the path that a request takes through your
   * distributed system. Enabling distributed tracing changes the behavior of some
   * New Relic features, so carefully consult the transition guide before you enable
   * this feature: https://docs.newrelic.com/docs/transition-guide-distributed-tracing
   * Default is true.
   */
  distributed_tracing: {
    /**
     * Enables/disables distributed tracing.
     *
     * @env NEW_RELIC_DISTRIBUTED_TRACING_ENABLED
     */
    enabled: true,
  },
  logging: {
    /**
     * Level at which to log. 'trace' is most useful to New Relic when diagnosing
     * issues with the agent, 'info' and higher will impose the least overhead on
     * production applications.
     */
    level: "info",
  },

  application_logging: {
    forwarding: {
      enabled: true,
      max_samples_stored: 10000,
    },
  },

  /**
   * When true, all request headers except for those listed in attributes.exclude
   * will be captured for all traces, unless otherwise specified in a destination's
   * attributes include/exclude lists.
   */
  allow_all_headers: true,
  attributes: {
    /**
     * Prefix of attributes to exclude from all destinations. Allows * as wildcard
     * at end.
     *
     * NOTE: If excluding headers, they must be in camelCase form to be filtered.
     *
     * @env NEW_RELIC_ATTRIBUTES_EXCLUDE
     */
    exclude: [
      "request.headers.cookie",
      "request.headers.authorization",
      "request.headers.proxyAuthorization",
      "request.headers.setCookie*",
      "request.headers.x*",
      "response.headers.cookie",
      "response.headers.authorization",
      "response.headers.proxyAuthorization",
      "response.headers.setCookie*",
      "response.headers.x*",
    ],
  },
};
```

## 4. NestJS Custom Logging Service

Assuming you have a custom logging service named `LoggerService`:

```typescript
import { Injectable } from "@nestjs/common";

@Injectable()
export class LoggerService {
  log(message: any) {
    console.log(message); // Replace with your logging mechanism
  }

  error(message: any, trace?: string) {
    console.error(message);
    if (trace) {
      console.error(trace);
    }
  }

  // ... other log levels like warn, debug, etc.
}
```

## 5. NestJS Exception Filter

Below are two examples of exception filters that log exceptions to New Relic. Choose the one that fits your project:

### Option A: Using NestJS's built-in `Logger`

```typescript
import {
  ArgumentsHost,
  Catch,
  ExceptionFilter,
  HttpException,
  HttpStatus,
  Logger,
} from "@nestjs/common";
import { Request, Response } from "express";

const newRelic = require("newrelic");

/**
 * Custom global exception filter that catches every thrown error.
 */
@Catch()
export class AppExceptionFilter implements ExceptionFilter {
  private readonly logger = new Logger(AppExceptionFilter.name);

  catch(exception: unknown, host: ArgumentsHost): Response {
    const context = host.switchToHttp();
    const request = context.getRequest<Request>();
    const response = context.getResponse<Response>();
    const message =
      exception instanceof HttpException
        ? exception.getResponse()
        : "Internal server error";

    this.logger.error(
      message,
      exception instanceof Error ? exception.stack : undefined
    );

    // Log the error to New Relic with request/response context
    newRelic.noticeError(
      exception,
      {
        request,
        response,
      },
      false
    );

    console.log(`❌ URL: ${request.url} ===> `, exception);

    if (exception instanceof HttpException) {
      const responseMsg = exception.getResponse();

      if (responseMsg["message"] && Array.isArray(responseMsg["message"])) {
        responseMsg["message"] = responseMsg["message"][0];
      }

      if (responseMsg["error"]) {
        responseMsg["code"] = responseMsg["error"]
          .split(" ")
          ?.join("_")
          ?.toUpperCase();
        delete responseMsg["error"];
      }
      delete responseMsg["statusCode"];

      // Record a log event in New Relic
      newRelic.recordLogEvent({
        message: exception.getResponse(),
        level: "ERROR",
        error: exception,
        statusCode: exception.getStatus() || HttpStatus.INTERNAL_SERVER_ERROR,
        timestamp: new Date().toISOString(),
        path: request?.url,
      });

      return response.status(exception.getStatus()).json(responseMsg);
    }

    return response
      .status(HttpStatus.INTERNAL_SERVER_ERROR)
      .json({ message: "Internal server error" });
  }
}
```

### Option B: Using a custom `LoggerService`

If you have a custom logging service (see section 4), you can inject it into the exception filter:

```typescript
import {
  ArgumentsHost,
  Catch,
  ExceptionFilter,
  HttpException,
  HttpStatus,
} from "@nestjs/common";
import { Request, Response } from "express";
import { LoggerService } from "./logger.service";

const newRelic = require("newrelic");

/**
 * Custom global exception filter that catches every thrown error.
 */
@Catch()
export class AppExceptionFilter implements ExceptionFilter {
  constructor(private readonly logger: LoggerService) {}

  catch(exception: unknown, host: ArgumentsHost): Response {
    const context = host.switchToHttp();
    const request = context.getRequest<Request>();
    const response = context.getResponse<Response>();
    const message =
      exception instanceof HttpException
        ? exception.getResponse()
        : "Internal server error";

    this.logger.error(
      message,
      exception instanceof Error ? exception.stack : undefined
    );

    // Log the error to New Relic with request/response context
    newRelic.noticeError(
      exception,
      {
        request,
        response,
      },
      false
    );

    console.log(`❌ URL: ${request.url} ===> `, exception);

    if (exception instanceof HttpException) {
      const responseMsg = exception.getResponse();

      if (responseMsg["message"] && Array.isArray(responseMsg["message"])) {
        responseMsg["message"] = responseMsg["message"][0];
      }

      if (responseMsg["error"]) {
        responseMsg["code"] = responseMsg["error"]
          .split(" ")
          ?.join("_")
          ?.toUpperCase();
        delete responseMsg["error"];
      }
      delete responseMsg["statusCode"];

      // Record a log event in New Relic
      newRelic.recordLogEvent({
        message: exception.getResponse(),
        level: "ERROR",
        error: exception,
        statusCode: exception.getStatus() || HttpStatus.INTERNAL_SERVER_ERROR,
        timestamp: new Date().toISOString(),
        path: request?.url,
      });

      return response.status(exception.getStatus()).json(responseMsg);
    }

    return response
      .status(HttpStatus.INTERNAL_SERVER_ERROR)
      .json({ message: "Internal server error" });
  }
}
```

### Registering the Exception Filter

To activate the exception filter, you must register it globally.

**For Option A (built-in Logger) - Register in `main.ts`:**

```typescript
import 'newrelic';

import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { AppExceptionFilter } from './app-exception.filter';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalFilters(new AppExceptionFilter());
  await app.listen(3000);
}
bootstrap();
```

**For Option B (custom LoggerService) - Register in `app.module.ts`:**

```typescript
import { Module } from '@nestjs/common';
import { APP_FILTER } from '@nestjs/core';
import { AppExceptionFilter } from './app-exception.filter';
import { LoggerService } from './logger.service';

@Module({
  providers: [
    LoggerService,
    {
      provide: APP_FILTER,
      useClass: AppExceptionFilter,
    },
  ],
})
export class AppModule {}
```

## 6. Docker Integration

For a Docker integration:

1. **docker-compose.yml**:

   ```yaml
   version: "3"

   services:
   nestjs-app:
     build:
     context: .
     dockerfile: Dockerfile
     environment:
       - NEW_RELIC_LICENSE_KEY=your_license_key_here
       - NEW_RELIC_APP_NAME=Your NestJS App
     ports:
       - "3000:3000"
   ```

   Make sure to adjust the `newrelic.js` to pick up environment variables for `license_key` and `app_name`.

2. **Run your Dockerized application**:

   ```bash
    docker-compose up --build
   ```

   ***

   Note: In your run of the mill docker-compose application , the new relic agent will crawl and capture basic telemetry of each container.
