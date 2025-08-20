# Kong Deck State Management Strategy

## Current Setup

This project uses an **OpenAPI-first approach** with selective state management:

### 1. **Configuration Generation Flow**
```
OpenAPI Spec (items-oas.yaml) 
    ↓ (deck file openapi2kong)
Generated Config 
    ↓ (deck file patch - patch.yaml)
Add Control Plane + Service Config + Kong 3.0 Paths + Tags
    ↓ (deck file add-plugins)
Add Plugins 
    ↓ (deck file patch - add-consumers.yaml)
Add Consumers + Tags
    ↓ (deck file patch - add-tags.yaml)
Add Route Tags
    ↓ (deck gateway sync --select-tag)
Deploy Only Tagged Resources
```

### 2. **Tag Strategy**
All generated resources are tagged with:
- `managed-by-deck`: Identifies resources managed by this workflow
- `generated-from-oas`: Identifies resources generated from OpenAPI spec
- `team:backend`: Identifies team ownership

### 3. **Selective Sync**
The workflow uses `--select-tag managed-by-deck` to:
- ✅ Only manage resources with this tag
- ✅ Preserve existing untagged resources (like simple-api)
- ✅ Avoid accidental deletions
- ✅ Enable multiple teams to manage different resources

### 4. **Kong 3.0 Compatibility**
- Regex route paths are prefixed with `~` for Kong 3.0+ compatibility
- Format version is set to `3.0`

### 5. **Safety Features**
- Configuration backup before changes
- Diff preview before sync
- Validation of generated config
- Rollback on failure
- Selective sync prevents unintended deletions

## Usage

### Local Development
```bash
# Generate config from OpenAPI
deck file openapi2kong --spec config/items-oas.yaml --output-file config/items-deck.yaml

# Apply patches
deck file patch -s config/items-deck.yaml config/patch.yaml -o config/items-deck.yaml
deck file add-plugins -s config/items-deck.yaml config/add-plugins.yaml -o config/items-deck.yaml
deck file patch -s config/items-deck.yaml config/add-consumers.yaml -o config/items-deck.yaml
deck file patch -s config/items-deck.yaml config/add-tags.yaml -o config/items-deck.yaml

# Preview changes
deck gateway diff --select-tag managed-by-deck config/items-deck.yaml

# Apply changes
deck gateway sync --select-tag managed-by-deck config/items-deck.yaml
```

### Adding New Resources
1. Update `items-oas.yaml` with new endpoints
2. Modify patch files if needed
3. Commit and push - CI/CD will handle deployment

### Managing Other Resources
Resources without `managed-by-deck` tag are ignored by this workflow.
Use different tags for different teams/services:
```bash
deck gateway sync --select-tag team:frontend config/frontend.yaml
deck gateway sync --select-tag team:backend config/backend.yaml
```
