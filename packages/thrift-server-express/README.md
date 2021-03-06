# Thrift Server Express

Express middleware for processing Thrift requests.

## Usage

Adding Thrift support to Express is as easy as just including the provided middleware. Because we are just including middleware it is easy for the same server to support APIs beyond Thrift RPC, such as REST.

### Codegen

Requires @creditkarma/thrift-typescript >= v2.0.0

The easiest way to get started is to generate your thrift services using @creditkarma/thrift-typescript.

```sh
npm install --save-dev @creditkarma/thrift-typescript
```

Add a script to your package.json to codegen. The 'target' option is important to make thrift-typescript generate for this library instead of the Apache libraries.

```json
"scripts": {
  "codegen": "thrift-typescript --target thrift-server --sourceDir thrift --outDir codegen
}
```

### Example Service

```c
service Calculator {
  i32 add(1: i32 left, 2: i32 right)
  i32 subtract(1: i32 left, 2: i32 right)
}
```

### Install

```sh
npm install --save express
npm install --save thrift
npm install --save @creditkarma/thrift-server-express
```

### Use the Plugin

To get things working you need to register the middleware as you would any other express middleware. An important thing to note is the use of bodyParse.raw(). This is required, Thrift must operate on the raw binary Buffers.

```typescript
import * as bodyParser from 'body-parser'
import * as express from 'express'
import { ThriftServerExpress } from '../main'

import {
    Calculator,
} from './codegen/calculator'

const PORT = 8080

const app = express()

/**
 * Implementation of our thrift service.
 *
 * Notice the optional final parameter, "context" - this is the Express request object,
 * passed along to our service by the Thrift Express middleware. Thus, you have access to
 * all HTTP request data from within your service implementation.
 */
const serviceHandlers: Calculator.IHandler<express.Request> = {
    add(left: number, right: number, context?: express.Request): number {
        return left + right
    },
    subtract(left: number, right: number, context?: express.Request): number {
        return left - right
    },
}

app.use(
    '/thrift',
    bodyParser.raw(),
    ThriftServerExpress<Calculator.Processor>({
        serviceName: 'calculator-service',
        handler: new Calculator.Processor(serviceHandlers),
    }),
)

app.get('/control', (req: express.Request, res: express.Response) => {
    res.send('PASS')
})

app.listen(PORT, () => {
    console.log(`Express server listening on port: ${PORT}`)
})
```

#### Options

* serviceName (required): The name of your service. Used for logging and tracing.
* handler (required): The service Processor instance to handle service method calls.
* transport (optional): The kind of Thrift transport to use. Only 'buffered' is currently supported.
* protocol (optional): The kind of Thrift protocol to use, either 'binary' or 'compact'.

### Thrift Server Factory

In the event that you will be creating an Express server only to serve Thrift, you can use the `createThriftServer` factory function to create a `Express.Application` and register the `thriftExpress` middleware in one step.

The factory function takes one additional config options `path`, the path on which to serve your Thrift API. The `path` parameter is optional and will default to `/thrift`. All of the plugin options are the same as above and passed in under the key `thriftOptions`.

```typescript
import * as express from 'express'
import { createThriftServer } from '@creditkarma/thrift-server-express'
import { Calculator } from './codegen/calculator'

const PORT = 8080

const serviceHandlers: Calculator.IHandler<express.Request> = {
    add(left: number, right: number, context?: express.Request): number {
        return left + right
    },
    subtract(left: number, right: number, context?: express.Request): number {
        return left - right
    },
}

const app: express.Application = createThriftServer({
    path: '/thrift',
    thriftOptions: {
        serviceName: 'calculator-service',
        handler: new Calculator.Processor(serviceHandlers),
    },
})

app.listen(PORT, () => {
    console.log(`Express server listening on port: ${PORT}`)
})
```

### Observability

Distributed tracing is provided out-of-the-box with [Zipkin](https://github.com/openzipkin/zipkin-js). Distributed tracing allows you to track a request across multiple service calls to see where latency is in your system or to see where a particular request is failing. Also, just to get a complete picture of how many services a request of a particular kind touch.

Zipkin tracing is added to your Express server through middleware.

```typescript
import * as express from 'express'

import {
    createThriftServer,
    ZipkinTracingExpress,
} from '@creditkarma/thrift-server-express'

import { Calculator } from './codegen/calculator'

const PORT = 8080
const SERVICE_NAME = 'calculator-service'

const app: express.Application = createThriftServer<Calculator.Processor>({
    path: '/thrift',
    thriftOptions: {
        serviceName: SERVICE_NAME,
        handler: new Calculator.Processor({
            add(left: number, right: number, context?: express.Request): number {
                return left + right
            },
            subtract(left: number, right: number, context?: express.Request): number {
                return left - right
            },
        })
    }
})

app.use(ZipkinTracingExpress({
    localServiceName: SERVICE_NAME,
    endpoint: 'http://localhost:9411/api/v1/spans',
    sampleRate: 0.1,
}))

app.listen(PORT, () => {
    console.log(`Express server listening on port: ${PORT}`)
})
```

In order for tracing to be useful other services in your system will also need to be setup with Zipkin tracing. Plugins are available for `thrift-server-hapi` and `thrift-client`. The provided plugins in Thrift Server only support HTTP transport at the moment.

#### Options

* localServiceName (required): The name of your service.
* remoteServiceName (optional): The name of the service you are calling.
* debug (optional): In debug mode all requests are sampled.
* port (optional): Port number on which local server operates. This is just added to recorded metadata. Defaults to 0.
* endpoint (optional): URL of your collector (where to send traces).
* sampleRate (optional): Percentage (from 0 to 1) of requests to sample. Defaults to 0.1.
* httpInterval (optional): Sampling data is batched to reduce network load. This is the rate (in milliseconds) at which to empty the sample queue. Defaults to 1000.

If the endpoint is set then the plugin will send sampling data to the given endpoint over HTTP. If the endpoint is not set then sampling data will just be logged to the console.

#### Tracing Non-Thrift Endpoints

Sometimes, as part of completing your service request, you may need to gather data from both Thrift and non-Thrift endpoints. To get a complete picture you need to trace all of these calls. You can add Zipkin to other requests with instrumentation provided by the [OpenZipkin](https://github.com/openzipkin/zipkin-js) project.

When constructing instrumentation provided by another library you need to use the same `Tracer` in order to maintain the correct trace context. You can import this shared `Tracer` through a call to `getTracerForService`. This assumes a `Tracer` has already been created for your service by usage of one of the Thrift Zipkin plugins.

```typescript
import { getTracerForService } from '@creditkarma/thrift-server-core'
import * as wrapRequest from 'zipkin-instrumentation-request'
import * as request from 'request'

const tracer = getTracerForService('calculator-client')
const zipkinRequest = wrapRequest(request, { tracer, remoteServiceName: 'calculator-service' })
zipkinRequest.get(url, (err, resp, body) => {
    // Do something
})
```

## Contributing

For more information about contributing new features and bug fixes, see our [Contribution Guidelines](https://github.com/creditkarma/CONTRIBUTING.md).
External contributors must sign Contributor License Agreement (CLA)

## License

This project is licensed under [Apache License Version 2.0](./LICENSE)
