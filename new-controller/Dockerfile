# Build stage
FROM --platform=$BUILDPLATFORM golang:1.24.3-alpine AS builder

# Arguments for cross-compilation
ARG TARGETOS
ARG TARGETARCH

WORKDIR /workspace

# Copy go mod files
COPY go.mod go.sum ./
RUN go mod download

# Copy the source code
COPY main.go main.go
COPY api/ api/
COPY controllers/ controllers/

# Build the manager binary for the target platform
RUN CGO_ENABLED=0 GOOS=${TARGETOS} GOARCH=${TARGETARCH} go build -a -o manager main.go

# Final stage - minimal container
FROM gcr.io/distroless/static:nonroot

WORKDIR /

# Copy the manager binary from builder
COPY --from=builder /workspace/manager .

USER 65532:65532

ENTRYPOINT ["/manager"]
