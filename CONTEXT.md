# String Diagram Editor + Compiler — VS Code Extension

## Project Overview

A visual editor for string diagrams that compiles to executable code. String diagrams are the natural syntax for morphisms in monoidal categories — this tool makes them practical for working programmers.

**Core thesis**: Category theory provides precise semantics for composition. String diagrams make that composition visual. Code generation makes it executable. This tool bridges all three.

### Goals

1. **Visual editing**: Draw string diagrams with boxes (morphisms) and wires (types)
2. **Type checking**: Ensure diagrams are well-typed in a monoidal category
3. **Rewriting**: Apply categorical equations (associativity, naturality, etc.)
4. **Code generation**: Compile diagrams to Rust, TypeScript, or other targets
5. **Accessibility**: Usable by programmers who don't know category theory (yet)

### Non-Goals (for v1)

- Full proof assistant capabilities
- Higher-dimensional diagrams (2-categories, etc.)
- Real-time collaboration
- Cloud storage

---

## Categorical Foundations

### What is a String Diagram?

A string diagram is a 2D representation of a morphism in a monoidal category:

```
    A   B                 
    │   │                 
    ├───┤                 
    │ f │    ← box = morphism f: A ⊗ B → C ⊗ D
    ├───┤                 
    │   │                 
    C   D                 
```

- **Wires** = objects (types)
- **Boxes** = morphisms (functions)
- **Vertical composition** = sequential composition (;)
- **Horizontal juxtaposition** = tensor product (⊗)
- **Crossings** = braiding/symmetry

### The Category We're Implementing

For v1, we implement **symmetric monoidal closed categories** (SMCC):

| Structure | Notation | Meaning |
|-----------|----------|---------|
| Objects | A, B, C | Types |
| Morphisms | f: A → B | Functions |
| Tensor | A ⊗ B | Pair/product type |
| Unit | I | Unit type |
| Internal hom | A ⊸ B | Function type (linear) |
| Braiding | σ: A ⊗ B → B ⊗ A | Swap |

The key laws (enforced by rewriting):

```
Associativity:  (A ⊗ B) ⊗ C ≅ A ⊗ (B ⊗ C)
Unit laws:      I ⊗ A ≅ A ≅ A ⊗ I
Symmetry:       A ⊗ B ≅ B ⊗ A
```

### Future Extensions

- **Compact closed**: Duals A* with evaluation/coevaluation (for session types)
- **Cartesian**: Diagonal Δ: A → A ⊗ A and terminal !: A → I (for copying)
- **Traced**: Feedback loops (for recursion)

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         VS Code                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                    Extension Host                         │  │
│  │  ┌─────────────────┐    ┌─────────────────────────────┐  │  │
│  │  │ DiagramEditor   │    │ Language Client             │  │  │
│  │  │ Provider        │    │ (LSP)                       │  │  │
│  │  └────────┬────────┘    └──────────────┬──────────────┘  │  │
│  └───────────┼────────────────────────────┼─────────────────┘  │
│              │                            │                     │
│              │ Webview                    │ JSON-RPC            │
│              │ Messages                   │ (stdio)             │
│              ▼                            ▼                     │
│  ┌──────────────────────┐    ┌─────────────────────────────┐   │
│  │      Webview         │    │     Language Server         │   │
│  │  ┌────────────────┐  │    │        (Rust)               │   │
│  │  │  React App     │  │    │  ┌───────────────────────┐  │   │
│  │  │  ┌──────────┐  │  │    │  │ Type Checker          │  │   │
│  │  │  │ Canvas   │  │  │◄──►│  │ Rewrite Engine        │  │   │
│  │  │  │ (SVG)    │  │  │    │  │ Code Generator        │  │   │
│  │  │  └──────────┘  │  │    │  └───────────────────────┘  │   │
│  │  └────────────────┘  │    │                             │   │
│  └──────────────────────┘    └─────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### Data Flow

1. User draws diagram in webview canvas
2. Webview sends diagram JSON to extension host
3. Extension host forwards to language server via LSP custom method
4. Language server type-checks, returns diagnostics
5. Extension host relays diagnostics to webview
6. Webview renders errors (red wires, etc.)

For compilation:
1. User triggers "Compile to Rust" command
2. Extension host sends compile request to language server
3. Language server generates code, returns as string
4. Extension host opens new editor with generated code

---

## File Structure

```
string-diagrams-vscode/
│
├── CONTEXT.md                    # This file
├── README.md                     # User-facing documentation
├── LICENSE                       # MIT
│
├── extension/                    # VS Code extension (TypeScript)
│   ├── src/
│   │   ├── extension.ts          # Entry point, activation
│   │   ├── diagramEditor.ts      # Custom editor provider
│   │   ├── languageClient.ts     # LSP client setup
│   │   ├── commands.ts           # Command handlers
│   │   └── types.ts              # Shared TypeScript types
│   ├── package.json              # Extension manifest
│   ├── tsconfig.json
│   └── esbuild.config.js         # Bundler config
│
├── editor/                       # Webview React app
│   ├── src/
│   │   ├── index.tsx             # Entry point
│   │   ├── App.tsx               # Main app component
│   │   ├── vscode.ts             # VS Code API bridge
│   │   ├── components/
│   │   │   ├── Canvas.tsx        # Main diagram canvas
│   │   │   ├── Node.tsx          # Box/morphism component
│   │   │   ├── Wire.tsx          # Wire component
│   │   │   ├── Port.tsx          # Input/output port
│   │   │   ├── Palette.tsx       # Node palette sidebar
│   │   │   └── Toolbar.tsx       # Top toolbar
│   │   ├── hooks/
│   │   │   ├── useDrag.ts        # Drag interaction
│   │   │   ├── useWiring.ts      # Wire drawing
│   │   │   └── useSelection.ts   # Selection state
│   │   ├── state/
│   │   │   ├── diagram.ts        # Diagram state management
│   │   │   └── history.ts        # Undo/redo
│   │   └── utils/
│   │       ├── geometry.ts       # Path calculations
│   │       └── layout.ts         # Auto-layout helpers
│   ├── package.json
│   ├── tsconfig.json
│   ├── vite.config.ts
│   └── index.html
│
├── language-server/              # LSP server (Rust)
│   ├── src/
│   │   ├── main.rs               # Entry point, LSP setup
│   │   ├── server.rs             # Request handlers
│   │   ├── diagram.rs            # Diagram data structures
│   │   ├── types.rs              # Categorical type system
│   │   ├── checker.rs            # Type checking algorithm
│   │   ├── rewrite.rs            # Rewrite rules engine
│   │   ├── codegen/
│   │   │   ├── mod.rs
│   │   │   ├── rust.rs           # Rust code generation
│   │   │   └── typescript.rs     # TypeScript code generation
│   │   └── stdlib.rs             # Built-in morphisms
│   ├── Cargo.toml
│   └── tests/
│       ├── checking_tests.rs
│       └── codegen_tests.rs
│
├── shared/                       # Shared definitions
│   └── schema/
│       ├── diagram.schema.json   # JSON schema for .diagram files
│       └── protocol.md           # Custom LSP method documentation
│
├── examples/                     # Example diagrams
│   ├── identity.diagram
│   ├── compose.diagram
│   ├── braiding.diagram
│   └── session-protocol.diagram
│
└── scripts/
    ├── build.sh                  # Build all components
    ├── dev.sh                    # Development mode
    └── package.sh                # Package for distribution
```

---

## Diagram Format (.diagram)

Diagrams are stored as JSON with this schema:

```json
{
  "$schema": "https://stringdiagrams.dev/schema/v1",
  "version": "1.0",
  "metadata": {
    "name": "example",
    "description": "An example diagram",
    "created": "2025-01-15T12:00:00Z"
  },
  "types": {
    "A": { "kind": "base" },
    "B": { "kind": "base" },
    "AB": { "kind": "tensor", "left": "A", "right": "B" }
  },
  "nodes": [
    {
      "id": "node-1",
      "kind": "box",
      "label": "f",
      "position": { "x": 100, "y": 100 },
      "inputs": [
        { "id": "port-1", "type": "A" },
        { "id": "port-2", "type": "B" }
      ],
      "outputs": [
        { "id": "port-3", "type": "C" }
      ]
    },
    {
      "id": "node-2",
      "kind": "box",
      "label": "g",
      "position": { "x": 100, "y": 250 },
      "inputs": [
        { "id": "port-4", "type": "C" }
      ],
      "outputs": [
        { "id": "port-5", "type": "D" },
        { "id": "port-6", "type": "E" }
      ]
    }
  ],
  "wires": [
    {
      "id": "wire-1",
      "source": { "nodeId": "input", "portIndex": 0 },
      "target": { "nodeId": "node-1", "portId": "port-1" }
    },
    {
      "id": "wire-2", 
      "source": { "nodeId": "input", "portIndex": 1 },
      "target": { "nodeId": "node-1", "portId": "port-2" }
    },
    {
      "id": "wire-3",
      "source": { "nodeId": "node-1", "portId": "port-3" },
      "target": { "nodeId": "node-2", "portId": "port-4" }
    },
    {
      "id": "wire-4",
      "source": { "nodeId": "node-2", "portId": "port-5" },
      "target": { "nodeId": "output", "portIndex": 0 }
    },
    {
      "id": "wire-5",
      "source": { "nodeId": "node-2", "portId": "port-6" },
      "target": { "nodeId": "output", "portIndex": 1 }
    }
  ],
  "boundary": {
    "inputs": ["A", "B"],
    "outputs": ["D", "E"]
  }
}
```

### Special Nodes

- `"nodeId": "input"` — the diagram's input boundary
- `"nodeId": "output"` — the diagram's output boundary
- Built-in structural nodes: `identity`, `braiding`, `associator`, `unitor`

---

## Custom LSP Methods

Beyond standard LSP (diagnostics, hover, etc.), we define custom methods:

### `diagram/typeCheck`

Request type checking for a diagram.

```typescript
// Request
interface TypeCheckParams {
  uri: string;
  diagram: DiagramJSON;
}

// Response
interface TypeCheckResult {
  valid: boolean;
  diagnostics: Diagnostic[];
  inferredSignature?: {
    inputs: Type[];
    outputs: Type[];
  };
}
```

### `diagram/compile`

Compile diagram to target language.

```typescript
// Request
interface CompileParams {
  uri: string;
  diagram: DiagramJSON;
  target: "rust" | "typescript" | "haskell";
  options?: {
    functionName?: string;
    modulePrefix?: string;
  };
}

// Response
interface CompileResult {
  success: boolean;
  code?: string;
  errors?: string[];
}
```

### `diagram/rewrite`

Apply a rewrite rule at a location.

```typescript
// Request
interface RewriteParams {
  uri: string;
  diagram: DiagramJSON;
  rule: string;  // e.g., "associativity-left", "braiding"
  location: {
    nodeIds: string[];  // nodes involved in rewrite
  };
}

// Response
interface RewriteResult {
  success: boolean;
  newDiagram?: DiagramJSON;
  error?: string;
}
```

### `diagram/listRewrites`

Get applicable rewrites at current selection.

```typescript
// Request
interface ListRewritesParams {
  uri: string;
  diagram: DiagramJSON;
  selection: string[];  // selected node IDs
}

// Response
interface ListRewritesResult {
  rewrites: {
    id: string;
    name: string;
    description: string;
  }[];
}
```

---

## Type System

### Type Grammar

```
Type ::= 
  | Ident                    -- Base type: A, B, Int, String
  | Type "⊗" Type            -- Tensor product
  | Type "⊸" Type            -- Linear function
  | "I"                      -- Unit
  | "(" Type ")"             -- Grouping
```

### Rust Representation

```rust
#[derive(Clone, Debug, PartialEq, Eq, Hash)]
pub enum Type {
    Base(String),
    Tensor(Box<Type>, Box<Type>),
    Hom(Box<Type>, Box<Type>),
    Unit,
}

impl Type {
    pub fn tensor(a: Type, b: Type) -> Type {
        Type::Tensor(Box::new(a), Box::new(b))
    }
    
    pub fn hom(a: Type, b: Type) -> Type {
        Type::Hom(Box::new(a), Box::new(b))
    }
    
    /// Pretty print: A ⊗ (B ⊸ C)
    pub fn display(&self) -> String {
        match self {
            Type::Base(s) => s.clone(),
            Type::Tensor(a, b) => format!("{} ⊗ {}", a.display(), b.display()),
            Type::Hom(a, b) => format!("({} ⊸ {})", a.display(), b.display()),
            Type::Unit => "I".to_string(),
        }
    }
}
```

### Type Checking Rules

A diagram is well-typed if:

1. **Boundary consistency**: Every input wire has a declared type, every output wire connects to output boundary
2. **Port matching**: For each wire, source port type equals target port type
3. **Linearity**: Every port is connected to exactly one wire (no dangling, no duplication)
4. **Node signature**: Each box's actual input/output wires match its declared signature

```rust
pub struct Checker {
    /// Known morphism signatures (name -> (inputs, outputs))
    signatures: HashMap<String, (Vec<Type>, Vec<Type>)>,
}

impl Checker {
    pub fn check(&self, diagram: &Diagram) -> Vec<Diagnostic> {
        let mut errors = vec![];
        
        // Check port connectivity
        let port_connections = self.build_connection_map(diagram);
        
        for node in &diagram.nodes {
            // Check each input port has exactly one incoming wire
            for (i, input) in node.inputs.iter().enumerate() {
                match port_connections.get(&(node.id.clone(), PortKind::Input, i)) {
                    None => errors.push(Diagnostic::unconnected_input(&node.id, i)),
                    Some(wires) if wires.len() > 1 => {
                        errors.push(Diagnostic::multiple_connections(&node.id, i))
                    }
                    _ => {}
                }
            }
            
            // Check each output port has exactly one outgoing wire
            for (i, output) in node.outputs.iter().enumerate() {
                match port_connections.get(&(node.id.clone(), PortKind::Output, i)) {
                    None => errors.push(Diagnostic::unconnected_output(&node.id, i)),
                    Some(wires) if wires.len() > 1 => {
                        errors.push(Diagnostic::multiple_connections(&node.id, i))
                    }
                    _ => {}
                }
            }
        }
        
        // Check wire type compatibility
        for wire in &diagram.wires {
            let source_type = self.resolve_port_type(diagram, &wire.source);
            let target_type = self.resolve_port_type(diagram, &wire.target);
            
            if source_type != target_type {
                errors.push(Diagnostic::type_mismatch(
                    &wire.id,
                    &source_type,
                    &target_type,
                ));
            }
        }
        
        errors
    }
}
```

---

## Code Generation Strategy

### General Approach

1. Topologically sort nodes (inputs first, outputs last)
2. For each node, generate a let-binding or expression
3. Wire connections become variable references
4. Tensor products become tuples/pairs
5. Output boundary becomes return expression

### Rust Codegen

```rust
impl RustCodegen {
    pub fn generate(&self, diagram: &Diagram, fn_name: &str) -> String {
        let mut code = String::new();
        
        // Function signature
        let input_type = self.types_to_rust(&diagram.boundary.inputs);
        let output_type = self.types_to_rust(&diagram.boundary.outputs);
        code.push_str(&format!(
            "fn {}(input: {}) -> {} {{\n",
            fn_name, input_type, output_type
        ));
        
        // Destructure input
        code.push_str(&self.destructure_input(&diagram.boundary.inputs));
        
        // Generate body in topological order
        let sorted = self.topological_sort(diagram);
        for node_id in sorted {
            if node_id == "input" || node_id == "output" {
                continue;
            }
            let node = diagram.get_node(&node_id).unwrap();
            code.push_str(&self.generate_node(diagram, node));
        }
        
        // Construct output
        code.push_str(&self.construct_output(diagram));
        code.push_str("}\n");
        
        code
    }
    
    fn types_to_rust(&self, types: &[Type]) -> String {
        match types.len() {
            0 => "()".to_string(),
            1 => self.type_to_rust(&types[0]),
            _ => {
                let inner: Vec<_> = types.iter()
                    .map(|t| self.type_to_rust(t))
                    .collect();
                format!("({})", inner.join(", "))
            }
        }
    }
    
    fn type_to_rust(&self, ty: &Type) -> String {
        match ty {
            Type::Base(name) => name.clone(),
            Type::Tensor(a, b) => {
                format!("({}, {})", self.type_to_rust(a), self.type_to_rust(b))
            }
            Type::Hom(a, b) => {
                format!("impl Fn({}) -> {}", self.type_to_rust(a), self.type_to_rust(b))
            }
            Type::Unit => "()".to_string(),
        }
    }
}
```

### Example Transformation

Diagram:
```
  A    B
  │    │
  ├────┤
  │ f  │   f: A ⊗ B → C
  ├────┤
     │
     C
     │
  ┌──┴──┐
  │  g  │   g: C → D ⊗ E
  └──┬──┘
     │
  ┌──┴──┐
  D     E
```

Generated Rust:
```rust
fn diagram(input: (A, B)) -> (D, E) {
    let (input_0, input_1) = input;
    let node_f_out = f(input_0, input_1);
    let (node_g_out_0, node_g_out_1) = g(node_f_out);
    (node_g_out_0, node_g_out_1)
}
```

---

## Rewrite System

### Available Rewrites

| Name | Pattern | Description |
|------|---------|-------------|
| `identity-left` | `id ; f` → `f` | Remove left identity |
| `identity-right` | `f ; id` → `f` | Remove right identity |
| `associativity-left` | `(A ⊗ B) ⊗ C` → `A ⊗ (B ⊗ C)` | Re-associate left |
| `associativity-right` | `A ⊗ (B ⊗ C)` → `(A ⊗ B) ⊗ C` | Re-associate right |
| `braiding` | `σ ; σ` → `id` | Cancel double swap |
| `unit-left` | `I ⊗ A` → `A` | Remove left unit |
| `unit-right` | `A ⊗ I` → `A` | Remove right unit |

### Rewrite Implementation

```rust
pub struct RewriteEngine {
    rules: Vec<RewriteRule>,
}

pub struct RewriteRule {
    pub name: String,
    pub pattern: Pattern,
    pub replacement: Pattern,
}

impl RewriteEngine {
    pub fn applicable_rewrites(&self, diagram: &Diagram, selection: &[NodeId]) -> Vec<&RewriteRule> {
        self.rules.iter()
            .filter(|rule| rule.matches(diagram, selection))
            .collect()
    }
    
    pub fn apply(&self, diagram: &Diagram, rule: &RewriteRule, selection: &[NodeId]) -> Result<Diagram, RewriteError> {
        // 1. Extract subdiagram at selection
        // 2. Match pattern
        // 3. Substitute replacement
        // 4. Reconnect wires
        // 5. Return new diagram
        todo!()
    }
}
```

---

## UI Components

### Canvas Interactions

| Action | Input | Result |
|--------|-------|--------|
| Pan | Middle-drag or Space+drag | Move viewport |
| Zoom | Scroll wheel | Zoom in/out |
| Select node | Click | Select single node |
| Multi-select | Shift+click or drag rectangle | Select multiple |
| Move node | Drag selected | Move node(s) |
| Start wire | Drag from port | Begin wire drawing |
| Complete wire | Drop on compatible port | Create wire |
| Cancel wire | Drop on empty or Escape | Cancel wire drawing |
| Delete | Backspace/Delete | Remove selected |
| Undo | Ctrl+Z | Undo last action |
| Redo | Ctrl+Shift+Z | Redo |

### Node Palette

Standard library of morphisms available from sidebar:

```
┌─────────────────────┐
│  Node Palette       │
├─────────────────────┤
│  ▸ Structural       │
│    ├ identity       │
│    ├ braiding       │
│    ├ associator     │
│    └ unitor         │
│  ▸ User Defined     │
│    ├ f              │
│    └ g              │
│  ▸ Import...        │
└─────────────────────┘
```

Drag from palette to canvas to create node.

---

## Development Setup

### Prerequisites

- Node.js 18+
- Rust 1.75+
- VS Code 1.85+

### Initial Setup

```bash
# Clone repository
git clone https://github.com/user/string-diagrams-vscode.git
cd string-diagrams-vscode

# Install extension dependencies
cd extension && npm install && cd ..

# Install editor dependencies  
cd editor && npm install && cd ..

# Build language server
cd language-server && cargo build && cd ..

# Build everything
./scripts/build.sh
```

### Development Mode

```bash
# Terminal 1: Watch editor
cd editor && npm run dev

# Terminal 2: Watch extension
cd extension && npm run watch

# Terminal 3: Watch language server
cd language-server && cargo watch -x build

# Then press F5 in VS Code to launch Extension Development Host
```

### Testing

```bash
# Rust tests
cd language-server && cargo test

# Extension tests
cd extension && npm test

# Editor tests
cd editor && npm test
```

---

## Roadmap

### Phase 1: Skeleton ✓
- [x] Project structure
- [ ] VS Code extension scaffolding
- [ ] Custom editor opens .diagram files
- [ ] Basic webview with canvas
- [ ] Rust language server responds to requests

### Phase 2: Basic Editing
- [ ] Node rendering (boxes with ports)
- [ ] Node dragging
- [ ] Wire drawing interaction
- [ ] Wire rendering (bezier curves)
- [ ] Save/load .diagram files

### Phase 3: Type Checking
- [ ] Type grammar and parser
- [ ] Type checking algorithm
- [ ] Diagnostic reporting
- [ ] Visual error feedback (red wires/ports)

### Phase 4: Node Library
- [ ] Node palette UI
- [ ] Built-in structural morphisms
- [ ] User-defined node creation
- [ ] Import/export node libraries

### Phase 5: Code Generation
- [ ] Rust codegen
- [ ] TypeScript codegen
- [ ] "Compile" command
- [ ] Generated code preview

### Phase 6: Rewrites
- [ ] Rewrite rule definitions
- [ ] Pattern matching on subdiagrams
- [ ] "Apply rewrite" command
- [ ] Visual rewrite preview

### Phase 7: Polish
- [ ] Undo/redo
- [ ] Copy/paste
- [ ] Auto-layout
- [ ] Better styling
- [ ] Documentation

---

## Design Decisions Log

### Why VS Code?

- Custom editor API provides canvas + document model
- LSP gives us language server infrastructure for free
- Users already have it installed
- Extension marketplace for distribution

### Why Rust for Language Server?

- Performance matters for large diagrams
- Type system aligns with categorical types we're implementing
- Can share code with future CLI tool
- Ibrahim's expertise

### Why React for Webview?

- Familiar component model
- Good SVG support
- Can use existing hooks for interactions
- Fast iteration during development

### Why JSON for Diagram Format?

- Human readable
- Versionable in git
- Easy to parse in both TS and Rust
- Can be hand-edited in emergencies

### Why SVG over Canvas?

- DOM elements for accessibility
- CSS for styling
- Event handling per-element
- Easier hit testing
- Vector export for free

---

## References

### Categorical Foundations

- Selinger, "A Survey of Graphical Languages for Monoidal Categories" (2009) — comprehensive reference
- Joyal & Street, "The Geometry of Tensor Calculus I" (1991) — original string diagram paper
- Baez & Stay, "Physics, Topology, Logic and Computation: A Rosetta Stone" (2009) — accessible intro

### Prior Art

- **Quantomatic** — quantum string diagram editor, academic
- **homotopy.io** — higher-dimensional diagrams, web-based
- **Globular** — another higher-dimensional tool
- **Catlab.jl** — computational CT in Julia (not visual, but good API reference)

### VS Code Extension Development

- [Custom Editors API](https://code.visualstudio.com/api/extension-guides/custom-editors)
- [Webview API](https://code.visualstudio.com/api/extension-guides/webview)
- [Language Server Protocol](https://microsoft.github.io/language-server-protocol/)

### Rust LSP

- [tower-lsp](https://github.com/ebkalderon/tower-lsp) — LSP server framework
- [lsp-types](https://github.com/gluon-lang/lsp-types) — LSP type definitions

---

## Open Questions

1. **How to handle recursive/traced diagrams?** Feedback loops require traced monoidal categories. Defer to v2?

2. **What's the right granularity for undo?** Each wire? Each node? Each "transaction"?

3. **How to represent partially-drawn diagrams?** During wire drawing, diagram is temporarily invalid.

4. **Should rewrites be automatic or manual?** Could offer "simplify" button that applies all safe rewrites.

5. **How to handle higher-order morphisms?** Boxes whose inputs/outputs are themselves morphisms (currying).

---

## Contributing

See CONTRIBUTING.md (to be written).

Key principles:
- Type safety in both TS and Rust code
- Tests for categorical invariants
- Documentation for non-obvious categorical concepts
- Accessibility in UI components
