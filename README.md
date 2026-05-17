# Income Tax Calculator

This project is a work-in-progress tax calculator for estimating US Federal and State Taxes.

## Current Scope

At the moment, the application supports:

* Single filers only
* State Taxes for Massachusetts

Support for additional filing statuses and states is planned for future releases.

## Running the Application

### Build the Docker image

```bash
docker build -t tax-app .
```

### Run the container

```bash
docker run -p 3000:3000 tax-filing-app
```

The application will then be available at:

```text
http://localhost:3000
```

## Status

This project is still under active development and APIs, workflows, and features may change frequently.
