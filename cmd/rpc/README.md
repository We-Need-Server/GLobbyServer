# /cmd/rpc

This directory contains the entry point (`main.go`) for the gRPC server.

This application is responsible for:
-   Running the gRPC server for internal microservice communication.
-   Exposing the `AuthService` for token validation and renewal.
