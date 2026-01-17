# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Pion WebRTC is a pure Go implementation of the WebRTC API (v4.x). It provides a complete WebRTC stack without CGo dependencies, supporting Windows, macOS, Linux, FreeBSD, iOS, Android, and WebAssembly.

## Development Commands

### Testing
```bash
# Run all tests
go test ./...

# Run a specific test
go test -run TestFunctionName ./...

# Run tests with race detection
go test -race ./...

# Run tests for a specific package
go test ./pkg/media/...
```

### Building
```bash
# Build the library
go build ./...

# Build an example
go build ./examples/play-from-disk/

# Build for WebAssembly
GOOS=js GOARCH=wasm go build -o demo.wasm ./examples/data-channels/jsfiddle/
```

### Running Examples
```bash
# Start the examples server (serves browser-based examples)
cd examples && go run examples.go

# With custom address
go run examples.go --address localhost:8080
```

### Linting
Linting uses golangci-lint via the shared Pion CI configuration.

## Architecture

### Core Components

**API Layer** (`api.go`): Factory for creating PeerConnections with customizable settings.
- `NewAPI()`: Creates API with MediaEngine, SettingEngine, and Interceptors
- `WithMediaEngine()`: Configure codec support
- `WithSettingEngine()`: Configure Pion-specific extensions
- `WithInterceptorRegistry()`: Configure RTP/RTCP interceptors

**PeerConnection** (`peerconnection.go`): Main WebRTC connection object implementing the W3C spec.
- Manages signaling state machine
- Coordinates ICE, DTLS, and SCTP transports
- Handles RTP transceivers and data channels

**Transport Stack** (layered architecture):
- `ICETransport` (`icetransport.go`): ICE connectivity (uses pion/ice)
- `DTLSTransport` (`dtlstransport.go`): DTLS encryption (uses pion/dtls)
- `SCTPTransport` (`sctptransport.go`): SCTP for data channels (uses pion/sctp)

**Media Handling**:
- `MediaEngine` (`mediaengine.go`): Codec registration and negotiation
- `RTPTransceiver` (`rtptransceiver.go`): Bidirectional RTP stream (sender + receiver)
- `TrackLocal`/`TrackRemote` (`track_local.go`, `track_remote.go`): Media track interfaces
- Interceptors (`interceptor.go`): RTP/RTCP processing pipeline (NACK, TWCC, reports)

**Data Channels** (`datachannel.go`): Bidirectional arbitrary data transfer over SCTP.

### Package Structure

- Root package (`webrtc`): Main WebRTC API
- `internal/fmtp`: Format parameter parsing (H264, VP9, AV1)
- `internal/mux`: Protocol multiplexing on single port
- `pkg/media/`: Media file readers/writers (IVF, OGG, H264, H265, RTP dump)
- `pkg/rtcerr`: WebRTC-specific error types

### Platform-Specific Code

Files with `_js.go` suffix contain WebAssembly implementations that wrap browser WebRTC APIs. Files with `//go:build !js` are native Go implementations.

## Key Pion Dependencies

- `pion/ice`: ICE agent implementation
- `pion/dtls`: DTLS 1.2 implementation
- `pion/srtp`: SRTP encryption
- `pion/sctp`: SCTP protocol
- `pion/sdp`: SDP parsing
- `pion/rtp`, `pion/rtcp`: RTP/RTCP packet handling
- `pion/interceptor`: RTP/RTCP processing pipeline
