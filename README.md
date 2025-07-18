<!-- Autogenerated by weave; DO NOT EDIT -->
# MCP Go SDK

***BREAKING CHANGES***

The latest version contains breaking changes:

- Server.AddTools is replaced by Server.AddTool.

- NewServerTool is replaced by AddTool. AddTool takes a Tool rather than a name and description, so you can 
  set any field on the Tool that you want before associating it with a handler.

- Tool options have been removed. If you don't want AddTool to infer a JSON Schema for you, you can construct one
  as a struct literal, or using any other code that suits you.

- AddPrompts, AddResources and AddResourceTemplates are similarly replaced by singular methods which pair the
  feature with a handler. The ServerXXX types have been removed.

[![PkgGoDev](https://pkg.go.dev/badge/github.com/modelcontextprotocol/go-sdk)](https://pkg.go.dev/github.com/modelcontextprotocol/go-sdk)

This repository contains an unreleased implementation of the official Go
software development kit (SDK) for the Model Context Protocol (MCP).

**WARNING**: The SDK should be considered unreleased, and is currently unstable
and subject to breaking changes. Please test it out and file bug reports or API
proposals, but don't use it in real projects. See the issue tracker for known
issues and missing features. We aim to release a stable version of the SDK in
August, 2025.

## Design

The design doc for this SDK is at [design.md](./design/design.md), which was
initially reviewed at
[modelcontextprotocol/discussions/364](https://github.com/orgs/modelcontextprotocol/discussions/364).

Further design discussion should occur in
[issues](https://github.com/modelcontextprotocol/go-sdk/issues) (for concrete
proposals) or
[discussions](https://github.com/modelcontextprotocol/go-sdk/discussions) for
open-ended discussion. See CONTRIBUTING.md for details.

## Package documentation

The SDK consists of two importable packages:

- The
  [`github.com/modelcontextprotocol/go-sdk/mcp`](https://pkg.go.dev/github.com/modelcontextprotocol/go-sdk/mcp)
  package defines the primary APIs for constructing and using MCP clients and
  servers.
- The
  [`github.com/modelcontextprotocol/go-sdk/jsonschema`](https://pkg.go.dev/github.com/modelcontextprotocol/go-sdk/jsonschema)
  package provides an implementation of [JSON
  Schema](https://json-schema.org/), used for MCP tool input and output schema.

## Example

In this example, an MCP client communicates with an MCP server running in a
sidecar process:

```go
package main

import (
	"context"
	"log"
	"os/exec"

	"github.com/modelcontextprotocol/go-sdk/mcp"
)

func main() {
	ctx := context.Background()

	// Create a new client, with no features.
	client := mcp.NewClient("mcp-client", "v1.0.0", nil)

	// Connect to a server over stdin/stdout
	transport := mcp.NewCommandTransport(exec.Command("myserver"))
	session, err := client.Connect(ctx, transport)
	if err != nil {
		log.Fatal(err)
	}
	defer session.Close()

	// Call a tool on the server.
	params := &mcp.CallToolParams{
		Name:      "greet",
		Arguments: map[string]any{"name": "you"},
	}
	res, err := session.CallTool(ctx, params)
	if err != nil {
		log.Fatalf("CallTool failed: %v", err)
	}
	if res.IsError {
		log.Fatal("tool failed")
	}
	for _, c := range res.Content {
		log.Print(c.(*mcp.TextContent).Text)
	}
}
```

Here's an example of the corresponding server component, which communicates
with its client over stdin/stdout:

```go
package main

import (
	"context"
	"log"

	"github.com/modelcontextprotocol/go-sdk/mcp"
)

type HiParams struct {
	Name string `json:"name", mcp:"the name of the person to greet"`
}

func SayHi(ctx context.Context, cc *mcp.ServerSession, params *mcp.CallToolParamsFor[HiParams]) (*mcp.CallToolResultFor[any], error) {
	return &mcp.CallToolResultFor[any]{
		Content: []mcp.Content{&mcp.TextContent{Text: "Hi " + params.Arguments.Name}},
	}, nil
}

func main() {
	// Create a server with a single tool.
	server := mcp.NewServer("greeter", "v1.0.0", nil)

	mcp.AddTool(server, &mcp.Tool{Name: "greet", Description: "say hi"}, SayHi)
	// Run the server over stdin/stdout, until the client disconnects
	if err := server.Run(context.Background(), mcp.NewStdioTransport()); err != nil {
		log.Fatal(err)
	}
}
```

The `examples/` directory contains more example clients and servers.

## Acknowledgements

Several existing Go MCP SDKs inspired the development and design of this
official SDK, notably [mcp-go](https://github.com/mark3labs/mcp-go), authored
by Ed Zynda. We are grateful to Ed as well as the other contributors to mcp-go,
and to authors and contributors of other SDKs such as
[mcp-golang](https://github.com/metoro-io/mcp-golang) and
[go-mcp](https://github.com/ThinkInAIXYZ/go-mcp). Thanks to their work, there
is a thriving ecosystem of Go MCP clients and servers.

## License

This project is licensed under the MIT License - see the [LICENSE](./LICENSE)
file for details.
