# Dify Node Type Awareness Plugin

A LangBot plugin that provides real-time visibility into Dify workflow node execution by monitoring specific node types and sending notifications when they execute.

## Features

- Monitors Dify workflow execution events in real-time
- Configurable node type filtering (e.g., `direct_output`, `llm`)
- Sends notifications through LangBot's built-in messaging system when monitored nodes execute
- Optional workflow-specific monitoring
- Metrics collection for observability

## Installation

1. Copy this plugin directory to your LangBot plugins directory (typically `data/plugins/`)
2. Restart LangBot
3. Configure the plugin via the LangBot plugin management UI

## Configuration

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `api_base_url` | string | Yes | `https://api.dify.ai/v1` | Base URL for Dify API |
| `dify_apikey` | string | Yes | *(empty)* | API key from your Dify instance |
| `workflow_id` | string | No | *(empty)* | Specific workflow ID to monitor (empty = all workflows) |
| `node_types` | array(string) | Yes | `["direct_output"]` | Node types that trigger notifications |
| `enable_metrics` | boolean | No | `true` | Whether to collect metrics |

## Usage

Once configured, the plugin will automatically:
1. Monitor Dify workflow executions triggered by LangBot
2. Send notifications when specified node types execute
3. Continue normal message processing for final responses

## Development

This plugin was created using the LangBot Plugin SDK.

## License

MIT