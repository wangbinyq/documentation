---
title: Deploy to Production - ElysiaJS
head:
  - - meta
    - property: 'og:title'
      content: Deploy to Production - ElysiaJS

  - - meta
    - name: 'description'
      content: This page

  - - meta
    - property: 'og:description'
      content: Elysia can be configured by passing an object to the constructor. We can configure Elysia behavior by passing an object to the constructor.
---

# Deploy to production
This page is a guide on how to deploy Elysia to production.

## Compile to binary
We recommended running a build command before deploying to production as it could potentially reduce memory usage and file size significantly.

We recommended compile Elysia into a single binary using the command as follows:
```bash
bun build \
	--compile \
	--minify-whitespace \
	--minify-syntax \
	--target bun \
	--outfile server \
	./src/index.ts
```

This will generate a portable binary `server` which we can run to start our server.

Compiling server to binary usually significantly reduce memory usage by 2-3x compared to development environment.

This command is a bit long, so let's break it down:
1. `--compile` - Compile TypeScript to binary
2. `--minify-whitespace` - Remove unnecessary whitespace
3. `--minify-syntax` - Minify JavaScript syntax to reduce file size
4. `--target bun` - Target the `bun` platform, this can optimize the binary for the target platform
5. `--outfile server` - Output the binary as `server`
6. `./src/index.ts` - The entry file of our server (codebase)

To start our server, simly run the binary.
```bash
./server
```

Once binary is compiled, you don't need `Bun` installed on the machine to run the server.

This is great as the deployment server doesn't need to install an extra runtime to run making binary portable.

### Why not --minify
Bun does have `--minify` flag that will minify the binary.

However if we are using [OpenTelemetry](/plugins/opentelemetry), it's going to reduce a function name to a single character.

This make tracing harder than it should as OpenTelemetry rely on a function name.

However, if you're not using OpenTelemetry, you may opt in for `--minify` instead
```bash
bun build \
	--compile \
	--minify \
	--target bun \
	--outfile server \
	./src/index.ts
```

### Permission
Some Linux distro might not be able to run the binary, we suggest enable executable permission to a binary if you're on Linux:
```bash
chmod +x ./server

./server
```

### Unknown random Chinese error
If you're trying to deploy a binary to your server but unable to run with random chinese character error.

It means that the machine you're running on **doesn't support AVX2**.

Unfortunately, Bun require machine that has an `AVX2` hardware support.

There's no workaround as far as we know.

## Compile to JavaScript
If you are unable to compile to binary or you are deploying on a Windows server.

You may bundle your server to a JavaScript file instead.

```bash
bun build \
	--compile \ // [!code --]
	--minify-whitespace \
	--minify-syntax \
	--target bun \
	--outfile ./dist/index.js \
	./src/index.ts
```

This will generate a single portable JavaScript file that you can deploy on your server.
```bash
NODE_ENV=production bun ./dist/index.js
```

## Docker
On Docker, we recommended to always compile to binary to reduce base image overhead.

Here's an example image using Distroless image using binary.
```dockerfile [Dockerfile]
FROM oven/bun AS build

WORKDIR /app

# Cache packages installation
COPY package.json package.json
COPY bun.lock bun.lock

RUN bun install

COPY ./src ./src

ENV NODE_ENV=production

RUN bun build \
	--compile \
	--minify-whitespace \
	--minify-syntax \
	--target bun \
	--outfile server \
	./src/index.ts

FROM gcr.io/distroless/base

WORKDIR /app

COPY --from=build /app/server server

ENV NODE_ENV=production

CMD ["./server"]

EXPOSE 3000
```

### OpenTelemetry
If you are using [OpenTelemetry](/integrations/opentelemetry) to deploys production server.

As OpenTelemetry rely on monkey-patching `node_modules/<library>`. It's required that make instrumentations works properly, we need to specify that libraries to be instrument is an external module to exclude it from being bundled.

For example, if you are using `@opentelemetry/instrumentation-pg` to instrument `pg` library. We need to exclude `pg` from being bundled and make sure that it is importing `node_modules/pg`.

To make this works, we may specified `pg` as an external module with `--external pg`
```bash
bun build --compile --external pg --outfile server src/index.ts
```

This tells bun to not `pg` bundled into the final output file, and will be imported from the `node_modules` directory at runtime. So on a production server, you must also keeps the `node_modules` directory.

It's recommended to specify packages that should be available in a production server as `dependencies` in `package.json` and use `bun install --production` to install only production dependencies.

```json
{
	"dependencies": {
		"pg": "^8.15.6"
	},
	"devDependencies": {
		"@elysiajs/opentelemetry": "^1.2.0",
		"@opentelemetry/instrumentation-pg": "^0.52.0",
		"@types/pg": "^8.11.14",
		"elysia": "^1.2.25"
	}
}
```

Then after running a build command, on a production server
```bash
bun install --production
```

If the node_modules directory still includes development dependencies, you may remove the node_modules directory and reinstall production dependencies again.

### Monorepo
If you are using Elysia with Monorepo, you may need to include dependent `packages`.

If you are using Turborepo, you may place a Dockerfile inside an your apps directory like **apps/server/Dockerfile**. This may apply to other monorepo manager such as Lerna, etc.

Assume that our monorepo are using Turborepo with structure as follows:
- apps
	- server
		- **Dockerfile (place a Dockerfile here)**
- packages
	- config

Then we can build our Dockerfile on monorepo root (not app root):
```bash
docker build -t elysia-mono .
```

With Dockerfile as follows:
```dockerfile [apps/server/Dockerfile]
FROM oven/bun:1 AS build

WORKDIR /app

# Cache packages
COPY package.json package.json
COPY bun.lock bun.lock

COPY /apps/server/package.json ./apps/server/package.json
COPY /packages/config/package.json ./packages/config/package.json

RUN bun install

COPY /apps/server ./apps/server
COPY /packages/config ./packages/config

ENV NODE_ENV=production

RUN bun build \
	--compile \
	--minify-whitespace \
	--minify-syntax \
	--target bun \
	--outfile server \
	./src/index.ts

FROM gcr.io/distroless/base

WORKDIR /app

COPY --from=build /app/server server

ENV NODE_ENV=production

CMD ["./server"]

EXPOSE 3000
```

## Railway
[Railway](https://railway.app) is one of the popular deployment platform.

Railway assigns a **random port** to expose for each deployment, which can be accessed via the `PORT` environment variable.

We need to modify our Elysia server to accept the `PORT` environment variable to comply with Railway port.

Instead of a fixed port, we may use `process.env.PORT` and provide a fallback on development instead.
```ts
new Elysia()
	.listen(3000) // [!code --]
	.listen(process.env.PORT ?? 3000) // [!code ++]
```

This should allows Elysia to intercept port provided by Railway.

::: tip
Elysia assign hostname to `0.0.0.0` automatically, which works with Railway
:::
