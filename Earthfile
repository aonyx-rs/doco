VERSION 0.8

IMPORT github.com/earthly/lib/rust AS rust

FROM rust:1.85.0-slim
WORKDIR /doco

all:
    BUILD +json-format
    BUILD +markdown-format
    BUILD +markdown-lint
    BUILD +rust-deps-latest
    BUILD +rust-deps-minimal
    BUILD +rust-doc
    BUILD +rust-features
    BUILD +rust-format
    BUILD +rust-lint
    BUILD +rust-test
    BUILD +yaml-format
    BUILD +yaml-lint

COPY_SOURCES:
    FUNCTION

    # Copy the source code into the container
    COPY . .

COPY_RUST_SOURCES:
    FUNCTION

    # Copy the source code in a cache-friendly way
    COPY --keep-ts Cargo.toml Cargo.lock ./
    COPY --keep-ts --dir crates ./

node-container:
    FROM node:alpine
    WORKDIR /doco

    # Install prettier
    RUN npm install -g prettier markdownlint-cli

    # Copy the source code into the container
    DO +COPY_SOURCES

rust-container:
    # Install clippy and rustfmt
    RUN rustup component add clippy rustfmt

    # Install system-level dependencies
    RUN apt update && apt upgrade -y && apt install -y curl libssl-dev pkg-config

    # Initialize Rust
    DO rust+INIT --keep_fingerprints=true

rust-tarpaulin-container:
    FROM +rust-container

    # Install system-level dependencies
    RUN apt update && apt upgrade -y && apt install -y curl libssl-dev pkg-config

    # Install cargo-tarpaulin
    DO rust+CARGO --args="install cargo-tarpaulin"

    # Cache the container
    SAVE IMAGE --cache-hint

example-axum-container:
    FROM +rust-sources

    # Copy the example
    COPY --keep-ts --dir examples/axum-postgres examples/axum-postgres

    # Change the working directory
    WORKDIR examples/axum-postgres

example-axum-docker:
    FROM DOCKERFILE -f examples/axum-postgres/Dockerfile .

    # Cache the container
    SAVE IMAGE --cache-hint

example-axum-format:
    FROM +example-axum-container

    # Check the code formatting
    DO rust+CARGO --args="fmt --all --check"

example-axum-lint:
    FROM +example-axum-container

    # Check the code for linting errors
    DO rust+CARGO --args="clippy --all-targets --all-features -- -D warnings"

example-axum-test:
    FROM +example-axum-container

    # Run the tests
    WITH DOCKER --load doco:axum-postgres=+example-axum-docker
        RUN cargo test --all-features --all-targets --locked
    END

example-leptos-container:
    FROM +rust-sources

    # Copy the example
    COPY --keep-ts --dir examples/leptos examples/leptos

    # Change the working directory
    WORKDIR examples/leptos

example-leptos-docker:
    FROM DOCKERFILE -f examples/leptos/Dockerfile .

    # Cache the container
    SAVE IMAGE --cache-hint

example-leptos-format:
    FROM +example-leptos-container

    # Check the code formatting
    DO rust+CARGO --args="fmt --all --check"

example-leptos-lint:
    FROM +example-leptos-container

    # Check the code for linting errors
    DO rust+CARGO --args="clippy --all-targets --all-features -- -D warnings"

example-leptos-test:
    FROM +example-leptos-container

    # Run the tests
    WITH DOCKER --load doco:leptos=+example-leptos-docker
        RUN cargo test --all-features --all-targets --locked
    END

json-format:
    FROM +node-container

    # Check the JSON formatting
    RUN prettier --check **/*.{json,json5}

markdown-format:
    FROM +node-container

    # Check the formatting of Markdown files
    RUN prettier --check **/*.md

markdown-lint:
    FROM +node-container

    # Check the Markdown files for linting errors
    RUN markdownlint **/*.md

rust-sources:
    FROM +rust-container

    # Copy the source code in a cache-friendly way
    DO +COPY_RUST_SOURCES

rust-build:
    FROM +rust-sources

    # Build the project
    DO rust+CARGO --args="build --all-features --locked"

rust-deps-latest:
    FROM +rust-sources

    # Switch to beta toolchain
    RUN rustup default beta

    # Update the dependencies to the latest versions
    DO rust+CARGO --args="update"

    # Compile code to ensure the latest versions are compatible
    RUN RUSTFLAGS="-D deprecated" cargo check --all-features --all-targets --locked

rust-deps-minimal:
    FROM +rust-sources

    # Switch to nightly toolchain
    RUN rustup default nightly

    # Set minimal versions for dependencies
    DO rust+CARGO --args="update -Z direct-minimal-versions"

    # Compile code to ensure the minimal versions are compatible
    DO rust+CARGO --args="check --all-features --all-targets --locked"

rust-doc:
    FROM +rust-sources

    # Generate the documentation
    RUN cargo doc --all-features --no-deps

    # Save the documentation to the local filesystem
    SAVE ARTIFACT target/doc AS LOCAL target/doc

rust-features:
    FROM +rust-build

    # Install cargo-hack
    DO rust+CARGO --args="install cargo-hack"

    # Test combinations of features
    DO rust+CARGO --args="hack --feature-powerset check --lib --tests"

rust-format:
    FROM +rust-sources

    # Check the code formatting
    DO rust+CARGO --args="fmt --all --check"

rust-lint:
    FROM +rust-build

    # Check the code for linting errors
    DO rust+CARGO --args="clippy --all-targets --all-features -- -D warnings"

rust-test:
    # Optionally save the report to the local filesystem
    ARG SAVE_REPORT=""

    FROM +rust-tarpaulin-container

    # Copy the source code in a cache-friendly way
    DO +COPY_RUST_SOURCES

    # Run the tests and measure the code coverage
    WITH DOCKER --pull selenium/standalone-firefox:latest
        RUN cargo tarpaulin \
            --all-features \
            --engine llvm \
            --exclude doco-derive \
            --out Xml \
            --skip-clean \
            --timeout 120 \
            --verbose \
            --workspace
    END

    # Save the coverage report
    IF [ "$SAVE_REPORT" != "" ]
        SAVE ARTIFACT cobertura.xml AS LOCAL cobertura.xml
    END

yaml-format:
    FROM +node-container

    # Check the YAML formatting
    RUN prettier --check **/*.{yml,yaml}

yaml-lint:
    FROM pipelinecomponents/yamllint:latest
    WORKDIR /doco

    # Copy the source code into the container
    DO +COPY_SOURCES

    # Check the YAML files for linting errors
    RUN yamllint .
