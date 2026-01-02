# Codebase Exploration Guide: AskVantage

## 1. Title and Overview

**AskVantage** is a .NET Aspire-based educational application that helps students learn by generating questions from images of text (e.g., book pages). The system performs OCR (Optical Character Recognition) on uploaded images, extracts text, and uses AI (OpenAI or local Ollama) to generate educational questions with answers and references.

> **🎯 Aspire Patterns**: For detailed coverage of **.NET Aspire patterns and practices** used in this application, see the separate document: **[ASKVANTAGE_ASPIRE_PATTERNS.md](./ASKVANTAGE_ASPIRE_PATTERNS.md)**

### What It Does Today

1. **Image Upload & OCR**: Users upload JPEG images (max 10MB) containing text, which are processed using Azure Computer Vision OCR service
2. **Text Extraction**: Extracted text is displayed to users and can be manually edited
3. **Question Generation**: Using Semantic Kernel with Prompty templates, the system generates 3 questions per text with answers and references
4. **State Management**: Questions and texts are persisted using Dapr state store (backed by Redis)
5. **Real-time Updates**: SignalR hub provides real-time notifications for OCR completion and question generation completion
6. **Question Management**: Users can view all saved questions, delete individual texts, or clear all data

### Current Limitations

- **Single-column text only**: OCR works best with single-column layouts
- **JPEG format only**: Only JPEG images are accepted
- **No user authentication**: All users share the same state store
- **Fire-and-forget pattern**: Long-running operations (OCR, question generation) use `Task.Run()` which can lead to request context issues
- **No retry logic for external services**: Direct calls to Azure services without built-in retry
- **Limited error handling**: Basic error handling with SignalR notifications
- **No rate limiting**: No protection against excessive API calls
- **State store key management**: Uses a custom "allKeys" pattern instead of Dapr's built-in query capabilities

---

> **📘 Aspire Patterns**: For comprehensive coverage of .NET Aspire patterns and practices, see **[ASKVANTAGE_ASPIRE_PATTERNS.md](./ASKVANTAGE_ASPIRE_PATTERNS.md)**

---

## 2. Object Relationships

```mermaid
classDiagram
    %% Frontend Layer
    class Home_razor {
        +UploadFile()
        +GenerateQuestions()
        +HandleOcrCompleted()
        +HandleGenerationCompleted()
    }
    
    class Questions_razor {
        +GetQuestions()
        +DeleteQuestion()
    }
    
    class ImageService {
        +AnalyzeImage()
        +GenerateQuestions()
        +GetQuestions()
        +DeleteQuestion()
    }
    
    class ImageApiHubClient {
        +StartAsync()
        +OcrCompleted event
        +GenerationCompleted event
    }
    
    %% API Layer - Controllers
    class ImageController {
        +AnalyzeImage() POST /api/image/analyze
    }
    
    class QuestionController {
        +GetAllTexts() GET /api/question
        +GenerateQuestionsForText() POST /api/question/generate
        +DeleteText() DELETE /api/question/{key}
        +DeleteAllTexts() DELETE /api/question
    }
    
    class ImageApiHub {
        <<SignalR Hub>>
    }
    
    %% Service Layer
    class IImageOcrService {
        <<interface>>
        +GetTextAsync()
    }
    
    class AzureImageOcrService {
        -ImageAnalysisClient
        +GetTextAsync()
    }
    
    class IQuestionGeneratorService {
        <<interface>>
        +GenerateQuestions()
    }
    
    class OpenAIQuestionGeneratorService {
        -Kernel
        +GenerateQuestions()
    }
    
    class ITextStateService {
        <<interface>>
        +SaveText()
        +GetAllTexts()
        +GetSingleText()
        +DeleteSingleText()
        +DeleteAllTexts()
    }
    
    class DaprTextStateService {
        -DaprClient
        +SaveText()
        +GetAllTexts()
        +GetSingleText()
        +DeleteSingleText()
        +DeleteAllTexts()
    }
    
    %% Models
    class Image {
        +Guid Id
        +string Name
        +byte[] Content
    }
    
    class ImageOcrResult {
        +Guid ImageId
        +string Text
    }
    
    class QuestionGenerationRequest {
        +Guid RequestId
        +string Text
        +string TextTitle
    }
    
    class QuestionGenerationResult {
        +Guid Id
        +Guid RequestId
        +string OriginalText
        +string TextTitle
        +ImmutableList~QuestionAndAnswer~ QuestionsAndAnswers
    }
    
    class TextState {
        +string Title
        +string Text
        +QuestionState[] Questions
    }
    
    class QuestionState {
        +string Question
        +string Answer
        +string Reference
    }
    
    %% Infrastructure
    class DaprClient {
        <<external>>
        +GetStateAsync()
        +SaveStateAsync()
        +DeleteStateAsync()
        +GetBulkStateAsync()
    }
    
    class ImageAnalysisClient {
        <<Azure>>
        +AnalyzeAsync()
    }
    
    class Kernel {
        <<SemanticKernel>>
        +InvokeAsync()
    }
    
    class Redis {
        <<state store>>
    }
    
    %% Relationships
    Home_razor --> ImageService
    Home_razor --> ImageApiHubClient
    Questions_razor --> ImageService
    
    ImageService --> ImageController : HTTP POST
    ImageService --> QuestionController : HTTP POST/GET/DELETE
    ImageApiHubClient --> ImageApiHub : SignalR
    
    ImageController --> IImageOcrService
    ImageController --> ImageApiHub
    QuestionController --> IQuestionGeneratorService
    QuestionController --> ITextStateService
    QuestionController --> ImageApiHub
    
    ImageApiHub --> IImageApiHubClient : SignalR events
    
    AzureImageOcrService ..|> IImageOcrService
    AzureImageOcrService --> ImageAnalysisClient
    
    OpenAIQuestionGeneratorService ..|> IQuestionGeneratorService
    OpenAIQuestionGeneratorService --> Kernel
    
    DaprTextStateService ..|> ITextStateService
    DaprTextStateService --> DaprClient
    
    DaprClient --> Redis
    ImageAnalysisClient --> Azure : External API
    
    ImageController --> Image
    ImageController --> ImageOcrResult
    QuestionController --> QuestionGenerationRequest
    QuestionController --> QuestionGenerationResult
    DaprTextStateService --> TextState
    TextState --> QuestionState
    QuestionGenerationResult --> QuestionAndAnswer
```

---

## 3. Learning Path (Progressive)

### **Start Here**: Understanding the User Flow

1. **User Journey** (`src/AskVantage/AskVantage.Frontend.Client/Pages/Home.razor`)
   - User uploads an image → `UploadFile()` method
   - Image is sent to API → `ImageService.AnalyzeImage()`
   - OCR result comes back via SignalR → `HandleOcrCompleted()`
   - User can edit text and click "Generate Questions"
   - Questions arrive via SignalR → `HandleGenerationCompleted()`

2. **API Entry Points** (`src/AskVantage/Apis/ImageApi/Controllers/`)
   - `ImageController.AnalyzeImage()` - Accepts image, fires async OCR
   - `QuestionController.GenerateQuestionsForText()` - Accepts text, fires async question generation
   - Both return `202 Accepted` immediately and notify via SignalR

### **Next**: Understanding Service Layer

1. **OCR Service** (`src/AskVantage/Apis/ImageApi/Services/AzureImageOcrService.cs`)
   - Wraps Azure Computer Vision `ImageAnalysisClient`
   - Extracts text from image bytes using `VisualFeatures.Read`
   - Joins all lines with commas (could be improved)

2. **Question Generation Service** (`src/AskVantage/Apis/ImageApi/Services/OpenAIQuestionGeneratorService.cs`)
   - Uses Semantic Kernel with Prompty templates
   - Invokes `GenerateQuestions` function from Prompty plugin
   - Parses JSON response using regex (fragile - could fail on malformed JSON)
   - Returns `QuestionAnswerResponse[]`

3. **State Management Service** (`src/AskVantage/Apis/ImageApi/Services/DaprTextStateService.cs`)
   - Uses Dapr state store (Redis-backed)
   - Maintains custom "allKeys" list for enumeration
   - Merges new questions into existing `TextState` if title matches
   - Uses `LastWrite` concurrency mode and `Strong` consistency

### **Advanced**: Infrastructure & Configuration

1. **Aspire AppHost** (`src/AskVantage/Aspire/AskVantage.AppHost/Program.cs`)
   - Orchestrates all services: Redis, Dapr, ImageApi, Frontend
   - Configures Dapr sidecar for ImageApi
   - Supports local Ollama or Azure OpenAI
   - Sets up service discovery and health checks

2. **Program.cs Setup** (`src/AskVantage/Apis/ImageApi/Program.cs`)
   - Configures Dapr client with 10MB message size limits
   - Sets up Semantic Kernel with Prompty functions
   - Configures JSON serialization (handles .NET 10 serialization quirks)
   - Registers SignalR hub

3. **Prompty Templates** (`src/AskVantage/Apis/ImageApi/Prompts/GenerateQuestions.prompty`)
   - Defines AI prompt for question generation
   - Uses different templates for local vs cloud (`GenerateQuestions.local.prompty`)
   - Returns JSON array of questions/answers/references

### **What You Should Understand**

Before making changes, ensure you understand:

- ✅ How SignalR hub events flow from server to client
- ✅ How Dapr state store keys are managed (custom "allKeys" pattern)
- ✅ How `Task.Run()` is used for fire-and-forget operations
- ✅ How Semantic Kernel Prompty functions are loaded and invoked
- ✅ How service discovery works in Aspire (services reference each other by name)

---

## 4. Code Map (with Precise References)

### Entry Points

| File | Purpose | Key Methods |
| ------ | --------- | ------------- |
| `src/AskVantage/AskVantage.Frontend.Client/Pages/Home.razor` | Main UI page | `UploadFile()`, `GenerateQuestions()`, `HandleOcrCompleted()`, `HandleGenerationCompleted()` |
| `src/AskVantage/Apis/ImageApi/Controllers/ImageController.cs` | Image OCR API | `AnalyzeImage()` - POST `/api/image/analyze` |
| `src/AskVantage/Apis/ImageApi/Controllers/QuestionController.cs` | Question management API | `GetAllTexts()` - GET `/api/question`, `GenerateQuestionsForText()` - POST `/api/question/generate`, `DeleteText()` - DELETE `/api/question/{key}` |

### Service Layer

| File | Purpose | Key Methods |
| ------ | --------- | ------------- |
| `src/AskVantage/Apis/ImageApi/Services/AzureImageOcrService.cs` | Azure OCR wrapper | `GetTextAsync(byte[] image, CancellationToken)` |
| `src/AskVantage/Apis/ImageApi/Services/OpenAIQuestionGeneratorService.cs` | AI question generation | `GenerateQuestions(string input, CancellationToken)` |
| `src/AskVantage/Apis/ImageApi/Services/DaprTextStateService.cs` | State persistence | `SaveText()`, `GetAllTexts()`, `GetSingleText()`, `DeleteSingleText()`, `DeleteAllTexts()` |
| `src/AskVantage/AskVantage.Frontend.Client/Services/ImageService.cs` | Frontend HTTP client | `AnalyzeImage()`, `GenerateQuestions()`, `GetQuestions()`, `DeleteQuestion()` |

### Models & Data Contracts

| File | Purpose | Key Properties |
| ------ | --------- | ------------- |
| `src/AskVantage/Apis/ImageApi/Models/Image.cs` | Image upload request | `Id`, `Name`, `Content` (byte[]) |
| `src/AskVantage/Apis/ImageApi/Models/ImageOcrResult.cs` | OCR response | `ImageId`, `Text` |
| `src/AskVantage/Apis/ImageApi/Models/QuestionGenerationRequest.cs` | Question generation request | `RequestId`, `Text`, `TextTitle` |
| `src/AskVantage/Apis/ImageApi/Models/QuestionGenerationResult.cs` | Question generation response | `Id`, `RequestId`, `OriginalText`, `TextTitle`, `QuestionsAndAnswers` |
| `src/AskVantage/Apis/ImageApi/Services/DaprTextStateService.cs` | Internal state models | `TextState`, `QuestionState` |

### Infrastructure & Configuration

| File | Purpose | Key Configuration |
| ------ | --------- | ------------- |
| `src/AskVantage/Aspire/AskVantage.AppHost/Program.cs` | Application orchestration | Redis, Dapr, ImageApi, Frontend setup |
| `src/AskVantage/Apis/ImageApi/Program.cs` | API startup | Dapr client, Semantic Kernel, SignalR, JSON serialization |
| `src/AskVantage/Aspire/AskVantage.AppHost/DaprComponents/statestore.yaml` | Dapr state store config | Redis connection (localhost:6380) |
| `src/AskVantage/Apis/ImageApi/Prompts/GenerateQuestions.prompty` | AI prompt template | System prompt, JSON schema, examples |

### SignalR & Real-time Communication

| File | Purpose | Key Components |
| ---- | ------- | -------------- |
| `src/AskVantage/Apis/ImageApi/Hubs/ImageApiHub.cs` | SignalR hub definition | `ImageApiHub`, `IImageApiHubClient` interface |
| `src/AskVantage/AskVantage.Frontend.Client/Services/IImageApiHubClient.cs` | Frontend hub client | `ImageApiHubClient`, event handlers |

### Frontend Components

| File | Purpose | Key Features |
| ------ | --------- | ------------- |
| `src/AskVantage/AskVantage.Frontend.Client/Pages/Home.razor` | Main page | Image upload, OCR display, question generation, quiz display |
| `src/AskVantage/AskVantage.Frontend.Client/Pages/Questions.razor` | Questions management | List all saved questions, delete functionality |
| `src/AskVantage/AskVantage.Frontend/Program.cs` | Frontend server | Blazor Server + WebAssembly, YARP forwarding to ImageApi |

---

## 5. Key Concepts

### Domain Terminology

- **TextState**: Internal representation of a text with its title, content, and associated questions
- **QuestionState**: A single question-answer pair with a reference to the source text
- **OCR**: Optical Character Recognition - extracting text from images
- **Prompty**: Semantic Kernel's prompt template format (YAML-based)
- **Dapr State Store**: Distributed state management abstraction (backed by Redis in this app)

### Non-Obvious Behaviors & Patterns

1. **Fire-and-Forget Pattern**: Both `ImageController.AnalyzeImage()` and `QuestionController.GenerateQuestionsForText()` use `Task.Run()` to execute long-running operations asynchronously. This means:
   - The HTTP request returns `202 Accepted` immediately
   - Results are delivered via SignalR hub events
   - No direct HTTP response with results
   - Request context is lost in `Task.Run()` (uses `IServiceScopeFactory` to create new scope)

2. **Custom Key Management**: `DaprTextStateService` maintains a separate "allKeys" entry in the state store to track all text titles. This is because Dapr doesn't provide a built-in "list all keys" operation for Redis state stores.

3. **Question Merging**: When saving a `TextState`, if a text with the same title already exists, new questions are appended to the existing list (no duplicates based on question text comparison).

4. **Prompty Function Loading**: The system loads different Prompty files based on `RunLocal` configuration:
   - `GenerateQuestions.prompty` - for Azure OpenAI
   - `GenerateQuestions.local.prompty` - for local Ollama
   - Files are loaded at startup and registered as Semantic Kernel functions

5. **JSON Parsing Fragility**: `OpenAIQuestionGeneratorService` uses regex to extract JSON from the LLM response. This is fragile and could fail if the LLM returns malformed JSON or includes extra text.

6. **SignalR Hub Naming**: The hub is mapped at `/hub` in `Program.cs`, and the frontend connects to `{baseUri}/hub`. The interface `IImageApiHubClient` defines the contract for both server and client.

7. **Service Discovery**: In Aspire, services reference each other by name (e.g., `"http://imageapi"`). The frontend uses YARP to forward `/api/image/*` and `/api/question/*` requests to the ImageApi service.

---

## 6. Data Flow

### Image Upload & OCR Flow

```text
1. User uploads image (Home.razor)
   ↓
2. ImageService.AnalyzeImage() → POST /api/image/analyze
   ↓
3. ImageController.AnalyzeImage() receives Image model
   ↓
4. Task.Run() fires async operation
   ↓
5. AzureImageOcrService.GetTextAsync()
   - Calls Azure Computer Vision API
   - Extracts text from image
   ↓
6. ImageApiHub.OcrCompleted() → SignalR event
   ↓
7. ImageApiHubClient.OcrCompleted event → Home.razor.HandleOcrCompleted()
   ↓
8. UI updates with recognized text
```

### Question Generation Flow

```text
1. User clicks "Generate Questions" (Home.razor)
   ↓
2. ImageService.GenerateQuestions() → POST /api/question/generate
   ↓
3. QuestionController.GenerateQuestionsForText() receives request
   ↓
4. Task.Run() fires async operation (creates new service scope)
   ↓
5. OpenAIQuestionGeneratorService.GenerateQuestions()
   - Invokes Semantic Kernel function
   - LLM generates questions/answers/references
   - Parses JSON response
   ↓
6. DaprTextStateService.SaveText()
   - Checks if TextState exists for title
   - Merges new questions if exists, creates new if not
   - Updates "allKeys" list
   - Saves to Dapr state store (Redis)
   ↓
7. DaprTextStateService.GetSingleText() - fetches complete state
   ↓
8. TextStateExtensions.BuildResponse() - maps to QuestionGenerationResult
   ↓
9. ImageApiHub.GenerationCompleted() → SignalR event
   ↓
10. ImageApiHubClient.GenerationCompleted event → Home.razor.HandleGenerationCompleted()
   ↓
11. UI updates with questions
```

### Question Retrieval Flow

```text
1. User navigates to /questions (Questions.razor)
   ↓
2. ImageService.GetQuestions() → GET /api/question
   ↓
3. QuestionController.GetAllTexts()
   ↓
4. DaprTextStateService.GetAllTexts()
   - Gets "allKeys" from state store
   - Bulk fetches all TextState objects
   ↓
5. TextStateExtensions.BuildResponse() for each text
   ↓
6. Returns IEnumerable<QuestionGenerationResult>
   ↓
7. UI displays all questions grouped by text title
```

### Data Transformations

- **Image (byte[]) → Text (string)**: Azure OCR extracts text
- **Text (string) → Questions (JSON)**: LLM generates structured questions
- **JSON (string) → QuestionAnswerResponse[]**: Regex parsing in `OpenAIQuestionGeneratorService`
- **TextState → QuestionGenerationResult**: Mapping via `TextStateExtensions.BuildResponse()`
- **QuestionGenerationRequest → TextState**: Created in `QuestionController` before saving

### Caching Points

- **None currently**: All state is persisted in Dapr/Redis, but there's no in-memory caching layer
- **Potential optimization**: Cache frequently accessed `TextState` objects in memory

---

## 7. Questions to Answer (Checklist)

### Data Flow

- [ ] How are large images (up to 10MB) handled in memory? Are they streamed or buffered?
- [ ] What happens if OCR fails mid-process? Is the error propagated correctly?
- [ ] How are concurrent question generation requests handled? Can they overwrite each other?
- [ ] What is the maximum size of text that can be sent for question generation?
- [ ] How does the "allKeys" list stay consistent if a save operation fails?

### Caching

- [ ] Is there any caching of OCR results? (Currently: No)
- [ ] Are generated questions cached? (Currently: No, regenerated each time)
- [ ] How does Redis state store handle expiration? (Not configured - data persists indefinitely)

### Dependencies

- [ ] What happens if Azure Computer Vision API is unavailable?
- [ ] What happens if OpenAI/Azure OpenAI API is unavailable?
- [ ] What happens if Redis/Dapr state store is unavailable?
- [ ] How are API keys and secrets managed? (User secrets, appsettings, Aspire parameters)
- [ ] What is the fallback behavior when `RunLocal` is true but Ollama is not running?

### Lifecycle

- [ ] When are SignalR connections established? (On page initialization)
- [ ] When are SignalR connections closed? (On page disposal)
- [ ] How are service scopes managed in `Task.Run()` operations? (New scope created via `IServiceScopeFactory`)
- [ ] What happens to in-flight operations when the application shuts down?
- [ ] How are Prompty functions loaded? (At startup, from `Prompts/` directory)

### Threading

- [ ] Why is `Task.Run()` used instead of async/await? (Fire-and-forget pattern, but loses request context)
- [ ] Are there any thread-safety concerns with the state service? (Dapr client should be thread-safe, but "allKeys" updates could race)
- [ ] How are SignalR hub context operations thread-safe? (SignalR handles concurrency)

### Error Handling

- [ ] How are OCR errors communicated to the user? (SignalR `OcrFailed` event)
- [ ] How are question generation errors communicated? (SignalR `GenerationFailed` event)
- [ ] What happens if JSON parsing fails in `OpenAIQuestionGeneratorService`? (Throws `InvalidOperationException`)
- [ ] Are there retry policies for external API calls? (No, direct calls without Polly/retry)
- [ ] How are Dapr state store errors handled? (Logged and re-thrown)

---

## 8. Next Steps

### Aspire-Specific Learning Path

> **See [ASKVANTAGE_ASPIRE_PATTERNS.md](./ASKVANTAGE_ASPIRE_PATTERNS.md)** for detailed Aspire learning paths and patterns.

### Application-Specific Investigation

1. **Review SignalR Connection Management**
   - Check if connections are properly disposed
   - Verify reconnection handling in production scenarios
   - Test behavior when multiple browser tabs are open

2. **Examine State Store Consistency**
   - Review the "allKeys" update logic for race conditions
   - Consider using Dapr's transactional state API for atomic updates
   - Test concurrent save operations

3. **Analyze Error Scenarios**
   - Test Azure OCR failure scenarios
   - Test OpenAI API failure scenarios
   - Test Redis connection failures
   - Verify error messages are user-friendly

4. **Review Performance Characteristics**
   - Measure OCR processing time for large images
   - Measure question generation time for long texts
   - Check memory usage with multiple concurrent requests
   - Profile Dapr state store operations

5. **Examine Security Considerations**
   - Review input validation (image size, text length)
   - Check for SQL injection risks (none - using Dapr, not SQL)
   - Verify API key storage and transmission
   - Review CORS configuration (if applicable)

### Suggested Diagrams to Review

1. **Aspire Resource Topology Diagram**: Visual representation of all resources and their dependencies
2. **Service Discovery Flow**: How services find each other using names
3. **Sequence Diagram**: Complete flow from image upload to question display (including Aspire orchestration)
4. **Deployment Diagram**: How services are deployed in Aspire (containers, ports, networking)
5. **State Diagram**: TextState lifecycle (created → questions added → deleted)
6. **Error Flow Diagram**: How errors propagate from services to UI

### Tests to Review

- Check if unit tests exist for services (likely none - exploratory codebase)
- Consider adding integration tests for:
  - OCR service with mock Azure client
  - Question generation with mock Semantic Kernel
  - State service with in-memory Dapr client
  - SignalR hub event delivery
- **Aspire Testing**: Explore Aspire's testing capabilities for AppHost

---

## 9. Additional Resources

### Aspire Documentation

> **See [ASKVANTAGE_ASPIRE_PATTERNS.md](./ASKVANTAGE_ASPIRE_PATTERNS.md)** for comprehensive Aspire documentation links and resources.

### Other External Documentation

- **Dapr State Management**: <https://docs.dapr.io/developing-applications/building-blocks/state-management/>
- **Semantic Kernel**: <https://learn.microsoft.com/en-us/semantic-kernel/>
- **Prompty Format**: <https://github.com/microsoft/semantic-kernel/blob/main/dotnet/src/SemanticKernel.Prompty/README.md>
- **Azure Computer Vision**: <https://learn.microsoft.com/en-us/azure/ai-services/computer-vision/>
- **SignalR**: <https://learn.microsoft.com/en-us/aspnet/core/signalr/introduction>

### Internal Documentation

- **Scripts**: `scripts/` directory contains various documentation files (script00.md through script99.md)
- **Publish Demo**: `scripts/publish_demo.md` may contain deployment instructions

### Related Patterns

- **Fire-and-Forget with SignalR**: Common pattern for long-running operations
- **Dapr State Store**: Distributed state management pattern
- **Semantic Kernel Prompty**: Prompt engineering pattern for LLM integration

> **Aspire Patterns**: See [ASKVANTAGE_ASPIRE_PATTERNS.md](./ASKVANTAGE_ASPIRE_PATTERNS.md) for Aspire-specific patterns including Service Discovery, Resource Model, Service Defaults, Eventing, and Quick Reference table.

---

## 10. Common Pitfalls to Avoid

### Thread Safety

- ⚠️ **Don't use request-scoped services in `Task.Run()`**: The code correctly uses `IServiceScopeFactory` to create new scopes
- ⚠️ **Race conditions in "allKeys" updates**: The `GetAllKeys()` → modify → `SetAllKeys()` pattern is not atomic. Consider using Dapr transactions or optimistic concurrency.

### Disposal

- ⚠️ **SignalR connections**: Ensure `ImageApiHubClient` is properly disposed in `Home.razor.DisposeAsync()` (currently implemented)
- ⚠️ **Service scopes**: Service scopes created in `Task.Run()` are properly disposed via `using var scope`

### Configuration Gotchas

- ⚠️ **Dapr message size limits**: Configured to 10MB in `Program.cs` (gRPC channel options). Ensure this matches your image size limits.
- ⚠️ **JSON serialization quirks**: The code has workarounds for .NET 10 serialization issues with `ChatTokenUsage.Patch` property. Be careful when updating Semantic Kernel versions.
- ⚠️ **Prompty file loading**: Prompty files must be copied to output directory (configured in `.csproj`). If files are missing, functions won't load.

### Error Handling Pitfalls

- ⚠️ **Silent failures in `Task.Run()`**: Exceptions in `Task.Run()` are caught and sent via SignalR, but if SignalR fails, the error is lost. Consider logging to a persistent store.
- ⚠️ **JSON parsing failures**: The regex-based JSON extraction in `OpenAIQuestionGeneratorService` can fail silently if the LLM returns malformed JSON. Consider using a more robust parser or structured output.

### Performance

- ⚠️ **Large images in memory**: Images up to 10MB are loaded entirely into memory. For production, consider streaming or chunking.
- ⚠️ **No request cancellation**: `CancellationTokenSource` is created in `Task.Run()` but the original request's cancellation token is not used. Long-running operations can't be cancelled by the client.

### State Management

- ⚠️ **Key normalization**: `CreateKey()` lowercases titles, which could cause issues with case-sensitive comparisons elsewhere.
- ⚠️ **Question deduplication**: Questions are deduplicated by case-insensitive string comparison. This might not catch semantically identical questions with different wording.

### SignalR

- ⚠️ **Hub context lifetime**: `IHubContext` is resolved per request in controllers, but in `Task.Run()` it's resolved from a new scope. Ensure the hub is registered as singleton or scoped appropriately.
- ⚠️ **Connection state**: The frontend doesn't check connection state before sending events. If the connection is down, events are lost.

---

## Current vs Future State

### Current State Summary

- **Architecture**: .NET Aspire microservices with Blazor frontend (Server + WebAssembly)
- **Communication**: HTTP REST APIs + SignalR for real-time updates
- **State**: Dapr state store (Redis-backed) with custom key management
- **AI Integration**: Semantic Kernel with Prompty templates, supports Azure OpenAI or local Ollama
- **OCR**: Azure Computer Vision API
- **Deployment**: Aspire orchestrates containers/services

### What Will Change (Assumptions)

*Note: This section should be updated based on the specific changes you plan to make.*

**If adding new features:**

- New API endpoints will follow the existing pattern (controllers → services → external APIs)
- SignalR events will need to be added for new real-time updates
- State models may need to be extended

**If improving existing features:**

- Consider replacing `Task.Run()` with proper async/await patterns
- Add retry policies for external API calls
- Improve JSON parsing robustness
- Add caching layer for frequently accessed data
- Implement proper cancellation token propagation

---

## Assumptions Made

1. **No authentication/authorization**: The system appears to be a single-user or shared-state application
2. **Development environment**: The codebase is set up for local development with Aspire
3. **Azure services required**: Azure Computer Vision and Azure OpenAI (or local Ollama) are required dependencies
4. **Redis availability**: Redis must be running for the state store to work
5. **Dapr sidecar**: Dapr sidecar is automatically managed by Aspire
6. **No database**: All persistence is through Dapr state store (Redis), no traditional database

---

*Last Updated: [Current Date]*
*Explored By: AI Assistant*
*Codebase Version: Based on .NET 10.0, Aspire, Semantic Kernel 1.66.0*
