## Local development

#### 1. Start local server in a separate terminal.

There are two ways to run the local server:

**Option 1: Run from source**

```bash
# Typically run the local server with:
cd server && make run

# Or run webapp and server on separate terminals for better performance
# First terminal: Build and run the webapp
cd webapp && make run
# Second terminal: Run the server
cd server && make run-server
```

**Option 2: Run using Docker (recommended for testing)**

```bash
# 1. Configure environment variables in e2e-tests/.ci/env
#    Create this file if it doesn't exist

# 2. Set the server image (optional)
#    To use the latest master image:
SERVER_IMAGE="mattermostdevelopment/mattermost-enterprise-edition:master"
#    If not set, it will use the current commit: mattermostdevelopment/mattermost-enterprise-edition:$(git rev-parse --short=7 HEAD)
#    Note: The image must exist in Docker Hub at https://hub.docker.com/r/mattermostdevelopment/mattermost-enterprise-edition/tags

# 3. Add your license if needed
MM_LICENSE=<your-license-key>

# 4. For additional configuration options, see e2e-tests/README.md

# 5. Run the server and Playwright's smoke tests from the e2e-tests directory
cd e2e-tests && TEST=playwright make
```

This approach uses the server's Docker image to create a consistent testing environment. It automatically configures the server with the necessary settings for Playwright tests and handles dependencies.

#### 2. Install dependencies and run the test.

```bash
# Install npm packages
npm i

# Install browser binaries as prompted if Playwright is just installed or updated
# See https://playwright.dev/docs/browsers
npx playwright install

# Run a specific test of all projects -- Chrome, Firefox, iPhone and iPad.
# See https://playwright.dev/docs/test-cli.
npm run test -- login

# Run a specific test of a project
npm run test -- login --project=chrome

# Run all tests (including visual tests)
npm run test

# Run CI tests (excludes visual tests, runs only in Chrome)
# Note: visual tests run in a separate workflow
npm run test:ci
```

#### 3. Inspect test results at `/results/output` folder when something fails unexpectedly.

## Run tests in UI mode

Check out https://playwright.dev/docs/test-ui-mode for detailed guide on UI Mode to learn more about its features.

```bash
npm run playwright-ui
```

> **Note:** If no tests appear in the UI, check your filter settings:
>
> - Test name filters
> - Project filters (setup, ipad, chrome, firefox)
> - Tag filters (@tag)
> - Execution status filters
>
> The "setup" project runs the initial configuration tests in `specs/test_setup.ts` (ensuring plugins are loaded and server deployment is correct). These setup tests are typically run only once before other tests and may be unchecked for subsequent runs, though they can remain checked if needed.

## Visual Testing

All visual tests must be placed in the `specs/visual/` directory and tagged with `@visual` in the test tags array. This organization ensures proper test discovery and execution patterns.

Visual tests are used to verify the UI appearance is consistent across browsers and remains stable across code changes. There are two types of visual tests supported:

1. **Built-in snapshot testing**: Uses Playwright's built-in snapshot comparison
2. **Percy integration**: Uses the Percy service for more advanced visual testing and reporting

### CI Pipeline for Visual Tests

In CI environments, visual tests run in a separate dedicated pipeline:

- Regular tests run with `npm run test:ci` which excludes all tests with the `@visual` tag
- Visual tests run in a separate workflow using the Playwright Docker container
- This separation prevents visual tests from slowing down the main test pipeline
- It also ensures visual tests always run in a consistent environment

### Writing Visual Tests

When creating visual tests:

1. **Follow the test documentation format** like other tests:
    - Include JSDoc with `@objective` tag
    - Use action-oriented test title
    - Add proper comment prefixes (`// #` for actions, `// *` for verifications)

2. **Place in the correct location**:
    - Put visual tests in the `specs/visual/` directory, organized by feature area
    - Example: `specs/visual/channels/intro_channel.spec.ts`

3. **Add required tags**:
    - Always include `@visual` tag
    - Add feature-specific tags as needed (e.g., `@login_page`, `@channel_page`)

4. **Manage dynamic content**:
    - Use `pw.hideDynamicChannelsContent()` to hide elements that could change between runs
    - Take snapshots only after UI is fully loaded and stable

Example:

```typescript
/**
 * @objective Capture visual snapshot of the landing/login page
 */
test(
    'displays landing page with login options',
    {tag: ['@visual', '@landing_page']},
    async ({pw, page, browserName, viewport}, testInfo) => {
        // # Go to landing login page
        await pw.landingLoginPage.goto();
        await pw.landingLoginPage.toBeVisible();

        // * Verify landing page appears as expected
        await pw.matchSnapshot(testInfo, {page, browserName, viewport});
    },
);
```

## Updating screenshots is done strictly via Playwright's docker container for consistency

#### 1. Run Playwright's docker container

Change to the `./` project directory, then run the docker container. (See https://playwright.dev/docs/docker for reference.)

```bash
docker run -it --rm -v "$(pwd):/mattermost/" --ipc=host mcr.microsoft.com/playwright:v1.54.0-noble /bin/bash
```

#### 2. Inside the docker container

```bash
export PW_BASE_URL=http://host.docker.internal:8065
export PW_HEADLESS=true
cd mattermost/e2e-tests/playwright

# Install npm packages. Use "npm ci" to match the automated environment
export PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD=1 npm ci

# Run specific test. See https://playwright.dev/docs/test-cli.
npm run test -- login --project=chrome

# Or run all tests
npm run test

# Run visual tests (must be run inside Docker for consistency)
npm run test -- specs/visual

# Update snapshots of visual tests (must be run inside Docker)
npm run test -- specs/visual --update-snapshots

# Run Percy visual tests (requires PERCY_TOKEN environment variable)
export PERCY_TOKEN=<your-percy-token>
npm run percy:docker
```

## Page/Component Object Model

See https://playwright.dev/docs/test-pom.

Page and component abstractions are in shared library located at `./lib/src/ui`. They should be established before writing a spec file so that any future changes in the DOM structure will be made in one place only. No static UI text or fixed locator should be written in the spec file.
