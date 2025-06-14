# Bifrost Plugin System

Bifrost provides a powerful plugin system that allows you to extend and customize the request/response pipeline. Plugins can be used to implement various functionalities like rate limiting, caching, logging, monitoring, and more.

## 1. How Plugins Work

Plugins in Bifrost follow a simple but powerful interface that allows them to intercept and modify requests and responses at different stages of processing:

1. **PreHook**: Executed before a request is sent to a provider

   - Can modify the request
   - Can add custom headers or parameters
   - Can implement rate limiting or validation
   - Executed in the order they are registered

2. **PostHook**: Executed after receiving a response from a provider
   - Can modify the response
   - Can implement caching
   - Can add monitoring or logging
   - Executed in reverse order of PreHooks

> **Note**: PostHooks maintain symmetry with PreHooks. If a plugin returns a response in its PreHook (short-circuiting the provider call), only the PostHook methods of plugins that had their PreHook executed are called, in reverse order. This ensures proper request/response pairing for each plugin.

## 2. Plugin Interface

```golang
type Plugin interface {
    // PreHook is called before a request is processed by a provider.
    // It allows plugins to modify the request before it is sent to the provider.
    // The context parameter can be used to maintain state across plugin calls.
    // Returns the modified request, an optional response (if the plugin wants to short-circuit the provider call), and any error that occurred during processing.
    // If a response is returned, the provider call is skipped and only the PostHook methods of plugins that had their PreHook executed are called in reverse order.
    PreHook(ctx *context.Context, req *BifrostRequest) (*BifrostRequest, *BifrostResponse, error)

    // PostHook is called after a response is received from a provider.
    // It allows plugins to modify the response before it is returned to the caller.
    // Returns the modified response and any error that occurred during processing.
    PostHook(ctx *context.Context, result *BifrostResponse) (*BifrostResponse, error)

    // Cleanup is called on bifrost shutdown.
    // It allows plugins to clean up any resources they have allocated.
    // Returns any error that occurred during cleanup, which will be logged as a warning by the Bifrost instance.
    Cleanup() error
}
```

## 3. Building Custom Plugins

### Basic Plugin Structure

```golang
type CustomPlugin struct {
    // Your plugin fields
}

func (p *CustomPlugin) PreHook(ctx *context.Context, req *BifrostRequest) (*BifrostRequest, *BifrostResponse, error) {
    // Modify request or add custom logic
    // Return nil for response to continue with provider call
    return req, nil, nil
}

func (p *CustomPlugin) PostHook(ctx *context.Context, result *BifrostResponse) (*BifrostResponse, error) {
    // Modify response or add custom logic
    return result, nil
}

func (p *CustomPlugin) Cleanup() error {
    // Clean up any resources
    return nil
}
```

### Example: Rate Limiting Plugin

```golang
type RateLimitPlugin struct {
    limiter *rate.Limiter
}

func NewRateLimitPlugin(rps float64) *RateLimitPlugin {
    return &RateLimitPlugin{
        limiter: rate.NewLimiter(rate.Limit(rps), 1),
    }
}

func (p *RateLimitPlugin) PreHook(ctx *context.Context, req *BifrostRequest) (*BifrostRequest, *BifrostResponse, error) {
    if err := p.limiter.Wait(*ctx); err != nil {
        return nil, nil, err
    }
    return req, nil, nil
}

func (p *RateLimitPlugin) PostHook(ctx *context.Context, result *BifrostResponse) (*BifrostResponse, error) {
    return result, nil
}

func (p *RateLimitPlugin) Cleanup() error {
    // Rate limiter doesn't need cleanup
    return nil
}
```

### Example: Logging Plugin

```golang
type LoggingPlugin struct {
    logger schemas.Logger
}

func NewLoggingPlugin(logger schemas.Logger) *LoggingPlugin {
    return &LoggingPlugin{logger: logger}
}

func (p *LoggingPlugin) PreHook(ctx *context.Context, req *BifrostRequest) (*BifrostRequest, *BifrostResponse, error) {
    p.logger.Info(fmt.Sprintf("Request to %s with model %s", req.Provider, req.Model))
    return req, nil, nil
}

func (p *LoggingPlugin) PostHook(ctx *context.Context, result *BifrostResponse) (*BifrostResponse, error) {
    p.logger.Info(fmt.Sprintf("Response from %s with %d tokens", result.Model, result.Usage.TotalTokens))
    return result, nil
}

func (p *LoggingPlugin) Cleanup() error {
    // Logger doesn't need cleanup
    return nil
}
```

## 4. Using Plugins

### Initializing Bifrost with Plugins

```golang
client, err := bifrost.Init(schemas.BifrostConfig{
    Account: &yourAccount,
    Plugins: []schemas.Plugin{
        NewRateLimitPlugin(10.0),    // 10 requests per second
        NewLoggingPlugin(logger),    // Custom logging
        // Add more plugins as needed
    },
})
```

## 5. Available Plugins

Bifrost comes with several built-in plugins that you can use out of the box. Each plugin has its own documentation in its respective folder:

- **Rate Limiting**: `plugins/rate-limiting/`
- **Caching**: `plugins/caching/`
- **Monitoring**: `plugins/monitoring/`
- **Logging**: `plugins/logging/`

To use these plugins, you can import them from their respective packages:

```golang
import (
    "github.com/maximhq/bifrost/plugins/rate-limiting"
    "github.com/maximhq/bifrost/plugins/caching"
    // ... other plugin imports
)

// Initialize with built-in plugins
client, err := bifrost.Init(schemas.BifrostConfig{
    Account: &yourAccount,
    Plugins: []schemas.Plugin{
        rate_limiting.New(10.0),
        caching.New(cacheConfig),
        // ... other plugins
    },
})
```

## 6. Best Practices

1. **Plugin Order**

   - Consider the order of plugins carefully
   - Rate limiting plugins should typically be first
   - Logging plugins should be last to capture all modifications

2. **Error Handling**

   - Always handle errors in both PreHook and PostHook
   - Return meaningful error messages
   - Consider the impact of errors on the request pipeline

3. **Performance**

   - Keep plugin logic lightweight
   - Avoid blocking operations in hooks
   - Use context for cancellation

4. **State Management**

   - Be careful with shared state between hooks
   - Use context for passing data between hooks
   - Consider thread safety for concurrent requests

5. **Testing**
   - Write unit tests for your plugins
   - Test error scenarios
   - Verify plugin order and execution

## 7. Plugin Development Guidelines

1. **Documentation**

   - Document your plugin's purpose and configuration
   - Provide usage examples
   - Include performance considerations

2. **Configuration**

   - Make plugins configurable
   - Use sensible defaults
   - Validate configuration

3. **Error Handling**

   - Use custom error types
   - Provide detailed error messages
   - Handle edge cases gracefully

4. **Contribution Process**

   - Open an issue first to discuss the plugin's use case and design
   - Create a pull request with a clear explanation of the plugin's purpose
   - Follow the plugin structure requirements below

5. **Plugin Structure**
   Each plugin should be organized as follows:

   ```
   plugins/
   └── your-plugin-name/
       ├── main.go           # Plugin implementation
       ├── plugin_test.go    # Plugin tests
       ├── README.md         # Documentation
       └── go.mod            # Module definition
   ```

   Example `main.go`:

   ```golang
   package your_plugin_name

   import (
       "context"
       "github.com/maximhq/bifrost/core/schemas"
   )

   type YourPlugin struct {
       // Plugin fields
   }

   func New(config YourPluginConfig) *YourPlugin {
       return &YourPlugin{
           // Initialize plugin
       }
   }

   func (p *YourPlugin) PreHook(ctx *context.Context, req *schemas.BifrostRequest) (*schemas.BifrostRequest, *schemas.BifrostResponse, error) {
       // Implementation
       return req, nil, nil
   }

   func (p *YourPlugin) PostHook(ctx *context.Context, result *schemas.BifrostResponse) (*schemas.BifrostResponse, error) {
       // Implementation
       return result, nil
   }

   func (p *YourPlugin) Cleanup() error {
       // Clean up any resources
       return nil
   }
   ```

   Example `README.md`:

   ````markdown
   # Your Plugin Name

   Brief description of what the plugin does.

   ## Installation

   ```bash
   go get github.com/maximhq/bifrost/plugins/your-plugin-name
   ```

   ## Usage

   Explain plugin usage.

   ## Configuration

   Describe configuration options.

   ## Examples

   Show usage examples.
   ````
