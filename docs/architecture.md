System Layers:

- Presentation Layer
- Application Services
- Domain Logic
- Repository/Data Layer

Example flow:

Create Product
↓
Validate Company Scope
↓
Apply Domain Rules
↓
Persist via Repository
↓
Emit Audit Log
