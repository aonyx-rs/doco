# Build stage
FROM rust:1.84.0-slim AS builder

# Install system-level dependencies
RUN apt-get update && apt-get install -y \
    npm \
    wget \
    # Clean up
    && apt-get autoremove -y \
    && apt-get clean -y \
    && rm -rf /var/lib/apt/lists/*

# Install dart-sass
RUN npm install -g sass

# Add the WASM target
RUN rustup target add wasm32-unknown-unknown

# Install cargo-binstall
RUN cargo install cargo-binstall

# Install cargo-run-bin
RUN cargo binstall cargo-run-bin -y

WORKDIR /app

# Create an empty Rust crate
RUN cargo new --lib --vcs none crates/doco \
    && cargo new --lib --vcs none crates/doco-derive \
    && cargo new --lib --vcs none --name leptos-ssr examples/leptos

# Copy the Cargo.toml and Cargo.lock
COPY Cargo.toml Cargo.lock ./

# Cache the workspace
RUN cargo build --release

# Copy the example's Cargo.toml and Cargo.lock
COPY examples/leptos/Cargo.toml examples/leptos/Cargo.lock examples/leptos/

# Change into example directory
WORKDIR examples/leptos

# Install cargo-leptos
RUN cargo bin --install && cargo bin --sync-aliases

# Copy the source code. To optimize the Docker build, we are only selectively
# copying files that are required by the example. The order of the following
# statements is determined by their likelihood of changing.
COPY examples/leptos .
COPY crates ../../crates
RUN cargo leptos build --release

# Runtime stage
FROM debian:bookworm-slim AS runtime

WORKDIR /app

# Copy server binary
COPY --from=builder /app/examples/leptos/target/release/leptos-ssr server

# Copy WASM module, CSS stylesheets, and other static files
COPY --from=builder /app/examples/leptos/target/leptos site

# Configure a health check for the server
HEALTHCHECK CMD curl --fail http://localhost:8080 || exit 1

# Configure the Leptos app
ENV RUST_LOG="info"
ENV LEPTOS_SITE_ADDR="0.0.0.0:8080"
ENV LEPTOS_SITE_ROOT="site"
EXPOSE 8080

# Start the server
CMD ["/app/server"]
