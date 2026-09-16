# Build stage
FROM golang:alpine AS builder

WORKDIR /src

COPY . .

WORKDIR /src/main

RUN go get -v ./...
RUN CGO_ENABLED=0 GOOS=linux go build -o /app ./...

# Runtime stage
FROM alpine:latest

RUN apk --no-cache add ca-certificates

WORKDIR /app

COPY --from=builder /app ./app

ENTRYPOINT ["./app"]

EXPOSE 8080

LABEL Name="auth" Version="2.0.0"