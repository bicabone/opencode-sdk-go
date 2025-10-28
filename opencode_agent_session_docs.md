# OpenCode Agent Session Documentation

This document provides a detailed analysis of the data models involved in an OpenCode agent session, based on the `bicabone/opencode-sdk-go` repository. It is intended to guide the development of a UI that can display this data and interact with the agent session, including interruption and resumption.

## Session Lifecycle

A session represents a single, stateful interaction with an OpenCode agent. It is the top-level container for all messages and their constituent parts. The session lifecycle is managed through a series of API calls and real-time events.

### Key Concepts

*   **Session:** The primary container for a conversation with an agent. It has a unique ID and holds all messages exchanged between the user and the agent.
*   **Message:** A single turn in the conversation. It can be from the `user` or the `assistant`. Each message has a unique ID and contains one or more `Part`s.
*   **Part:** A component of a message. A message is composed of different types of parts, such as text, tool calls, and files. Each part has a unique ID.
*   **Event:** A real-time notification about a change in the state of a session, message, or part. Clients subscribe to these events to update their UI in real-time.

### Data Models

The core data models are defined in [`session.go`](https://github.com/bicabone/opencode-sdk-go/blob/main/session.go). The following sections detail the most important structs.

#### Session

The `Session` struct represents the state of a conversation.

| Field | Type | Description |
| :--- | :--- | :--- |
| `ID` | `string` | The unique identifier for the session. |
| `Directory` | `string` | The working directory of the session. |
| `ProjectID` | `string` | The ID of the project this session belongs to. |
| `Time` | `SessionTime` | Timestamps for session creation and updates. |
| `Title` | `string` | A title for the session. |
| `Version` | `string` | The version of the OpenCode agent. |
| `ParentID` | `string` | The ID of the parent session, if this is a child session. |
| `Revert` | `SessionRevert` | Information about a reverted message. |
| `Share` | `SessionShare` | Information about a shared session. |

**Source:** [`session.go#L1286-L1297`](https://github.com/bicabone/opencode-sdk-go/blob/main/session.go#L1286-L1297)

#### Message

The `Message` struct represents a single message in the conversation. It can be a `UserMessage` or an `AssistantMessage`.

| Field | Type | Description |
| :--- | :--- | :--- |
| `ID` | `string` | The unique identifier for the message. |
| `Role` | `MessageRole` | The role of the message sender (`user` or `assistant`). |
| `SessionID` | `string` | The ID of the session this message belongs to. |
| `Time` | `interface{}` | Timestamps for the message. Can be `UserMessageTime` or `AssistantMessageTime`. |
| `Cost` | `float64` | The cost of the message. |
| `Error` | `interface{}` | Error information, if any. Can be `AssistantMessageError`. |
| `Finish` | `string` | The reason the message finished. |
| `Mode` | `string` | The mode of the agent when this message was generated. |
| `ModelID` | `string` | The ID of the model used to generate this message. |
| `Path` | `interface{}` | The working directory path. Can be `AssistantMessagePath`. |
| `ProviderID` | `string` | The ID of the provider that supplied the model. |
| `Summary` | `bool` | Whether this is a summary message. |
| `System` | `interface{}` | System messages. Can be `[]string`. |
| `Tokens` | `interface{}` | Token usage information. Can be `AssistantMessageTokens`. |

**Source:** [`session.go#L914-L937`](https://github.com/bicabone/opencode-sdk-go/blob/main/session.go#L914-L937)

#### Part

The `Part` struct is a generic container for different types of message content. The `Type` field determines the specific type of the part.

| Field | Type | Description |
| :--- | :--- | :--- |
| `ID` | `string` | The unique identifier for the part. |
| `MessageID` | `string` | The ID of the message this part belongs to. |
| `SessionID` | `string` | The ID of the session this part belongs to. |
| `Type` | `PartType` | The type of the part. See the table below for possible values. |
| `...` | `...` | Other fields depend on the `Type` of the part. |

**Source:** [`session.go#L1015-L1045`](https://github.com/bicabone/opencode-sdk-go/blob/main/session.go#L1015-L1045)

#### Part Types

The `PartType` enum determines the content of a `Part`. The following table lists the possible values and their corresponding structs.

| Part Type | Struct | Description |
| :--- | :--- | :--- |
| `text` | `TextPart` | A simple text message. |
| `reasoning` | `ReasoningPart` | The agent's reasoning process. |
| `file` | `FilePart` | A file sent or received in the message. |
| `tool` | `ToolPart` | A tool call made by the agent. |
| `step-start` | `StepStartPart` | Marks the beginning of a step in the agent's process. |
| `step-finish` | `StepFinishPart` | Marks the end of a step. |
| `snapshot` | `SnapshotPart` | A snapshot of the session state. |
| `patch` | `PartPatchPart` | A patch to a file. |
| `agent` | `AgentPart` | Information about the agent. |

**Source:** [`session.go#L1193-L1205`](https://github.com/bicabone/opencode-sdk-go/blob/main/session.go#L1193-L1205)

### Tool State Transitions

The `ToolPart` is particularly important for understanding the agent's actions. It has a `State` field that describes the status of the tool call. The `ToolPartState` has a `Status` field that can have one of the following values:

*   `pending`: The tool call has been initiated but has not yet started running.
*   `running`: The tool call is currently in progress.
*   `completed`: The tool call has completed successfully.
*   `error`: The tool call failed with an error.

These state transitions are communicated through `message.part.updated` events. By monitoring these events, a UI can display the real-time status of each tool call.

**Source:** [`session.go#L1962-L1969`](https://github.com/bicabone/opencode-sdk-go/blob/main/session.go#L1962-L1969)

## Interruption and Resumption

The OpenCode agent session can be interrupted and resumed. This is crucial for building interactive UIs that allow users to intervene in the agent's workflow.

### Abort

The `Abort` method on the `SessionService` is used to interrupt a running session. This will stop the agent from processing any further.

```go
func (r *SessionService) Abort(ctx context.Context, id string, body SessionAbortParams, opts ...option.RequestOption) (res *bool, err error)
```

**Source:** [`session.go#L85`](https://github.com/bicabone/opencode-sdk-go/blob/main/session.go#L85)

### Revert and Unrevert

The `Revert` method allows you to undo a message and revert the session to a previous state. This is useful for correcting mistakes or exploring different paths in the conversation.

```go
func (r *SessionService) Revert(ctx context.Context, id string, params SessionRevertParams, opts ...option.RequestOption) (res *Session, err error)
```

**Source:** [`session.go#L185`](https://github.com/bicabone/opencode-sdk-go/blob/main/session.go#L185)

The `Unrevert` method restores all reverted messages, effectively redoing the reverted actions.

```go
func (r *SessionService) Unrevert(ctx context.Context, id string, body SessionUnrevertParams, opts ...option.RequestOption) (res *Session, err error)
```

**Source:** [`session.go#L233`](https://github.com/bicabone/opencode-sdk-go/blob/main/session.go#L233)

### Resume

There is no explicit `Resume` method. After an `Abort`, the session can be resumed by sending a new prompt. After a `Revert`, the session is already in a new state, and the user can continue the conversation from there.

## Event Streaming

Real-time updates to the session state are communicated through a server-sent event (SSE) stream. The `EventService` provides a `ListStreaming` method to subscribe to these events.

```go
func (r *EventService) ListStreaming(ctx context.Context, query EventListParams, opts ...option.RequestOption) (stream *ssestream.Stream[EventListResponse])
```

**Source:** [`event.go#L42`](https://github.com/bicabone/opencode-sdk-go/blob/main/event.go#L42)

The `EventListResponse` is a union type that can represent many different types of events. The most important events for building a UI are:

*   `message.updated`: A message has been updated. The `info` field contains the updated `Message` object.
*   `message.removed`: A message has been removed.
*   `message.part.updated`: A part of a message has been updated. The `part` field contains the updated `Part` object. This is how tool state transitions are communicated.
*   `message.part.removed`: A part of a message has been removed.
*   `session.idle`: The session is now idle and waiting for user input.
*   `session.updated`: The session has been updated. The `info` field contains the updated `Session` object.

By handling these events, a UI can stay in sync with the state of the agent session and provide a rich, real-time user experience.

## Conclusion

This document has provided a comprehensive overview of the data models and mechanics of an OpenCode agent session. By understanding these concepts, developers can build powerful and interactive UIs that leverage the full capabilities of the OpenCode platform. For further details, please refer to the source code in the `bicabone/opencode-sdk-go` repository.
