# Claude Marketplace

> [!WARNING]
> This repository is archived. Check out our [skills](https://docs.tuist.dev/en/guides/features/agentic-coding/skills) page in the documentation to learn about how to install and update our officially-maintained skills.

A collection of Claude Code plugins designed for [Tuist](https://tuist.dev) users. These plugins enhance Claude's capabilities with domain-specific knowledge and tooling for various development workflows.

## What are Claude Plugins?

Claude plugins are `CLAUDE.md` files that provide Claude with specialized instructions, context, and guidelines for specific domains or tools. When you include a plugin in your project, Claude gains expertise in that particular area.

## Available Plugins

| Plugin | Description |
|--------|-------------|
| [Xcode](./plugins/xcode) | Best practices and guidelines for Xcode and Apple platform development |

## Installation

### 1. Add the Marketplace

First, add the Tuist marketplace to your Claude Code configuration:

```
/plugin marketplace add tuist/claude-marketplace
```

### 2. Install a Plugin

Install any plugin from the marketplace:

```
/plugin install xcode@tuist-marketplace
```

Or browse available plugins interactively:

```
/plugin
```

### Managing the Marketplace

List configured marketplaces:

```
/plugin marketplace list
```

Update marketplace metadata:

```
/plugin marketplace update tuist-marketplace
```

Remove the marketplace (this will uninstall any plugins you installed from it):

```
/plugin marketplace remove tuist-marketplace
```

## Contributing

We welcome contributions! To add a new plugin:

1. Create a new directory under `plugins/` with your plugin name (kebab-case)
2. Add a `CLAUDE.md` file with the plugin instructions
3. Add a `README.md` explaining what the plugin does and how to use it
4. Update `marketplace.json` to include your plugin entry
5. Submit a pull request

### Validating Changes

Before submitting, validate the marketplace configuration:

```
claude plugin validate .
```

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
