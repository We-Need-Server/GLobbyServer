# /internal

This directory contains all the private application code for the GLobbyServer.

The logic within this directory is shared between the web server (`cmd/was`) and the gRPC server (`cmd/rpc`). No other project is expected to import code from here.

Subdirectories include:
-   `auth`: Core authentication logic (JWT, hashing).
-   `handler`: HTTP and gRPC request handlers.
-   `repository`: Database interaction logic.
-   `server`: Server setup and lifecycle management for Gin and gRPC.
