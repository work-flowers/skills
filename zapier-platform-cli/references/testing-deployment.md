# Testing and Deployment — Zapier Platform CLI

## Writing Tests

Tests use Node.js test frameworks (Mocha is the default). The `zapier-platform-core` package provides `createAppTester` to simulate Zapier's execution environment.

### Basic Test Setup

```js
const zapier = require('zapier-platform-core');
const App = require('../index');
const appTester = zapier.createAppTester(App);

// Load .env file for local testing
zapier.tools.env.inject();

describe('triggers', () => {
  it('should load new contacts', async () => {
    const bundle = {
      inputData: { list_id: 'abc123' },
    };
    const results = await appTester(
      App.triggers.new_contact.operation.perform,
      bundle
    );

    expect(results).toBeInstanceOf(Array);
    expect(results.length).toBeGreaterThan(0);
    expect(results[0]).toHaveProperty('id');
    expect(results[0]).toHaveProperty('name');
  });
});
```

### Testing Authentication

```js
it('should authenticate successfully', async () => {
  const bundle = {
    authData: {
      api_key: process.env.TEST_API_KEY,
    },
  };
  const response = await appTester(
    App.authentication.test,
    bundle
  );

  expect(response).toHaveProperty('id');
});
```

### Testing Creates

```js
it('should create a contact', async () => {
  const bundle = {
    inputData: {
      name: 'Test User',
      email: 'test@example.com',
    },
  };
  const result = await appTester(
    App.creates.create_contact.operation.perform,
    bundle
  );

  expect(result).toHaveProperty('id');
  expect(result.name).toBe('Test User');
});
```

### Testing with Auth Data

Include `authData` in the bundle when your perform function needs credentials:

```js
const bundle = {
  authData: {
    access_token: process.env.ACCESS_TOKEN,
    subdomain: process.env.SUBDOMAIN,
  },
  inputData: {
    name: 'Test',
  },
};
```

### Environment Variables in Tests

Create a `.env` file (don't commit it):

```
CLIENT_ID=your_test_client_id
CLIENT_SECRET=your_test_client_secret
ACCESS_TOKEN=your_test_token
```

Load it in tests:

```js
const zapier = require('zapier-platform-core');
zapier.tools.env.inject();
// Now process.env.CLIENT_ID etc. are available
```

You can also set variables inline: `CLIENT_ID=1234 zapier-platform test`

## Running Tests

```bash
# Run all tests
zapier-platform test

# Same as npm test but adds Zapier-specific environment setup
npm test

# Run specific test file
npx jest test/triggers.test.js

# Run with specific Node version via nvm
nvm exec v22 zapier-platform test
```

## Debugging

### Console Logs

```js
// In your perform function
z.console.log('Input data:', bundle.inputData);
z.console.log('API response:', response.data);
```

View logs after pushing:

```bash
# Console logs (z.console.log output)
zapier-platform logs --type=console

# HTTP logs (all z.request calls)
zapier-platform logs --type=http
zapier-platform logs --type=http --detailed  # Include headers and bodies

# Bundle logs (what bundle looked like for each execution)
zapier-platform logs --type=bundle
```

### Local Debugging

Use standard Node.js debugging:

```bash
# With Node inspector
node --inspect node_modules/.bin/jest test/triggers.test.js
```

## Validation

Check your integration structure is valid before pushing:

```bash
zapier-platform validate
```

This checks your App definition against the Zapier Platform Schema and reports errors.

## Deployment

### First-Time Setup

```bash
# Register your integration with Zapier
zapier-platform register "My Integration"

# Push your first version
zapier-platform push
```

### Pushing Updates

1. Update the `version` in `package.json`
2. Run `zapier-platform push`

Environment variables from the previous version are automatically copied.

### Version Management

```bash
# List all versions
zapier-platform versions

# See integration history
zapier-platform history

# Describe your current integration
zapier-platform describe
```

### Sharing (Private)

```bash
# Invite a specific user to use version 1.0.0
zapier-platform users:add user@example.com 1.0.0

# Add an admin collaborator (can make changes)
zapier-platform team:add user@example.com admin

# Get public sharing links
zapier-platform users:links
```

### Going Public

```bash
# Promote a version to production (first time requires Zapier review)
zapier-platform promote 1.0.1
```

### Migrating Users

```bash
# Move all users from old version to new
zapier-platform migrate 1.0.0 1.0.1 100%

# Or move a percentage
zapier-platform migrate 1.0.0 1.0.1 50%

# Deprecate old version with a sunset date
zapier-platform deprecate 1.0.0 2025-12-01
```

### Deleting Versions

```bash
# Only works if version has no users
zapier-platform delete:version 1.0.0
```

### Pulling Updates from Zapier

If Zapier has made updates to your integration (e.g. bug fixes):

```bash
zapier-platform pull
```

This updates local files to match the latest version on Zapier. Destructive changes prompt for confirmation.

## CI/CD

### Running Tests in CI

Use a CI tool (Travis CI, Circle CI, GitHub Actions) to test on Node.js v22. Example `.github/workflows/test.yml`:

```yaml
name: Test
on: [push]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '22'
      - run: npm install
      - run: npm test
        env:
          CLIENT_ID: ${{ secrets.CLIENT_ID }}
          CLIENT_SECRET: ${{ secrets.CLIENT_SECRET }}
```

### Automated Deployment

```bash
# Use a deploy key instead of interactive login
# Generate at https://developer.zapier.com/partner-settings/deploy-keys/
ZAPIER_DEPLOY_KEY=your_key zapier-platform push
```

## Using npm Modules

Install modules normally:

```bash
npm install --save lodash
```

During `zapier-platform push`, code is copied to a temp folder and `npm install` runs fresh. If a package isn't found in production, try `--disable-dependency-detection` flag.

Since v17.3.1, use `--skip-npm-install` for faster builds if your `node_modules` are already correct.

Do not use compiled/native modules unless they're compatible with AWS Lambda's Node.js v22 runtime.

## Transpilers (Babel, etc.)

Add a `_zapier-build` script to `package.json`:

```json
{
  "scripts": {
    "_zapier-build": "babel src --out-dir lib"
  }
}
```

This runs automatically during `zapier-platform build` and `zapier-platform push`.

## Converting from Platform UI

```bash
# Convert a Platform UI integration to CLI
zapier-platform convert 1234 --version 1.0.1 my-app
```

There is no way to convert CLI back to Platform UI.
