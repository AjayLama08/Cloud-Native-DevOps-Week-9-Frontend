# Products app — CI example

A Vue and TypeScript single-page application that displays products from the Product API. This example demonstrates application layers, dependency injection, validated HTTP responses and build-only continuous integration with GitHub Actions.

## Development environment

Use your own private copy of this repository, following the course worksheet's private-copy procedure. You can choose your repository name freely. Work on `main` and check that `origin` points to your private repository:

```bash
git branch --show-current
git remote -v
```

The repository includes a [Dev Container configuration](.devcontainer/devcontainer.json) using the course image `ghcr.io/tpdavison/devops-labs:2026-latest`.

Install Docker, Visual Studio Code and the Dev Containers extension. Start Docker, open your local clone in VS Code, then select **Dev Containers: Reopen in Container** from the Command Palette. Run the commands below in the container terminal, from the repository root. The container setup also configures Git's safe-directory and file-mode settings for Windows hosts.

If working outside the container, use Node.js 22 and npm. Use the container terminal for the shell commands shown here.

## Run locally

Install the dependency versions recorded in the lockfile:

```bash
npm ci
```

For a new clone, create your local configuration file:

```bash
cp .env.example .env.local
```

If `.env.local` already exists, keep your settings. It is ignored by Git and must not be committed. The example contains:

```dotenv
VITE_API_BASE_URL=http://localhost:7071/api/
```

### Start the Product API

Use your private copy of the [Product service example](https://github.com/tpdavison/cis3039-example-cicd-svc) and follow its README to install dependencies and create its local settings file.

In the **backend repository's terminal**, start the service with the frontend's browser origin allowed by CORS:

```bash
npm start -- --cors http://localhost:5173
```

The `--cors` option configures allowed origins on the local Functions host; see the [Azure Functions Core Tools reference](https://learn.microsoft.com/en-us/azure/azure-functions/functions-core-tools-reference#func-start). Keep the backend running. If you already have a local backend running, stop it before restarting with this option.

### Start the frontend

In the **frontend repository's terminal**, run:

```bash
npm run dev
```

Open [http://localhost:5173](http://localhost:5173) in your browser. When using a Dev Container, forward port **5173** in VS Code if necessary, and forward backend port **7071** from its container. Use the same browser origin you allowed in CORS.

The page should show **Products** and the backend's product cards, with prices displayed in pounds. In the browser's developer tools, open **Network** and confirm that `GET http://localhost:7071/api/products` returns HTTP 200 with a JSON array.

The app displays loading, empty-list and error states. If a request fails, correct the API or CORS configuration, then select **Retry**. Stop the frontend and backend with **Ctrl+C** in their respective terminals when finished.

## API configuration

`VITE_API_BASE_URL` is the API's base path. The app appends `products`, so use `http://localhost:7071/api/`, rather than the full `/api/products` endpoint. A missing final slash is normalised automatically. The URL must be an absolute HTTP(S) URL without credentials, a query string or a fragment.

If the setting is absent, the app defaults to `http://localhost:7071/api/`. Restart the development server after editing `.env.local`.

Vite embeds this public setting in the generated JavaScript **at build time**. Changing it for a production build requires rebuilding the app. Never put passwords, connection strings, publishing profiles or other secrets in a `VITE_*` value: browser users can read these values.

## Continuous integration

The [Build workflow](.github/workflows/build.yml) runs on:

- A push to `main`.
- A pull request targeting `main`.
- A manual run using `workflow_dispatch`.

Its single job runs on `ubuntu-latest`, checks out the repository, selects Node.js 22 with npm caching, installs dependencies using `npm ci`, and runs `npm run build`. Repository permissions are limited to `contents: read`.

The build command runs Vue/TypeScript type-checking and creates the production files in `dist`. A type error or build error fails the workflow. This is build validation; it does not verify application behaviour. There is no automated test suite in this example.

For clarity, the workflow explicitly sets:

```yaml
env:
  VITE_API_BASE_URL: http://localhost:7071/api/
```

This URL is embedded in the CI build output. The build does not contact the API, so no running backend is required for CI. The workflow has no deployment steps, uploaded artefacts or Azure credential requirements.

Ensure GitHub Actions is enabled in your private repository. Make a small change, such as changing the heading in `src/views/Products.vue`, check it locally, then commit and push to `main`. Open your repository's **Actions** tab, select **Build**, and inspect the install and build logs. Expect a successful run.

Run the same install and build commands locally with:

```bash
npm ci
npm run build
```

To inspect the production build in a browser, run:

```bash
npm run preview
```

Open the URL printed by Vite. Forward the preview port if using a container, and allow that exact browser origin in backend CORS. The preview serves the existing `dist` output; it does not rebuild after configuration changes. Stop it with **Ctrl+C** when finished.

## Troubleshooting

| Symptom | Check |
| --- | --- |
| The app cannot start and mentions public configuration | Check `.env.local` for a valid API base URL, then restart Vite. |
| Products cannot be loaded | Check that the backend is running, port 7071 is forwarded if needed, and the browser's Network tab shows the expected `/api/products` URL. |
| The browser reports a CORS error | Allow the exact frontend origin, including scheme and port, on the backend. A successful request outside the browser does not establish that CORS is configured. |
| The API responds but the app shows an error | Inspect the response: the app requires an array of valid products with non-empty IDs and names, and no duplicate IDs. Optional prices must be non-negative safe integers in pence; optional descriptions must be strings. |
| Port 5173 is occupied | Stop the process already using it. This project's development server requires port 5173. |
| CI fails | Open the failing step's log and reproduce with Node.js 22, `npm ci` and `npm run build`. Commit matching `package.json` and `package-lock.json` changes together when changing dependencies. |

## npm commands

| Command | Purpose |
| --- | --- |
| `npm run dev` | Start the development server on port 5173 |
| `npm run type-check` | Check Vue and TypeScript source |
| `npm run build` | Type-check and generate production files in `dist` |
| `npm run build-only` | Generate production files without the separate type-check |
| `npm run preview` | Serve the existing production build locally |
| `npm run format` | Format files under `src` with Prettier |

## Source structure

| Directory | Responsibility |
| --- | --- |
| `src/domain` | Product list model |
| `src/app` | List-products use case and Product service interface |
| `src/infra` | HTTP adapter and API response validation |
| `src/config` | Composition root and public API URL validation |
| `src/composables` | Dependency injection, loading/error state and retry behaviour |
| `src/views` | Product page and price display |
| `src/router` | Route configuration |

Generated files in `dist`, installed dependencies in `node_modules`, and local environment files are ignored by Git. The public configuration example and dependency lockfile are committed so a new clone can reproduce the setup. Local development and build-only CI require no Azure resources.
