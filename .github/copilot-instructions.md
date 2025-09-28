# ioBroker Adapter Development with GitHub Copilot

**Version:** 0.4.0
**Template Source:** https://github.com/DrozmotiX/ioBroker-Copilot-Instructions

This file contains instructions and best practices for GitHub Copilot when working on ioBroker adapter development.

## Project Context

You are working on an ioBroker adapter. ioBroker is an integration platform for the Internet of Things, focused on building smart home and industrial IoT solutions. Adapters are plugins that connect ioBroker to external systems, devices, or services.

### Adapter-Specific Context
This adapter (`icons-eclipse-smarthome-classic`) provides visualization icons for ioBroker systems:

- **Purpose**: Provides Eclipse SmartHome Classic icons for ioBroker visualizations (WebUI, vis, etc.)
- **Type**: `visualization-icons` adapter with `onlyWWW: true` and `noConfig: true` 
- **Mode**: `none` - No active adapter instance required
- **Icons Source**: Eclipse SmartHome Classic iconset (originally from OpenHAB)
- **License**: Eclipse Public License (EPL) v2.0
- **Target Usage**: Icons are consumed by visualization adapters like vis, webui, habpanel
- **Content Location**: Static icon files served from `www/` directory
- **No Configuration**: Adapter requires no user configuration (`noConfig: true`)
- **Singleton**: Only one instance can run (`singleton: true`)

### Icon Management Patterns
When working with this visualization icons adapter:
- Icon files are static resources served from the `www/` directory
- No dynamic code execution or runtime logic required
- Focus on file organization, metadata, and proper licensing
- Icons should be organized by category/type for easy discovery
- Maintain consistent naming conventions for icon files
- Ensure proper attribution to Eclipse Foundation in all icon-related files

## Testing

### Unit Testing
- Use Jest as the primary testing framework for ioBroker adapters
- Create tests for all adapter main functions and helper methods
- Test error handling scenarios and edge cases
- Mock external API calls and hardware dependencies
- For adapters connecting to APIs/devices not reachable by internet, provide example data files to allow testing of functionality without live connections
- Example test structure:
  ```javascript
  describe('AdapterName', () => {
    let adapter;
    
    beforeEach(() => {
      // Setup test adapter instance
    });
    
    test('should initialize correctly', () => {
      // Test adapter initialization
    });
  });
  ```

### Integration Testing

**IMPORTANT**: Use the official `@iobroker/testing` framework for all integration tests. This is the ONLY correct way to test ioBroker adapters.

**Official Documentation**: https://github.com/ioBroker/testing

#### Framework Structure
Integration tests MUST follow this exact pattern:

```javascript
const path = require('path');
const { tests } = require('@iobroker/testing');

// Define test coordinates or configuration
const TEST_COORDINATES = '52.520008,13.404954'; // Berlin
const wait = ms => new Promise(resolve => setTimeout(resolve, ms));

// Use tests.integration() with defineAdditionalTests
tests.integration(path.join(__dirname, '..'), {
    defineAdditionalTests({ suite }) {
        suite('Test adapter with specific configuration', (getHarness) => {
            let harness;

            before(() => {
                harness = getHarness();
            });

            it('should configure and start adapter', function () {
                return new Promise(async (resolve, reject) => {
                    try {
                        harness = getHarness();
                        
                        // Get adapter object using promisified pattern
                        const obj = await new Promise((res, rej) => {
                            harness.objects.getObject('system.adapter.your-adapter.0', (err, o) => {
                                if (err) return rej(err);
                                res(o);
                            });
                        });
                        
                        if (!obj) {
                            return reject(new Error('Adapter object not found'));
                        }

                        // Configure adapter properties
                        Object.assign(obj.native, {
                            position: TEST_COORDINATES,
                            createCurrently: true,
                            createHourly: true,
                            createDaily: true,
                            // Add other configuration as needed
                        });

                        // Set the updated configuration
                        harness.objects.setObject(obj._id, obj);

                        console.log('✅ Step 1: Configuration written, starting adapter...');
                        
                        // Start adapter and wait
                        await harness.startAdapterAndWait();
                        
                        console.log('✅ Step 2: Adapter started');

                        // Wait for adapter to process data
                        const waitMs = 15000;
                        await wait(waitMs);

                        console.log('🔍 Step 3: Checking states after adapter run...');
                        
                        // Add test assertions here
                        resolve();
                    } catch (error) {
                        reject(error);
                    }
                });
            });
        });
    }
});
```

### Package Validation Tests
For icon adapters, ensure you have comprehensive package validation:

```javascript
const path = require("path");
const { tests } = require("@iobroker/testing");

// Validate the package files
tests.packageFiles(path.join(__dirname, ".."));
```

This validates:
- `package.json` structure and required fields
- `io-package.json` structure and ioBroker-specific metadata
- Version consistency between files
- License information consistency
- Icon file references and paths

## Configuration and Setup

### ioBroker Adapter Configuration

#### Basic Pattern:
```javascript
// Main adapter class
class MyAdapter extends utils.Adapter {
  constructor(options = {}) {
    super({
      ...options,
      name: 'my-adapter',
    });
    
    this.on('ready', this.onReady.bind(this));
    this.on('unload', this.onUnload.bind(this));
  }
}
```

#### Configuration Access:
```javascript
// Access adapter configuration
const config = this.config;
const apiKey = config.apiKey;
const pollingInterval = config.pollingInterval || 60000;
```

### State Management

#### Creating States:
```javascript
// Create a state with proper definition
await this.setObjectNotExistsAsync('devices.sensor1.temperature', {
  type: 'state',
  common: {
    name: 'Temperature',
    type: 'number',
    role: 'value.temperature',
    unit: '°C',
    read: true,
    write: false,
  },
  native: {},
});
```

#### Setting State Values:
```javascript
// Set state value with proper error handling
try {
  await this.setStateAsync('devices.sensor1.temperature', {
    val: 23.5,
    ack: true,
    ts: Date.now()
  });
} catch (error) {
  this.log.error(`Failed to set temperature: ${error}`);
}
```

### Error Handling and Logging

#### Logging Levels:
```javascript
this.log.error('Critical error occurred');    // Always shown
this.log.warn('Warning message');             // Default and above
this.log.info('Information message');         // Info and above
this.log.debug('Debug information');          // Only in debug mode
this.log.silly('Very detailed information');  // Only in silly mode
```

#### Error Handling Pattern:
```javascript
try {
  const result = await someAsyncOperation();
  this.log.debug(`Operation successful: ${result}`);
} catch (error) {
  this.log.error(`Operation failed: ${error.message}`);
  // Handle error appropriately - don't just log and ignore
  throw error; // or return appropriate error response
}
```

### Timer Management

```javascript
class MyAdapter extends utils.Adapter {
  constructor(options = {}) {
    super(options);
    this.pollingTimer = null;
  }

  onReady() {
    this.startPolling();
  }

  startPolling() {
    if (this.pollingTimer) {
      clearInterval(this.pollingTimer);
    }
    
    this.pollingTimer = setInterval(async () => {
      try {
        await this.fetchData();
      } catch (error) {
        this.log.error(`Polling failed: ${error}`);
      }
    }, this.config.pollingInterval || 60000);
  }

  onUnload(callback) {
    try {
      if (this.pollingTimer) {
        clearInterval(this.pollingTimer);
        this.pollingTimer = null;
      }
      callback();
    } catch (e) {
      callback();
    }
  }
}
```

### Connection Management

```javascript
async onReady() {
  try {
    this.setState('info.connection', false, true);
    
    // Initialize connection
    await this.connect();
    
    this.setState('info.connection', true, true);
    this.log.info('Connected successfully');
    
  } catch (error) {
    this.log.error(`Connection failed: ${error}`);
    this.setState('info.connection', false, true);
  }
}

onUnload(callback) {
  try {
    // Clean up timers
    if (this.pollingTimer) {
      clearInterval(this.pollingTimer);
      this.pollingTimer = undefined;
    }
    
    if (this.reconnectTimer) {
      clearTimeout(this.reconnectTimer);
      this.reconnectTimer = undefined;
    }
    // Close connections, clean up resources
    callback();
  } catch (e) {
    callback();
  }
}
```

## Code Style and Standards

- Follow JavaScript/TypeScript best practices
- Use async/await for asynchronous operations
- Implement proper resource cleanup in `unload()` method
- Use semantic versioning for adapter releases
- Include proper JSDoc comments for public methods

## CI/CD and Testing Integration

### GitHub Actions for API Testing
For adapters with external API dependencies, implement separate CI/CD jobs:

```yaml
# Tests API connectivity with demo credentials (runs separately)
demo-api-tests:
  if: contains(github.event.head_commit.message, '[skip ci]') == false
  
  runs-on: ubuntu-22.04
  
  steps:
    - name: Checkout code
      uses: actions/checkout@v4
      
    - name: Use Node.js 20.x
      uses: actions/setup-node@v4
      with:
        node-version: 20.x
        cache: 'npm'
        
    - name: Install dependencies
      run: npm ci
      
    - name: Run demo API tests
      run: npm run test:integration-demo
```

### CI/CD Best Practices
- Run credential tests separately from main test suite
- Use ubuntu-22.04 for consistency
- Don't make credential tests required for deployment
- Provide clear failure messages for API connectivity issues
- Use appropriate timeouts for external API calls (120+ seconds)

### Package.json Script Integration
Add dedicated script for credential testing:
```json
{
  "scripts": {
    "test:integration-demo": "mocha test/integration-demo --exit"
  }
}
```

### Practical Example: Complete API Testing Implementation
Here's a complete example based on lessons learned from the Discovergy adapter:

#### test/integration-demo.js
```javascript
const path = require("path");
const { tests } = require("@iobroker/testing");

// Helper function to encrypt password using ioBroker's encryption method
async function encryptPassword(harness, password) {
    const systemConfig = await harness.objects.getObjectAsync("system.config");
    
    if (!systemConfig || !systemConfig.native || !systemConfig.native.secret) {
        throw new Error("Could not retrieve system secret for password encryption");
    }
    
    const secret = systemConfig.native.secret;
    let result = '';
    for (let i = 0; i < password.length; ++i) {
        result += String.fromCharCode(secret[i % secret.length].charCodeAt(0) ^ password.charCodeAt(i));
    }
    
    return result;
}

// Run integration tests with demo credentials
tests.integration(path.join(__dirname, ".."), {
    defineAdditionalTests({ suite }) {
        suite("API Testing with Demo Credentials", (getHarness) => {
            let harness;
            
            before(() => {
                harness = getHarness();
            });

            it("Should connect to API and initialize with demo credentials", async () => {
                console.log("Setting up demo credentials...");
                
                if (harness.isAdapterRunning()) {
                    await harness.stopAdapter();
                }
                
                const encryptedPassword = await encryptPassword(harness, "demo_password");
                
                await harness.changeAdapterConfig("your-adapter", {
                    native: {
                        username: "demo@provider.com",
                        password: encryptedPassword,
                        // other config options
                    }
                });

                console.log("Starting adapter with demo credentials...");
                await harness.startAdapter();
                
                // Wait for API calls and initialization
                await new Promise(resolve => setTimeout(resolve, 60000));
                
                const connectionState = await harness.states.getStateAsync("your-adapter.0.info.connection");
                
                if (connectionState && connectionState.val === true) {
                    console.log("✅ SUCCESS: API connection established");
                    return true;
                } else {
                    throw new Error("API Test Failed: Expected API connection to be established with demo credentials. " +
                        "Check logs above for specific API errors (DNS resolution, 401 Unauthorized, network issues, etc.)");
                }
            }).timeout(120000);
        });
    }
});
```

### Visualization Icons Adapter Specific Patterns
Since this is a visualization icons adapter, focus on:

- **File Organization**: Icons should be properly categorized and easily discoverable
- **Metadata Validation**: Ensure `io-package.json` correctly declares the adapter type as `visualization-icons`
- **Static Resource Testing**: Verify icon files are accessible via the web server
- **License Compliance**: All icons must maintain proper Eclipse Public License attribution
- **Documentation**: Maintain clear documentation about icon usage and attribution requirements

#### Example Icon File Structure Validation
```javascript
const fs = require('fs');
const path = require('path');

// Test icon file structure
describe('Icon Files Structure', () => {
    test('should have www directory with icons', () => {
        const wwwDir = path.join(__dirname, '..', 'www');
        expect(fs.existsSync(wwwDir)).toBe(true);
    });
    
    test('should maintain proper file extensions', () => {
        const wwwDir = path.join(__dirname, '..', 'www');
        const files = fs.readdirSync(wwwDir, { recursive: true });
        const iconFiles = files.filter(file => /\.(png|svg|jpg|jpeg)$/i.test(file));
        expect(iconFiles.length).toBeGreaterThan(0);
    });
});
```