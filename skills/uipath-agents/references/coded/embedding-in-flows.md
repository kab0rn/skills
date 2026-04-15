# Coded Agents in Flow Projects (Sibling Folder)

A coded agent can live as a sibling folder to a flow project inside the same solution. The flow references it via `uipath.core.agent.<resourceKey>` with `section: "In this solution"` — resolved by Studio Web after a single `uip solution upload`, no separate Orchestrator deployment.

For the published-agent path, see [coded/flow-integration.md § Pattern 2](flow-integration.md#pattern-2-published-coded-agent).

## Sibling Folder Structure

The `<resourceKey>` is a local UUID written to `resources/solution_folder/process/agent/<CodedAgentProject>.json` by `uip solution project add`, so it is stable the moment the agent joins the solution. The `--local` flag on `uip maestro flow registry list`/`get` reads those files to surface sibling projects as discoverable node types — no upload round-trip needed.

```
<SolutionDir>/
├── <SolutionName>.uipx
├── resources/                      # created and maintained by `uip solution project add`
│   └── solution_folder/
│       ├── process/agent/<CodedAgentProject>.json   # holds the `resource.key` UUID read by --local
│       └── package/<CodedAgentProject>.json
├── <FlowProject>/
│   ├── <FlowName>.flow
│   └── project.uiproj
└── <CodedAgentProject>/
    ├── main.py
    ├── pyproject.toml
    ├── uipath.json
    ├── entry-points.json
    ├── project.uiproj
    └── .venv/                      # local only, excluded from upload
```

### Key differences from a published coded agent

- **`resource.key`** is minted locally at `uip solution project add` time, not by Orchestrator at `uip codedagent deploy` time
- **No `codedagent deploy`** — the agent ships inside the solution package, uploaded together with the flow via `uip solution upload`
- **Registry discovery is `--local`** (reads `resources/solution_folder/process/` files); no login or `registry pull` required
- **`model.section`** is `"In this solution"`

## Creating the Sibling-Folder Coded Agent

1. From the solution directory, create the agent folder and scaffold inside it:

   ```bash
   mkdir <CodedAgentProject>
   cd <CodedAgentProject>
   uv sync
   source .venv/bin/activate       # .venv/Scripts/activate on Windows
   uip codedagent setup --output json
   uip codedagent new <agent-name>
   ```

2. Implement `main.py`. Use lazy LLM initialization (create clients inside functions, never at module level).

3. Generate entry points:

   ```bash
   uip codedagent init
   ```

4. Register the agent in the solution — **this is the step that mints the `resource.key`** the flow will reference:

   ```bash
   cd <SolutionDir>
   uip solution project add <CodedAgentProject> <SolutionName>.uipx --output json
   ```

   After this command, `resources/solution_folder/process/agent/<CodedAgentProject>.json` holds the `resource.key` UUID that the flow's `--local` discovery surfaces in the next step.

### Flow Wiring

From inside the flow project directory, discover the agent via the in-solution registry. The `--local` flag tells the registry commands to walk up from CWD to find a `.uipx`, read the `resources/solution_folder/process/` files written by step 4, and surface every sibling project as a discoverable node type — all offline, without a tenant registry round-trip:

```bash
# list every in-solution node type (all sibling projects)
uip maestro flow registry list --local --output json

# fetch the full manifest for the coded agent
uip maestro flow registry get "uipath.core.agent.<resourceKey>" --local --output json
```

The second command's `Data.Node` object is what you paste into the flow's `definitions[]`. Build the node instance and top-level `bindings[]` entries using the shape documented in [agent/impl.md § In-solution variant](../../../uipath-maestro-flow/references/plugins/agent/impl.md#node-instance-inside-nodes--in-solution-variant).

Without `--local`, `registry list`/`get` query the tenant registry (Orchestrator-published resources only) and will not surface the sibling project.

## Upload the Solution

`uip solution upload` pushes the coded agent's files alongside the flow; Studio Web resolves the flow node's `type: "uipath.core.agent.<resourceKey>"` via the uploaded `resources/solution_folder/process/agent/*.json` file.

Before running the upload, move `.venv/` outside the project directory so it isn't bundled into the solution archive. Move it back afterward — `pyproject.toml` + `uv.lock` let `uv sync` recreate it on demand.

```bash
mv <CodedAgentProject>/.venv /tmp/<CodedAgentProject>-venv
uip solution upload <SolutionDir> --output json
mv /tmp/<CodedAgentProject>-venv <CodedAgentProject>/.venv
```

Subsequent edits to the agent's Python or the flow go back through `uip solution upload`.
