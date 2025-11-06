# AFv2 Pattern #3: Routing

Intent-based routing to domain-specific agents with confidence threshold validation.

## Pattern Structure

```
Start → Router → [Billing | Technical | General | FAIL] → Synthesize → Direct Reply
```

## Key Features

- 4-path routing with confidence threshold (0.6)
- Domain-specific agents (Billing, Technical, General)
- FAIL path for invalid/unclear input
- Metadata tracking (confidence scores, alternate routes)

## Files

- `03-routing.json` - Complete Flowise workflow (944 lines)

## Quick Start

1. Import `03-routing.json` into Flowise
2. Configure Anthropic API key for all agents
3. Test with support ticket scenarios

## Use Cases

- Customer support routing
- Ticketing systems with domain categorization
- Multi-domain chatbots
- Intent classification workflows

## Documentation

See [Context Foundry Pattern Library](https://github.com/context-foundry/context-foundry/tree/main/extensions/flowise/templates/afv2-patterns) for complete documentation.

🤖 Built with Context Foundry
