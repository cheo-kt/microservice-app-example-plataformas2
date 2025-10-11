FROM golang:1.13 AS builder

# Establecemos la ruta clásica del GOPATH
ENV GOPATH=/go
WORKDIR /go/src/auth-api

# Copiamos el código dentro del GOPATH/src
COPY . .

# Instalamos dep
RUN go get -u github.com/golang/dep/cmd/dep

# Descargamos dependencias
RUN dep ensure

# Compilamos el binario
RUN go build -o /go/bin/auth-api

# Imagen final mínima
FROM alpine:latest
WORKDIR /app
COPY --from=builder /go/bin/auth-api .
EXPOSE 8080
CMD ["./auth-api"]

