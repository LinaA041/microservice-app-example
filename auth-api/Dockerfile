
FROM golang:1.18 as builder

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .

RUN CGO_ENABLED=0 GOOS=linux go build -o auth-api .


FROM alpine:3.18

RUN apk --no-cache add ca-certificates libc6-compat

WORKDIR /app
COPY --from=builder /app/auth-api .

EXPOSE 8000

CMD ["./auth-api"]
