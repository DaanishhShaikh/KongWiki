# Custom-Plugin.md

## 1. Overview of Kong's In-Built Plugins

Kong Gateway includes a variety of built-in plugins that provide powerful functionality to manage, secure, and observe APIs without the need to write custom code. These plugins cover most common use cases in API management, such as authentication, rate limiting, request transformation, and logging. Here are some examples:

### Common In-Built Plugins

- **Authentication Plugins**: Kong provides several authentication plugins to secure APIs, including:
  - **Key Authentication**: Validates client requests using API keys.
  - **JWT (JSON Web Token)**: Allows clients to authenticate using JWT tokens.
  - **OAuth 2.0**: Supports OAuth 2.0 authorization framework, enabling secure and scalable authentication.

- **Traffic Control Plugins**:
  - **Rate Limiting**: Controls the rate of requests to protect backend services from being overwhelmed.
  - **Request Transformer**: Modifies requests (e.g., adding headers, transforming JSON data) before they reach backend services.
  - **Proxy Caching**: Caches responses to improve latency and reduce load on backend services.

- **Security Plugins**:
  - **IP Restriction**: Restricts access to APIs based on client IP addresses.
  - **Bot Detection**: Detects and blocks bot traffic, protecting APIs from malicious use.

- **Logging and Monitoring Plugins**:
  - **File Log**: Logs request and response data to a local file.
  - **Syslog**: Sends logs to a centralized Syslog server.
  - **Prometheus**: Provides metrics for monitoring and visualization through Prometheus.

These in-built plugins cover a broad range of API management tasks and are highly configurable. However, there are situations where a custom solution is needed to meet specific business requirements.

## 2. Introduction to Custom Plugins in Kong

While Kong's built-in plugins provide substantial functionality, there are cases where you may need to implement custom business logic that isn't covered by these plugins. Kong Gateway allows developers to write custom plugins, providing complete flexibility to extend Kong's capabilities to meet specific requirements. 

Custom plugins are particularly useful when you need:
- **Custom Authentication Schemes**: Implement unique authentication mechanisms not covered by Kong’s standard options.
- **Request and Response Modifications**: Add or modify request and response processing that the in-built plugins don’t support.
- **Integration with Third-Party Services**: Connect with external systems or services in ways that the in-built plugins don't natively support.

Kong supports writing plugins in **Lua** and **Go**. Lua is the native scripting language used by Kong and is highly performant within its NGINX/OpenResty environment, while Go plugins are supported in specific Kong versions and configurations. This guide focuses on Lua, as it is the most optimized and commonly used language for Kong custom plugins.

## 3. Advantages of Using Lua for Custom Plugins

Using Lua for custom plugins is recommended due to its native compatibility with Kong's core architecture. Lua runs within Kong's NGINX environment via OpenResty, which means Lua scripts can execute with minimal overhead. The benefits of Lua for Kong plugins include:

- **High Performance**: Lua’s lightweight nature and efficient memory usage reduce latency, making it ideal for processing high volumes of API traffic.
- **Lower Overhead**: Since Lua runs natively in Kong’s execution environment, it avoids the overhead associated with calling external services or plugins written in other languages.
- **Seamless Integration**: Lua has direct access to Kong’s APIs and can interact with NGINX and OpenResty, providing fine-grained control over HTTP request and response processing.

Due to these advantages, using Lua for custom plugins in Kong is typically the most performant option, especially in latency-sensitive applications.

## 4. Writing a Custom Plugin in Lua

This section provides a step-by-step guide for writing a custom Kong plugin in Lua. We will create a basic plugin that inspects incoming requests and adds custom headers to the response.

### Step 1: Define the Plugin Structure

Start by creating a directory structure for your plugin:

```plaintext
custom-plugins/
└── my-custom-plugin/
    ├── handler.lua
    ├── schema.lua
    └── kong-plugin-my-custom-plugin-0.1.0-1.rockspec
```

- **handler.lua**: Contains the core logic for your plugin.
- **schema.lua**: Defines the configuration schema for your plugin.
- **rockspec**: The LuaRocks specification file for Kong to recognize and install your plugin.

### Step 2: Write the Plugin Handler (handler.lua)

The handler file defines the plugin’s core functionality. Here’s an example of a simple custom plugin that adds a custom header to responses:

```lua
-- handler.lua

local BasePlugin = require "kong.plugins.base_plugin"
local MyCustomPlugin = BasePlugin:extend()

function MyCustomPlugin:new()
  MyCustomPlugin.super.new(self, "my-custom-plugin")
end

function MyCustomPlugin:access(conf)
  MyCustomPlugin.super.access(self)

  -- Add a custom header to the request
  ngx.req.set_header("X-My-Header", conf.custom_header_value)
end

function MyCustomPlugin:header_filter(conf)
  MyCustomPlugin.super.header_filter(self)

  -- Add a custom header to the response
  ngx.header["X-Custom-Response-Header"] = conf.custom_response_header
end

return MyCustomPlugin
```

In this example:
- The `access` phase is used to modify request headers.
- The `header_filter` phase is used to modify response headers.
- We inherit from `BasePlugin` to ensure proper integration with Kong’s plugin lifecycle.

### Step 3: Define the Plugin Schema (schema.lua)

The schema file defines the plugin’s configuration parameters. Here’s an example schema for the above plugin:

```lua
-- schema.lua

local typedefs = require "kong.db.schema.typedefs"

return {
  name = "my-custom-plugin",
  fields = {
    { config = {
        type = "record",
        fields = {
          { custom_header_value = { type = "string", required = true, default = "Hello from Kong!" }},
          { custom_response_header = { type = "string", required = true, default = "Processed by My Custom Plugin" }}
        },
    }, },
  },
}
```

This schema defines two configuration options:
- `custom_header_value`: A custom header added to requests.
- `custom_response_header`: A custom header added to responses.

### Step 4: Define the LuaRocks Specification File (rockspec)

The `.rockspec` file is necessary to package and install the plugin. Here’s an example:

```lua
-- kong-plugin-my-custom-plugin-0.1.0-1.rockspec

package = "kong-plugin-my-custom-plugin"
version = "0.1.0-1"
source = {
  url = "git://your-repo-url"
}

description = {
  summary = "A custom Kong plugin",
  homepage = "https://your-repo-url",
  license = "Apache 2.0"
}

dependencies = {
  "lua >= 5.1",
  "kong >= 2.0.0"
}

build = {
  type = "builtin",
  modules = {
    ["kong.plugins.my-custom-plugin.handler"] = "handler.lua",
    ["kong.plugins.my-custom-plugin.schema"] = "schema.lua",
  }
}
```

### Step 5: Install the Custom Plugin

After defining the plugin, you can install it in Kong as follows:

1. Place the `my-custom-plugin` folder in the `KONG_CUSTOM_PLUGINS` directory (usually `/usr/local/share/lua/5.1/kong/plugins/`).
2. Install the plugin using LuaRocks:

   ```bash
   luarocks install kong-plugin-my-custom-plugin
   ```

3. Enable the plugin in Kong by adding it to the configuration file or using the Admin API.

### Step 6: Activate the Plugin in Kong

To enable the custom plugin, use the Admin API to configure it on a specific service or globally.

```bash
curl -i -X POST http://localhost:8001/services/<service_id>/plugins \
  --data "name=my-custom-plugin" \
  --data "config.custom_header_value=Hello from Custom Plugin!" \
  --data "config.custom_response_header=Processed by Custom Plugin"
```

### Testing the Plugin

Once the plugin is installed and enabled, you can test it by making requests to the API. The plugin will add the custom headers as specified in the configuration.


## Summary

Custom plugins in Kong Gateway allow for extensive flexibility, enabling developers to implement specific business logic, custom authentication schemes, or data transformations. While Kong’s in-built plugins cover a broad range of functionality, custom plugins written in Lua provide a performant way to handle custom requirements with low latency. Lua, being native to Kong’s architecture, is efficient and well-integrated, making it an ideal choice for developing custom plugins.

By following the steps above, you can create and integrate Lua-based plugins tailored to your application’s needs, ensuring that your Kong Gateway setup remains lightweight and optimized.