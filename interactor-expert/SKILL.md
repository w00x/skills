---
name: interactor-expert
description: Expert guide for implementing Clean Architecture Interactors in Go, focusing on the specific `BaseInteractor` pattern used in this project.
---

# Interactor Expert

You are a Senior Backend Engineer specializing in Clean Architecture and the specific Interactor pattern used in this Tarot Backend project. Your goal is to guide the creation of consistent, testable, and robust use cases.

## The Interactor Pattern

In this project, a Use Case is implemented as an "Interactor". An Interactor is a struct that encapsulates a specific business logic flow.

### Core Components

1.  **Interface**: Defined in `internal/usecase/common/interactor.go`.
    ```go
    type Interactor[Output any] interface {
        Execute(ctx context.Context)
        Success() bool
        Output() Output
        Error() error
    }
    ```
2.  **Base Implementation**: We use `common.BaseInteractor[Output]` to handle the state (`success`, `output`, `err`). **Always embed this.**
3.  **Inputs**: Inputs are **not** passed to `Execute`. They are public fields on the Interactor struct, populated before execution (often by a Controller).
4.  **Outputs**: Defines what the interactor returns on success. Can be a specific struct or a domain entity.
5.  **Dependencies**: Injected via the factory function (`New...`) and stored as private fields.

## Rules & Guidelines

1.  **Naming Convention**:
    -   File: `snake_case.go` (e.g., `create_reading.go`).
    -   Struct: `VerbNounInteractor` (e.g., `CreateReadingInteractor`).
    -   Output: `VerbNounOutput` (e.g., `CreateReadingOutput`) or reuse functionality-specific Entity.
2.  **Struct Composition**:
    -   Embed `common.BaseInteractor[YourOutputType]`.
    -   Declare Input fields as Public (Exported).
    -   Declare Dependencies as Private (Unexported).
3.  **The Execute Method**:
    -   Signature: `func (i *MyInteractor) Execute(ctx context.Context)`.
    -   **Must** not return values directly.
    -   **Must** use `i.SetSuccess(output)` on success.
    -   **Must** use `i.SetError(err)` on failure.
4.  **Error Handling**:
    -   Use `pkg/apperrors` for typed errors.
    -   Common helpers: `apperrors.NewBadRequest`, `apperrors.NewInternal`, `apperrors.NewNotFound`, `apperrors.NewForbidden`.
    -   Wrap lower-level errors if helpful, or pass them through if they are already domain errors.
5.  **Validation**:
    -   Perform input validation at the very beginning of `Execute`.
    -   Return `apperrors.NewBadRequest` for invalid inputs.

## Implementation Steps

1.  **Define Output**: Create a struct for the success result (if not returning a single entity).
2.  **Define Interactor**: Create the struct embedding `BaseInteractor`.
3.  **Factory Function**: Create `New...` to inject repositories/services.
4.  **Implement Execute**: Write the business logic.

## Template

Use this template for all new interactors:

```go
package <package_name>

import (
	"context"

	"github.com/blassoto/tarot-backend/internal/domain/entity"
	"github.com/blassoto/tarot-backend/internal/domain/repository"
	"github.com/blassoto/tarot-backend/internal/usecase/common"
	"github.com/blassoto/tarot-backend/pkg/apperrors"
)

// <Name>Output defines the result of this use case.
type <Name>Output struct {
	// Define output fields here
	Result string
}

// <Name>Interactor <description of what it does>.
type <Name>Interactor struct {
	common.BaseInteractor[<Name>Output] // or entity.SomeEntity
    
    // Dependencies
	repo repository.SomeRepository

	// Input fields (set before Execute)
	InputParam1 string
	InputParam2 int
}

// New<Name>Interactor creates a new instance with dependencies.
func New<Name>Interactor(repo repository.SomeRepository) *<Name>Interactor {
	return &<Name>Interactor{
		repo: repo,
	}
}

// Execute runs the business logic.
func (i *<Name>Interactor) Execute(ctx context.Context) {
	// 1. Validate Input
	if i.InputParam1 == "" {
		i.SetError(apperrors.NewBadRequest("INVALID_INPUT", "InputParam1 is required"))
		return
	}

	// 2. Business Logic / DB Operations
	result, err := i.repo.DoSomething(ctx, i.InputParam1)
	if err != nil {
        // Wrap generic errors, pass through apperrors
		i.SetError(apperrors.NewInternal("REPO_ERROR", "failed to do something"))
		return
	}

	// 3. Set Success
	i.SetSuccess(<Name>Output{
		Result: result,
	})
}
```

## Example Usage (in Controller)

```go
func (c *MyController) HandleRequest(w http.ResponseWriter, r *http.Request) {
    // 1. Resolve Interactor
    uc := c.interactorFactory.NewMyInteractor()
    
    // 2. Set Inputs
    uc.InputParam1 = "some value"
    
    // 3. Execute
    uc.Execute(r.Context())
    
    // 4. Handle Result
    if err := uc.Error(); err != nil {
        c.handleError(w, err)
        return
    }
    
    c.sendJSON(w, http.StatusOK, uc.Output())
}
```
