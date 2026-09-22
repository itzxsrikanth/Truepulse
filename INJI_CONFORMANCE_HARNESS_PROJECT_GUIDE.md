# Inji Conformance Testing Harness — Complete Project Guide

**Project:** Automated Conformance Testing for Inji Certify and Inji Verify against OpenID Suite  
**Complexity:** Medium  
**Duration:** 13 weeks  
**Status:** Planning Phase

---

## Table of Contents
1. [Project Overview](#project-overview)
2. [Problem Statement](#problem-statement)
3. [Solution Architecture](#solution-architecture)
4. [Directory Structure](#directory-structure)
5. [Component Architecture](#component-architecture)
6. [Technology Stack](#technology-stack)
7. [Implementation Roadmap](#implementation-roadmap)
8. [Core Implementation Details](#core-implementation-details)
9. [Configuration Management](#configuration-management)
10. [Shell Scripts & Execution](#shell-scripts--execution)
11. [GitHub Actions Workflow](#github-actions-workflow)
12. [Data Flow & Models](#data-flow--models)
13. [Testing Strategy](#testing-strategy)
14. [Documentation Plan](#documentation-plan)
15. [Success Criteria & Metrics](#success-criteria--metrics)
16. [Deployment & Release](#deployment--release)
17. [Troubleshooting Guide](#troubleshooting-guide)

---

## Project Overview

### What We're Building
An **automated orchestration harness** that:
- Spins up Inji Certify, Inji Verify, and OpenID Foundation Conformance Suite in Docker
- Programmatically drives OpenID REST API (no manual UI clicking)
- Runs issuer & verifier test plans against each component
- Feeds results into existing TestNG/Extent-based api-testrig framework
- Produces consolidated, benchmark-gated reports
- Integrates with CI/CD for automatic testing on every release

### Why It Matters
**Current State:** Manual, separate, per-component testing  
**Problem:** Doesn't scale. Not integrated. Slow. Error-prone.  
**Solution:** One-command execution. Automated. Integrated. Production-ready.

---

## Problem Statement

### The Current State
- Inji Certify and Inji Verify are validated **independently** against OpenID Foundation conformance suite
- Tests are run **manually** through conformance suite's web UI
- **Separate setups**, **separate runs**, **separate result screens**
- Nothing ties them into the existing test pipeline
- No automatic execution on release

### The Problem
```
❌ It doesn't scale
❌ It's not integrated  
❌ It's not in CI/CD
❌ It's slow and error-prone
```

### The Solution
**Build an automated harness that:**
- Runs conformance for each module → feeds result into that module's existing api-testrig
- Supports both **per-module** (independent gates) and **combined** (cross-module badge) modes
- Spins up containers, executes REST API calls programmatically, generates reports
- Runs on every release via CI/CD, failing the build if either component regresses

---

## Solution Architecture

### High-Level Flow
```
User / CI/CD Trigger
    ↓
run-conformance.sh (--component certify|verify|combined)
    ↓
OrchestrationManager
├─→ ComponentLifecycleManager (docker-compose up)
├─→ ConformanceRunner
│   ├─ TestPlanBuilder (load endpoints, config)
│   ├─ RestApiClient (POST /api/plan, GET /api/status)
│   ├─ PythonProcessBridge (invoke run-test-plan.py)
│   └─ ResultMapper (JSON → TestNG model)
└─→ ExtentReportGenerator
    ├─ TestNG integration
    ├─ Extent visualization
    └─ BenchmarkGateway (pass/fail)
    ↓
minio/S3 Bucket + TestNG Report
```

### Architectural Layers

```
┌─────────────────────────────────────────────────────────────┐
│                    CI/CD Integration Layer                  │
│              (GitHub Actions Workflows)                     │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                    Orchestration Layer                      │
│  (Lifecycle Mgmt, Container Coordination, Run Modes)        │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                  Automation/Execution Layer                 │
│  (Conformance Runner, Python Bridge, REST Driving)         │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                  Configuration Layer                        │
│  (Test Plan Builders, Config Validators, Env Mgmt)         │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                  Integration/Reporting Layer                │
│  (TestNG Adapter, Extent Reports, Benchmark Gating)        │
└─────────────────────────────────────────────────────────────┘
```

### Two Run Modes

**1. Per-Module (Default)**
- Inji Certify's conformance runs inside Inji Certify's api-testrig
- Inji Verify's conformance runs inside Inji Verify's api-testrig
- Each module's gate is independent
- Matches existing api-testrig module-specific structure

**2. Combined (Optional)**
- Single orchestrated run that executes both issuer & verifier plans together
- One consolidated cross-module conformance report
- Badge for "Certify ↔ Verify OpenID 1.0" status

---

## Directory Structure

```
inji-conformance-harness/
│
├── .github/
│   └── workflows/
│       └── full-stack-conformance.yml          [CI/CD pipeline]
│
├── docker/
│   ├── compose/
│   │   ├── docker-compose.yml                  [Multi-service orchestration]
│   │   ├── docker-compose.dev.yml              [Development variant]
│   │   └── .env.template                       [Environment variables]
│   └── scripts/
│       ├── init-conformance-suite.sh
│       └── cleanup.sh
│
├── src/
│   ├── main/
│   │   │
│   │   ├── java/
│   │   │   └── io/mosip/inji/conformance/
│   │   │       │
│   │   │       ├── OpenIDConformanceTest.java  [MAIN TestNG entry point]
│   │   │       │
│   │   │       ├── runner/
│   │   │       │   ├── ConformanceRunner.java  [Core executor]
│   │   │       │   ├── ConformancePlanExecutor.java  [Plan lifecycle]
│   │   │       │   └── ResultMapper.java  [REST response → TestNG]
│   │   │       │
│   │   │       ├── config/
│   │   │       │   ├── ConformanceConfig.java  [Global config]
│   │   │       │   ├── TestPlanConfig.java  [Test plan config]
│   │   │       │   ├── ConfigValidator.java  [Validation logic]
│   │   │       │   └── ComponentEndpointConfig.java
│   │   │       │
│   │   │       ├── integration/
│   │   │       │   ├── RestApiClient.java  [OpenID REST client]
│   │   │       │   ├── ConformanceSuiteClient.java  [Wrapper]
│   │   │       │   └── PythonProcessBridge.java  [Subprocess executor]
│   │   │       │
│   │   │       ├── model/
│   │   │       │   ├── TestResult.java  [Single test result]
│   │   │       │   ├── ConformanceResult.java  [Batch results]
│   │   │       │   ├── TestPlan.java  [Test plan DTO]
│   │   │       │   ├── ComponentType.java  [CERTIFY, VERIFY enum]
│   │   │       │   └── TestStatus.java  [PASS, FAIL, SKIP enum]
│   │   │       │
│   │   │       ├── reporting/
│   │   │       │   ├── ExtentReportGenerator.java  [Report creation]
│   │   │       │   ├── TestNGResultAdapter.java  [Map results]
│   │   │       │   └── BenchmarkGateway.java  [Gating logic]
│   │   │       │
│   │   │       ├── orchestration/
│   │   │       │   ├── OrchestrationManager.java  [Main orchestrator]
│   │   │       │   ├── ComponentLifecycleManager.java  [Docker mgmt]
│   │   │       │   └── CombinedRunOrchestrator.java  [Cross-module runs]
│   │   │       │
│   │   │       └── exception/
│   │   │           ├── ConformanceException.java  [Base exception]
│   │   │           ├── ConfigurationException.java
│   │   │           ├── ExecutionException.java
│   │   │           └── RestApiException.java
│   │   │
│   │   ├── python/
│   │   │   ├── conformance_runner.py  [Main runner script]
│   │   │   ├── rest_api_wrapper.py  [REST API abstraction]
│   │   │   ├── config_builder.py  [Config construction]
│   │   │   ├── result_parser.py  [JSON result parsing]
│   │   │   └── requirements.txt  [Python dependencies]
│   │   │
│   │   └── resources/
│   │       ├── application.properties  [App config]
│   │       ├── logback.xml  [Logging config]
│   │       ├── test-plan-templates/
│   │       │   ├── issuer-plan-1.0.json  [Certify template]
│   │       │   └── verifier-plan-1.0.json  [Verify template]
│   │       └── sql/  [If DB setup needed]
│   │
│   ├── test/
│   │   ├── java/
│   │   │   └── io/mosip/inji/conformance/
│   │   │       ├── runner/
│   │   │       │   ├── ConformanceRunnerTest.java
│   │   │       │   └── ResultMapperTest.java
│   │   │       ├── integration/
│   │   │       │   └── RestApiClientTest.java
│   │   │       ├── config/
│   │   │       │   └── ConfigValidatorTest.java
│   │   │       └── reporting/
│   │   │           └── BenchmarkGatewayTest.java
│   │   │
│   │   └── resources/
│   │       ├── test-application.properties
│   │       ├── mock-responses/  [Mock JSON responses]
│   │       │   ├── issuer-plan-response.json
│   │       │   └── verifier-plan-response.json
│   │       └── testng.xml  [Test configuration]
│   │
│   └── integration-test/
│       ├── java/
│       │   └── io/mosip/inji/conformance/
│       │       ├── E2EConformanceTest.java
│       │       └── FullStackIntegrationTest.java
│       └── resources/
│           ├── testng-integration.xml
│           └── docker-compose-integration.yml
│
├── scripts/
│   ├── run-conformance.sh  [Main entry point]
│   ├── run-conformance-certify.sh  [Certify only]
│   ├── run-conformance-verify.sh  [Verify only]
│   ├── run-conformance-combined.sh  [Both]
│   ├── cleanup-conformance.sh  [Cleanup]
│   ├── validate-config.py  [Config validation]
│   └── diff-results.py  [Compare results]
│
├── config/
│   ├── conformance-config.json  [Global configuration]
│   ├── certify-endpoints.json  [Certify endpoints]
│   ├── verify-endpoints.json  [Verify endpoints]
│   ├── benchmark-thresholds.json  [Pass/fail thresholds]
│   └── environment/
│       ├── dev.env  [Dev environment]
│       ├── staging.env  [Staging environment]
│       └── production.env  [Production environment]
│
├── docs/
│   ├── ARCHITECTURE.md  [System design]
│   ├── API_AUTOMATION.md  [REST API details]
│   ├── SETUP_GUIDE.md  [Installation & setup]
│   ├── TROUBLESHOOTING.md  [Common issues]
│   ├── CERTIFICATION_WORKFLOW.md  [OpenID certification]
│   ├── DEVELOPMENT.md  [Development guide]
│   ├── diagrams/
│   │   ├── system-architecture.md
│   │   ├── execution-flow.md
│   │   └── data-model.md
│   └── examples/
│       ├── single-component-run.md
│       ├── combined-run-example.md
│       └── ci-cd-integration.md
│
├── pom.xml  [Maven configuration]
├── Dockerfile  [Container definition]
├── .gitignore
├── README.md  [Quick start]
└── LICENSE
```

---

## Component Architecture

### Module Responsibilities

| Module | Responsibility | Key Classes | Dependencies |
|--------|---|---|---|
| **Runner** | Drives conformance execution, orchestrates steps | `ConformanceRunner`, `ConformancePlanExecutor`, `ResultMapper` | RestApiClient, ConfigValidator |
| **Config** | Parses, validates, manages test plan & environment | `TestPlanConfig`, `ConfigValidator`, `ConformanceConfig` | Jackson, File I/O |
| **Integration** | Bridges to REST API & Python subprocess | `ConformanceSuiteClient`, `RestApiClient`, `PythonProcessBridge` | OkHttp/RestTemplate, Process API |
| **Model** | Data structures for test results, plans, status | `TestResult`, `ConformanceResult`, `TestPlan`, `ComponentType` | Lombok |
| **Reporting** | Integrates TestNG + Extent, generates reports | `ExtentReportGenerator`, `TestNGResultAdapter`, `BenchmarkGateway` | TestNG, Extent Reports |
| **Orchestration** | Component lifecycle, Docker coordination, combined runs | `OrchestrationManager`, `ComponentLifecycleManager`, `CombinedRunOrchestrator` | Docker Java API, Testcontainers |
| **Exception** | Custom exception hierarchy | `ConformanceException`, `ConfigurationException`, `ExecutionException` | - |

### Class Diagram (Simplified)

```
OpenIDConformanceTest (TestNG)
    └── OrchestrationManager
        ├── ComponentLifecycleManager
        │   └── [Docker Containers]
        └── ConformanceRunner
            ├── ConfigValidator
            ├── ConformancePlanExecutor
            │   ├── RestApiClient
            │   ├── PythonProcessBridge
            │   └── ResultMapper
            ├── ExtentReportGenerator
            └── BenchmarkGateway
```

---

## Technology Stack

| Layer | Technology | Version | Purpose | Why |
|-------|-----------|---------|---------|-----|
| **Language** | Java | 11 | Backend automation | Matches mosip-functional-tests stack |
| **Build Tool** | Maven | 3.8+ | Dependency & build management | Standard for Java projects |
| **Testing** | TestNG | 7.x | Test execution framework | Native integration with api-testrig |
| **Reporting** | Extent Reports | 5.x | Visual HTML reports | Seamless TestNG integration |
| **REST Client** | OkHttp / RestTemplate | Latest | HTTP API calls | Lightweight, reliable |
| **JSON** | Jackson | 2.x | JSON parsing/serialization | Standard Jackson for Java |
| **Logging** | SLF4J + Logback | Latest | Logging framework | Industry standard |
| **Container** | Docker | 20.10+ | Containerization | Orchestrate Inji + OpenID Suite |
| **Orchestration** | Docker Compose | 1.29+ | Multi-container coordination | Simple, declarative |
| **Automation (Python)** | Python | 3.8+ | REST API driving | OpenID Suite provides run-test-plan.py |
| **Python Libs** | requests, json | Latest | HTTP & JSON parsing | For REST automation |
| **CI/CD** | GitHub Actions | - | Pipeline automation | Native GitHub integration |

### Maven Dependencies (pom.xml excerpt)

```xml
<dependencies>
    <!-- Core Testing -->
    <dependency>
        <groupId>org.testng</groupId>
        <artifactId>testng</artifactId>
        <version>7.7.0</version>
    </dependency>
    
    <!-- Reporting -->
    <dependency>
        <groupId>com.aventstack</groupId>
        <artifactId>extentreports</artifactId>
        <version>5.0.9</version>
    </dependency>
    
    <!-- REST Client -->
    <dependency>
        <groupId>com.squareup.okhttp3</groupId>
        <artifactId>okhttp</artifactId>
        <version>4.9.3</version>
    </dependency>
    
    <!-- JSON -->
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
        <version>2.14.0</version>
    </dependency>
    
    <!-- Logging -->
    <dependency>
        <groupId>org.slf4j</groupId>
        <artifactId>slf4j-api</artifactId>
        <version>1.7.36</version>
    </dependency>
    <dependency>
        <groupId>ch.qos.logback</groupId>
        <artifactId>logback-classic</artifactId>
        <version>1.2.11</version>
    </dependency>
    
    <!-- Docker -->
    <dependency>
        <groupId>com.github.docker-java</groupId>
        <artifactId>docker-java</artifactId>
        <version>3.2.13</version>
    </dependency>
    
    <!-- Utilities -->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <version>1.18.24</version>
        <scope>provided</scope>
    </dependency>
</dependencies>
```

---

## Implementation Roadmap

### Phase 1: Foundation (Weeks 1–2) ⚙️
**Goal:** Setup project infrastructure & Docker orchestration

**Tasks:**
- [ ] Initialize Maven project with standard directory structure
- [ ] Configure pom.xml with core dependencies
- [ ] Create Docker Compose file (Inji Certify, Inji Verify, OpenID Suite)
- [ ] Implement `ConformanceConfig` class with JSON parsing
- [ ] Build JSON test plan templates for issuer/verifier (OpenID 1.0)
- [ ] Create `ConfigValidator` with validation logic
- [ ] Write unit tests for config loading/validation

**Deliverables:**
- Working Maven project
- Docker Compose runs all 3 services
- Config validation passing

**Success Metrics:**
- `docker-compose up` brings up all services
- Config JSON loads without errors
- Unit tests > 90% passing

---

### Phase 2: Automation Engine (Weeks 3–5) 🤖
**Goal:** Build REST API driving layer

**Tasks:**
- [ ] Implement `RestApiClient` (POST /api/plan, GET /api/status, GET /api/results)
- [ ] Build `PythonProcessBridge` to invoke `run-test-plan.py` as subprocess
- [ ] Implement `ConformancePlanExecutor` (create → run → poll → retrieve)
- [ ] Build `ResultMapper` (REST response JSON → TestNG `TestResult` model)
- [ ] Add retry logic & exponential backoff for API calls
- [ ] Add error handling for all failure scenarios
- [ ] Write integration tests with mock OpenID Suite responses

**Deliverables:**
- `ConformanceRunner` can execute issuer & verifier plans independently
- Results mapped to TestNG models
- Integration tests passing

**Success Metrics:**
- Can create & execute test plans via REST API
- Result mapping 100% accurate
- Retry logic works (tested with delays/failures)

---

### Phase 3: Framework Integration (Weeks 6–8) 🔗
**Goal:** Integrate with existing api-testrig

**Tasks:**
- [ ] Create `OpenIDConformanceTest` TestNG class
- [ ] Integrate into mosip-functional-tests project
- [ ] Implement `ExtentReportGenerator` to create HTML reports
- [ ] Build `TestNGResultAdapter` to map conformance results to TestNG lifecycle
- [ ] Implement `BenchmarkGateway` for threshold-based gating
- [ ] Add minio/S3 upload for reports
- [ ] Write per-module test suites (Certify test suite, Verify test suite)

**Deliverables:**
- TestNG class executes conformance & emits results
- Reports visible in Extent/TestNG viewer
- Benchmark gating works (pass/fail)

**Success Metrics:**
- Per-module tests integrated into api-testrig
- Reports generated & uploaded to S3
- Gating prevents build if thresholds not met

---

### Phase 4: Orchestration & Combined Runs (Weeks 9–11) 🎯
**Goal:** Full orchestration & multi-run support

**Tasks:**
- [ ] Implement `OrchestrationManager` for lifecycle management
- [ ] Build `ComponentLifecycleManager` (docker-compose up/down)
- [ ] Implement `CombinedRunOrchestrator` for cross-module runs
- [ ] Create shell script wrappers:
  - `run-conformance.sh --component certify`
  - `run-conformance.sh --component verify`
  - `run-conformance.sh --combined`
- [ ] Add result consolidation logic for combined mode
- [ ] Implement expected-failures handling (track known failures)
- [ ] Add result diff utility (`diff-results.py`)

**Deliverables:**
- Single-command execution for all run modes
- Consolidated reports for combined runs
- Shell scripts production-ready

**Success Metrics:**
- All 3 run modes execute successfully
- Combined report generated & consolidated
- Shell scripts tested locally & in CI

---

### Phase 5: CI/CD & Polish (Weeks 12–13) ✨
**Goal:** Production readiness & deployment

**Tasks:**
- [ ] Create GitHub Actions workflow (`.github/workflows/full-stack-conformance.yml`)
- [ ] Configure matrix for parallel Certify/Verify runs (optional)
- [ ] Write comprehensive documentation:
  - ARCHITECTURE.md
  - API_AUTOMATION.md
  - SETUP_GUIDE.md
  - TROUBLESHOOTING.md
  - DEVELOPMENT.md
- [ ] Add logging & observability throughout
- [ ] Final integration testing (end-to-end)
- [ ] Code review & refactoring
- [ ] Create artifact & publish (JAR)

**Deliverables:**
- GitHub Actions workflow active
- Full documentation in `/docs`
- Build artifact published
- Project ready for production deployment

**Success Metrics:**
- All tests passing in CI
- Documentation complete & reviewable
- Build artifact published & versioned
- Team trained on operation

---

## Core Implementation Details

### 1. ConformanceRunner (Main Executor)

```java
package io.mosip.inji.conformance.runner;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.mosip.inji.conformance.config.ConformanceConfig;
import io.mosip.inji.conformance.config.TestPlanConfig;
import io.mosip.inji.conformance.integration.RestApiClient;
import io.mosip.inji.conformance.integration.PythonProcessBridge;
import io.mosip.inji.conformance.model.ConformanceResult;
import io.mosip.inji.conformance.model.ComponentType;
import io.mosip.inji.conformance.model.TestResult;
import io.mosip.inji.conformance.reporting.BenchmarkGateway;
import lombok.extern.slf4j.Slf4j;

import java.util.List;
import java.util.Map;

@Slf4j
public class ConformanceRunner {
    private final ConformanceConfig config;
    private final RestApiClient apiClient;
    private final PythonProcessBridge pythonBridge;
    private final ResultMapper resultMapper;
    private final BenchmarkGateway benchmarkGateway;
    
    // Constructor with DI
    public ConformanceRunner(ConformanceConfig config,
                             RestApiClient apiClient,
                             PythonProcessBridge pythonBridge,
                             ResultMapper resultMapper,
                             BenchmarkGateway benchmarkGateway) {
        this.config = config;
        this.apiClient = apiClient;
        this.pythonBridge = pythonBridge;
        this.resultMapper = resultMapper;
        this.benchmarkGateway = benchmarkGateway;
    }
    
    /**
     * Execute issuer (Certify) test plan
     */
    public ConformanceResult runIssuerPlan(String certifyEndpoint) {
        log.info("Starting Issuer (Certify) conformance plan execution");
        
        try {
            // Step 1: Load & validate test plan config
            TestPlanConfig planConfig = loadIssuerTestPlan(certifyEndpoint);
            log.debug("Issuer test plan loaded: {}", planConfig);
            
            // Step 2: Create test plan via REST API
            String planId = apiClient.createTestPlan(planConfig);
            log.info("Test plan created with ID: {}", planId);
            
            // Step 3: Run test plan (via Python automation library)
            String runStatus = pythonBridge.runTestPlan(planId);
            log.info("Test plan execution started: {}", runStatus);
            
            // Step 4: Poll for completion
            String finalStatus = pollForCompletion(planId);
            log.info("Test plan execution completed: {}", finalStatus);
            
            // Step 5: Retrieve results
            Map<String, Object> rawResults = apiClient.getTestResults(planId);
            log.debug("Raw results retrieved: {} test cases", rawResults.size());
            
            // Step 6: Map to ConformanceResult
            ConformanceResult result = resultMapper.mapToConformanceResult(
                rawResults,
                ComponentType.CERTIFY,
                planId
            );
            
            // Step 7: Apply benchmark gating
            boolean passed = benchmarkGateway.validate(result, ComponentType.CERTIFY);
            result.setBenchmarkPassed(passed);
            
            log.info("Issuer conformance completed. Passed: {}", passed);
            return result;
            
        } catch (Exception e) {
            log.error("Issuer conformance execution failed", e);
            throw new ExecutionException("Issuer plan execution failed: " + e.getMessage(), e);
        }
    }
    
    /**
     * Execute verifier (Verify) test plan
     */
    public ConformanceResult runVerifierPlan(String verifyEndpoint) {
        log.info("Starting Verifier (Verify) conformance plan execution");
        
        try {
            TestPlanConfig planConfig = loadVerifierTestPlan(verifyEndpoint);
            log.debug("Verifier test plan loaded: {}", planConfig);
            
            String planId = apiClient.createTestPlan(planConfig);
            log.info("Test plan created with ID: {}", planId);
            
            String runStatus = pythonBridge.runTestPlan(planId);
            log.info("Test plan execution started: {}", runStatus);
            
            String finalStatus = pollForCompletion(planId);
            log.info("Test plan execution completed: {}", finalStatus);
            
            Map<String, Object> rawResults = apiClient.getTestResults(planId);
            log.debug("Raw results retrieved: {} test cases", rawResults.size());
            
            ConformanceResult result = resultMapper.mapToConformanceResult(
                rawResults,
                ComponentType.VERIFY,
                planId
            );
            
            boolean passed = benchmarkGateway.validate(result, ComponentType.VERIFY);
            result.setBenchmarkPassed(passed);
            
            log.info("Verifier conformance completed. Passed: {}", passed);
            return result;
            
        } catch (Exception e) {
            log.error("Verifier conformance execution failed", e);
            throw new ExecutionException("Verifier plan execution failed: " + e.getMessage(), e);
        }
    }
    
    /**
     * Poll for test plan completion with exponential backoff
     */
    private String pollForCompletion(String planId) {
        int maxRetries = config.getPollMaxRetries(); // e.g., 60
        long pollInterval = config.getPollIntervalMs(); // e.g., 2000ms
        int retries = 0;
        
        while (retries < maxRetries) {
            try {
                String status = apiClient.checkTestPlanStatus(planId);
                
                if (isComplete(status)) {
                    return status;
                }
                
                retries++;
                Thread.sleep(pollInterval);
                
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                throw new ExecutionException("Poll interrupted", e);
            }
        }
        
        throw new ExecutionException("Test plan did not complete within timeout");
    }
    
    private boolean isComplete(String status) {
        return status.equalsIgnoreCase("FINISHED") ||
               status.equalsIgnoreCase("COMPLETED") ||
               status.equalsIgnoreCase("FAILED");
    }
    
    private TestPlanConfig loadIssuerTestPlan(String endpoint) {
        ObjectMapper mapper = new ObjectMapper();
        TestPlanConfig template = config.getIssuerPlanTemplate();
        template.setEndpoint(endpoint);
        // Override with endpoint, populate aliases, etc.
        return template;
    }
    
    private TestPlanConfig loadVerifierTestPlan(String endpoint) {
        ObjectMapper mapper = new ObjectMapper();
        TestPlanConfig template = config.getVerifierPlanTemplate();
        template.setEndpoint(endpoint);
        return template;
    }
}
```

### 2. OpenIDConformanceTest (TestNG Entry Point)

```java
package io.mosip.inji.conformance;

import io.mosip.inji.conformance.config.ConformanceConfig;
import io.mosip.inji.conformance.model.ConformanceResult;
import io.mosip.inji.conformance.model.ComponentType;
import io.mosip.inji.conformance.orchestration.OrchestrationManager;
import io.mosip.inji.conformance.reporting.ExtentReportGenerator;
import io.mosip.inji.conformance.runner.ConformanceRunner;
import lombok.extern.slf4j.Slf4j;
import org.testng.ITestContext;
import org.testng.annotations.*;

/**
 * Main TestNG test class for OpenID conformance testing
 * Integrates with api-testrig framework
 */
@Slf4j
public class OpenIDConformanceTest {
    
    private ConformanceConfig config;
    private OrchestrationManager orchestrationManager;
    private ConformanceRunner conformanceRunner;
    private ExtentReportGenerator reportGenerator;
    private ConformanceResult issuerResult;
    private ConformanceResult verifierResult;
    
    @BeforeSuite
    public void initializeTestEnvironment(ITestContext context) {
        log.info("Initializing OpenID Conformance Test Suite");
        
        // Load configuration
        config = ConformanceConfig.loadFromFile("config/conformance-config.json");
        
        // Initialize orchestration manager
        orchestrationManager = new OrchestrationManager(config);
        
        // Spin up Docker containers
        orchestrationManager.spinUpContainers();
        log.info("All containers spun up successfully");
        
        // Wait for services to be ready
        orchestrationManager.waitForServicesReady(config.getStartupTimeoutSeconds());
        
        // Initialize report generator
        reportGenerator = new ExtentReportGenerator(
            "conformance-reports",
            context.getSuite().getName()
        );
    }
    
    @Test(
        groups = {"conformance", "issuer"},
        description = "Inji Certify OpenID 1.0 Issuer Conformance Test"
    )
    public void testIssuerConformance(ITestContext context) {
        log.info("Executing Inji Certify (Issuer) conformance test");
        
        try {
            // Initialize runner
            conformanceRunner = orchestrationManager.getConformanceRunner();
            
            // Run issuer plan
            issuerResult = conformanceRunner.runIssuerPlan(
                config.getCertifyEndpoint()
            );
            
            // Validate benchmark
            if (!issuerResult.isBenchmarkPassed()) {
                throw new AssertionError(
                    "Issuer conformance failed benchmark validation. " +
                    "Pass rate: " + issuerResult.getPassPercentage() + "%"
                );
            }
            
            log.info("Issuer conformance test passed");
            
        } catch (Exception e) {
            log.error("Issuer conformance test failed", e);
            throw new AssertionError("Issuer test execution failed: " + e.getMessage(), e);
        }
    }
    
    @Test(
        groups = {"conformance", "verifier"},
        description = "Inji Verify OpenID 1.0 Verifier Conformance Test"
    )
    public void testVerifierConformance(ITestContext context) {
        log.info("Executing Inji Verify (Verifier) conformance test");
        
        try {
            conformanceRunner = orchestrationManager.getConformanceRunner();
            
            verifierResult = conformanceRunner.runVerifierPlan(
                config.getVerifyEndpoint()
            );
            
            if (!verifierResult.isBenchmarkPassed()) {
                throw new AssertionError(
                    "Verifier conformance failed benchmark validation. " +
                    "Pass rate: " + verifierResult.getPassPercentage() + "%"
                );
            }
            
            log.info("Verifier conformance test passed");
            
        } catch (Exception e) {
            log.error("Verifier conformance test failed", e);
            throw new AssertionError("Verifier test execution failed: " + e.getMessage(), e);
        }
    }
    
    @Test(
        groups = {"conformance", "combined"},
        dependsOnMethods = {"testIssuerConformance", "testVerifierConformance"},
        description = "Combined Certify ↔ Verify OpenID 1.0 Conformance Report"
    )
    public void testCombinedConformance(ITestContext context) {
        log.info("Generating combined conformance report");
        
        try {
            if (issuerResult == null || verifierResult == null) {
                throw new IllegalStateException(
                    "Issuer or Verifier results missing. " +
                    "Ensure per-module tests executed first."
                );
            }
            
            // Generate consolidated report
            reportGenerator.generateCombinedReport(issuerResult, verifierResult);
            
            // Create consolidated badge
            boolean allPassed = issuerResult.isBenchmarkPassed() &&
                                verifierResult.isBenchmarkPassed();
            
            if (!allPassed) {
                throw new AssertionError(
                    "Combined conformance failed. " +
                    "Issuer: " + issuerResult.getPassPercentage() + "%, " +
                    "Verifier: " + verifierResult.getPassPercentage() + "%"
                );
            }
            
            log.info("Combined conformance report generated successfully");
            
        } catch (Exception e) {
            log.error("Combined conformance report generation failed", e);
            throw new AssertionError("Combined report failed: " + e.getMessage(), e);
        }
    }
    
    @AfterSuite
    public void cleanupTestEnvironment() {
        log.info("Cleaning up OpenID Conformance Test Suite");
        
        try {
            // Generate reports
            if (reportGenerator != null) {
                reportGenerator.finalizeReports();
            }
            
            // Cleanup containers
            if (orchestrationManager != null) {
                orchestrationManager.cleanupContainers();
                log.info("Containers cleaned up");
            }
            
        } catch (Exception e) {
            log.error("Error during cleanup", e);
            // Don't re-throw; cleanup errors shouldn't fail the suite
        }
    }
}
```

### 3. RestApiClient (HTTP Interface)

```java
package io.mosip.inji.conformance.integration;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.mosip.inji.conformance.config.ConformanceConfig;
import io.mosip.inji.conformance.config.TestPlanConfig;
import io.mosip.inji.conformance.exception.RestApiException;
import lombok.extern.slf4j.Slf4j;
import okhttp3.*;

import java.io.IOException;
import java.util.Map;
import java.util.concurrent.TimeUnit;

@Slf4j
public class RestApiClient {
    
    private final String baseUrl;
    private final OkHttpClient client;
    private final ObjectMapper objectMapper;
    
    public RestApiClient(ConformanceConfig config) {
        this.baseUrl = config.getOpenIDSuiteUrl(); // e.g., http://localhost:8080
        this.objectMapper = new ObjectMapper();
        
        // Configure OkHttpClient with timeouts
        this.client = new OkHttpClient.Builder()
            .connectTimeout(30, TimeUnit.SECONDS)
            .readTimeout(30, TimeUnit.SECONDS)
            .writeTimeout(30, TimeUnit.SECONDS)
            .retryOnConnectionFailure(true)
            .build();
    }
    
    /**
     * Create a test plan via REST API
     * POST /api/plan
     */
    public String createTestPlan(TestPlanConfig planConfig) {
        log.debug("Creating test plan: {}", planConfig);
        
        try {
            String jsonBody = objectMapper.writeValueAsString(planConfig);
            
            Request request = new Request.Builder()
                .url(baseUrl + "/api/plan")
                .post(RequestBody.create(jsonBody, MediaType.parse("application/json")))
                .addHeader("Content-Type", "application/json")
                .build();
            
            try (Response response = client.newCall(request).execute()) {
                if (!response.isSuccessful()) {
                    throw new RestApiException(
                        "Failed to create test plan. Status: " + response.code() +
                        ", Body: " + response.body().string()
                    );
                }
                
                String responseBody = response.body().string();
                Map<String, Object> responseMap = 
                    objectMapper.readValue(responseBody, Map.class);
                String planId = (String) responseMap.get("plan_id");
                
                log.info("Test plan created successfully. Plan ID: {}", planId);
                return planId;
            }
            
        } catch (IOException e) {
            log.error("Error creating test plan", e);
            throw new RestApiException("Create test plan failed: " + e.getMessage(), e);
        }
    }
    
    /**
     * Run/execute a test plan
     * POST /api/plan/{planId}/run
     */
    public String runTestPlan(String planId) {
        log.debug("Running test plan: {}", planId);
        
        try {
            Request request = new Request.Builder()
                .url(baseUrl + "/api/plan/" + planId + "/run")
                .post(RequestBody.create("", MediaType.parse("application/json")))
                .build();
            
            try (Response response = client.newCall(request).execute()) {
                if (!response.isSuccessful()) {
                    throw new RestApiException(
                        "Failed to run test plan. Status: " + response.code()
                    );
                }
                
                String responseBody = response.body().string();
                Map<String, Object> responseMap = 
                    objectMapper.readValue(responseBody, Map.class);
                String status = (String) responseMap.get("status");
                
                log.info("Test plan execution started. Status: {}", status);
                return status;
            }
            
        } catch (IOException e) {
            log.error("Error running test plan", e);
            throw new RestApiException("Run test plan failed: " + e.getMessage(), e);
        }
    }
    
    /**
     * Check test plan status
     * GET /api/plan/{planId}/status
     */
    public String checkTestPlanStatus(String planId) {
        try {
            Request request = new Request.Builder()
                .url(baseUrl + "/api/plan/" + planId + "/status")
                .get()
                .build();
            
            try (Response response = client.newCall(request).execute()) {
                if (!response.isSuccessful()) {
                    throw new RestApiException(
                        "Failed to check status. Status: " + response.code()
                    );
                }
                
                String responseBody = response.body().string();
                Map<String, Object> responseMap = 
                    objectMapper.readValue(responseBody, Map.class);
                return (String) responseMap.get("status");
            }
            
        } catch (IOException e) {
            log.error("Error checking test plan status", e);
            throw new RestApiException("Check status failed: " + e.getMessage(), e);
        }
    }
    
    /**
     * Get test results
     * GET /api/plan/{planId}/results
     */
    public Map<String, Object> getTestResults(String planId) {
        log.debug("Retrieving test results for plan: {}", planId);
        
        try {
            Request request = new Request.Builder()
                .url(baseUrl + "/api/plan/" + planId + "/results")
                .get()
                .build();
            
            try (Response response = client.newCall(request).execute()) {
                if (!response.isSuccessful()) {
                    throw new RestApiException(
                        "Failed to retrieve results. Status: " + response.code()
                    );
                }
                
                String responseBody = response.body().string();
                return objectMapper.readValue(responseBody, Map.class);
            }
            
        } catch (IOException e) {
            log.error("Error retrieving test results", e);
            throw new RestApiException("Get results failed: " + e.getMessage(), e);
        }
    }
}
```

### 4. PythonProcessBridge

```java
package io.mosip.inji.conformance.integration;

import io.mosip.inji.conformance.config.ConformanceConfig;
import io.mosip.inji.conformance.exception.ExecutionException;
import lombok.extern.slf4j.Slf4j;

import java.io.*;
import java.util.ArrayList;
import java.util.List;

@Slf4j
public class PythonProcessBridge {
    
    private final ConformanceConfig config;
    private final String pythonExecutable;
    private final String runTestPlanScript;
    
    public PythonProcessBridge(ConformanceConfig config) {
        this.config = config;
        this.pythonExecutable = config.getPythonExecutable(); // e.g., "python3"
        this.runTestPlanScript = config.getRunTestPlanScriptPath(); 
        // e.g., "/path/to/conformance-suite/scripts/run-test-plan.py"
    }
    
    /**
     * Invoke run-test-plan.py to execute a test plan
     * Command: python3 run-test-plan.py -server localhost:8080 -plan_id <id>
     */
    public String runTestPlan(String planId) {
        log.info("Invoking Python script to run test plan: {}", planId);
        
        try {
            List<String> command = buildCommand(planId);
            log.debug("Command: {}", String.join(" ", command));
            
            ProcessBuilder pb = new ProcessBuilder(command);
            pb.redirectErrorStream(true);
            
            Process process = pb.start();
            
            // Capture output
            String output = captureProcessOutput(process);
            log.debug("Python process output:\n{}", output);
            
            int exitCode = process.waitFor();
            
            if (exitCode != 0) {
                throw new ExecutionException(
                    "Python process exited with code " + exitCode + ". " +
                    "Output: " + output
                );
            }
            
            log.info("Test plan execution via Python script completed successfully");
            return "RUNNING";
            
        } catch (Exception e) {
            log.error("Error invoking Python script", e);
            throw new ExecutionException("Python process execution failed: " + e.getMessage(), e);
        }
    }
    
    private List<String> buildCommand(String planId) {
        List<String> command = new ArrayList<>();
        command.add(pythonExecutable);
        command.add(runTestPlanScript);
        command.add("-server");
        command.add(config.getOpenIDSuiteHost() + ":" + config.getOpenIDSuitePort());
        command.add("-plan_id");
        command.add(planId);
        
        if (config.getPythonVenv() != null) {
            command.add("-venv");
            command.add(config.getPythonVenv());
        }
        
        return command;
    }
    
    private String captureProcessOutput(Process process) throws IOException {
        StringBuilder output = new StringBuilder();
        
        try (BufferedReader reader = 
             new BufferedReader(new InputStreamReader(process.getInputStream()))) {
            String line;
            while ((line = reader.readLine()) != null) {
                output.append(line).append("\n");
            }
        }
        
        return output.toString();
    }
}
```

### 5. ResultMapper

```java
package io.mosip.inji.conformance.runner;

import io.mosip.inji.conformance.model.ComponentType;
import io.mosip.inji.conformance.model.ConformanceResult;
import io.mosip.inji.conformance.model.TestResult;
import io.mosip.inji.conformance.model.TestStatus;
import lombok.extern.slf4j.Slf4j;

import java.time.LocalDateTime;
import java.util.*;

@Slf4j
public class ResultMapper {
    
    /**
     * Map OpenID Conformance Suite REST response to ConformanceResult
     */
    public ConformanceResult mapToConformanceResult(
            Map<String, Object> rawResults,
            ComponentType component,
            String planId) {
        
        log.debug("Mapping results for component: {} with plan ID: {}", component, planId);
        
        ConformanceResult result = new ConformanceResult();
        result.setComponent(component);
        result.setPlanId(planId);
        result.setExecutionTime(LocalDateTime.now());
        
        List<TestResult> testResults = new ArrayList<>();
        
        // Iterate through raw results from OpenID Suite API
        List<Map<String, Object>> tests = 
            (List<Map<String, Object>>) rawResults.get("tests");
        
        int passCount = 0;
        int failCount = 0;
        int skipCount = 0;
        
        if (tests != null) {
            for (Map<String, Object> testData : tests) {
                TestResult testResult = mapTestResult(testData);
                testResults.add(testResult);
                
                switch (testResult.getStatus()) {
                    case PASS:
                        passCount++;
                        break;
                    case FAIL:
                        failCount++;
                        break;
                    case SKIP:
                        skipCount++;
                        break;
                }
            }
        }
        
        result.setTestResults(testResults);
        result.setTotalTests(testResults.size());
        result.setPassedTests(passCount);
        result.setFailedTests(failCount);
        result.setSkippedTests(skipCount);
        result.setPassPercentage((passCount * 100.0) / result.getTotalTests());
        
        log.info("Results mapped: {} total, {} passed, {} failed, {} skipped",
                 result.getTotalTests(), passCount, failCount, skipCount);
        
        return result;
    }
    
    /**
     * Map individual test result from OpenID response
     */
    private TestResult mapTestResult(Map<String, Object> testData) {
        TestResult result = new TestResult();
        
        result.setTestName((String) testData.get("test_name"));
        result.setTestId((String) testData.get("test_id"));
        result.setDescription((String) testData.get("description"));
        
        String status = ((String) testData.get("result")).toUpperCase();
        result.setStatus(TestStatus.valueOf(status));
        
        if (testData.containsKey("error")) {
            result.setErrorMessage((String) testData.get("error"));
        }
        
        result.setExecutionTime(
            ((Number) testData.getOrDefault("execution_time_ms", 0)).longValue()
        );
        
        return result;
    }
}
```

### 6. BenchmarkGateway

```java
package io.mosip.inji.conformance.reporting;

import io.mosip.inji.conformance.config.ConformanceConfig;
import io.mosip.inji.conformance.model.ComponentType;
import io.mosip.inji.conformance.model.ConformanceResult;
import lombok.extern.slf4j.Slf4j;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.Map;

import com.fasterxml.jackson.databind.ObjectMapper;

@Slf4j
public class BenchmarkGateway {
    
    private final ConformanceConfig config;
    private final Map<ComponentType, Double> benchmarks;
    private final ObjectMapper objectMapper;
    
    public BenchmarkGateway(ConformanceConfig config) {
        this.config = config;
        this.objectMapper = new ObjectMapper();
        this.benchmarks = loadBenchmarks();
    }
    
    /**
     * Validate conformance result against benchmark threshold
     * Returns true if result meets or exceeds minimum pass percentage
     */
    public boolean validate(ConformanceResult result, ComponentType component) {
        Double threshold = benchmarks.getOrDefault(component, 95.0);
        
        log.info("Validating {} conformance against benchmark: {}%",
                 component, threshold);
        log.info("Actual pass rate: {}%", result.getPassPercentage());
        
        boolean passed = result.getPassPercentage() >= threshold;
        
        if (passed) {
            log.info("✓ {} conformance PASSED benchmark validation", component);
        } else {
            log.warn("✗ {} conformance FAILED benchmark validation. " +
                    "Expected: {}%, Got: {}%",
                    component, threshold, result.getPassPercentage());
        }
        
        return passed;
    }
    
    /**
     * Check if result shows regression from baseline
     */
    public boolean hasRegression(ConformanceResult current, ConformanceResult baseline) {
        double diff = baseline.getPassPercentage() - current.getPassPercentage();
        
        if (diff > 0) {
            log.warn("Regression detected: {}% drop in pass rate", diff);
            return true;
        }
        
        return false;
    }
    
    /**
     * Load benchmark thresholds from config file
     */
    private Map<ComponentType, Double> loadBenchmarks() {
        try {
            String configContent = new String(
                Files.readAllBytes(Paths.get(config.getBenchmarkConfigPath()))
            );
            
            Map<String, Double> rawBenchmarks = 
                objectMapper.readValue(configContent, Map.class);
            
            Map<ComponentType, Double> typedBenchmarks = new java.util.HashMap<>();
            rawBenchmarks.forEach((k, v) -> {
                typedBenchmarks.put(ComponentType.valueOf(k.toUpperCase()), v);
            });
            
            log.info("Benchmarks loaded: {}", typedBenchmarks);
            return typedBenchmarks;
            
        } catch (IOException e) {
            log.error("Failed to load benchmarks, using defaults", e);
            return Map.of(
                ComponentType.CERTIFY, 95.0,
                ComponentType.VERIFY, 95.0
            );
        }
    }
}
```

### 7. OrchestrationManager

```java
package io.mosip.inji.conformance.orchestration;

import io.mosip.inji.conformance.config.ConformanceConfig;
import io.mosip.inji.conformance.integration.PythonProcessBridge;
import io.mosip.inji.conformance.integration.RestApiClient;
import io.mosip.inji.conformance.model.ConformanceResult;
import io.mosip.inji.conformance.reporting.BenchmarkGateway;
import io.mosip.inji.conformance.reporting.ExtentReportGenerator;
import io.mosip.inji.conformance.runner.ConformanceRunner;
import io.mosip.inji.conformance.runner.ResultMapper;
import lombok.extern.slf4j.Slf4j;

@Slf4j
public class OrchestrationManager {
    
    private final ConformanceConfig config;
    private final ComponentLifecycleManager lifecycleManager;
    private ConformanceRunner conformanceRunner;
    
    public OrchestrationManager(ConformanceConfig config) {
        this.config = config;
        this.lifecycleManager = new ComponentLifecycleManager(config);
    }
    
    /**
     * Initialize infrastructure (Docker containers, services)
     */
    public void spinUpContainers() {
        log.info("Spinning up Docker containers...");
        lifecycleManager.spinUpDockerCompose();
        log.info("Containers spun up successfully");
    }
    
    /**
     * Wait for services to be healthy and ready
     */
    public void waitForServicesReady(int timeoutSeconds) {
        log.info("Waiting for services to be ready (timeout: {}s)", timeoutSeconds);
        lifecycleManager.waitForServiceHealth(timeoutSeconds);
        log.info("All services ready");
    }
    
    /**
     * Get ConformanceRunner (lazy initialization)
     */
    public ConformanceRunner getConformanceRunner() {
        if (conformanceRunner == null) {
            RestApiClient apiClient = new RestApiClient(config);
            PythonProcessBridge pythonBridge = new PythonProcessBridge(config);
            ResultMapper resultMapper = new ResultMapper();
            BenchmarkGateway benchmarkGateway = new BenchmarkGateway(config);
            
            conformanceRunner = new ConformanceRunner(
                config,
                apiClient,
                pythonBridge,
                resultMapper,
                benchmarkGateway
            );
        }
        return conformanceRunner;
    }
    
    /**
     * Execute combined run (Certify → Verify)
     */
    public void executeCombinedRun() {
        log.info("Executing combined (Certify ↔ Verify) run");
        
        ConformanceRunner runner = getConformanceRunner();
        
        ConformanceResult issuerResult = 
            runner.runIssuerPlan(config.getCertifyEndpoint());
        
        ConformanceResult verifierResult = 
            runner.runVerifierPlan(config.getVerifyEndpoint());
        
        boolean allPassed = issuerResult.isBenchmarkPassed() &&
                            verifierResult.isBenchmarkPassed();
        
        log.info("Combined run completed. Issuer: {}%, Verifier: {}%",
                 issuerResult.getPassPercentage(),
                 verifierResult.getPassPercentage());
        
        if (!allPassed) {
            throw new AssertionError(
                "Combined conformance failed benchmark validation"
            );
        }
    }
    
    /**
     * Cleanup Docker containers and temporary resources
     */
    public void cleanupContainers() {
        log.info("Cleaning up containers...");
        lifecycleManager.cleanupDockerCompose();
        log.info("Cleanup completed");
    }
}
```

---

## Configuration Management

### conformance-config.json

```json
{
  "openidSuite": {
    "host": "localhost",
    "port": 8080,
    "apiPath": "/api",
    "baseUrl": "http://localhost:8080"
  },
  "components": {
    "certify": {
      "name": "Inji Certify",
      "endpoint": "http://localhost:8091",
      "port": 8091,
      "testPlanTemplate": "issuer-plan-1.0.json",
      "aliases": ["certify_issuer", "issuer"],
      "variants": ["oid4vc_sd_jwt", "oid4vc_jwt_vc"]
    },
    "verify": {
      "name": "Inji Verify",
      "endpoint": "http://localhost:8092",
      "port": 8092,
      "testPlanTemplate": "verifier-plan-1.0.json",
      "aliases": ["verify_verifier", "verifier"],
      "variants": ["oid4vp"]
    }
  },
  "benchmark": {
    "certify": {
      "minPassPercentage": 95.0,
      "failOnRegression": true
    },
    "verify": {
      "minPassPercentage": 95.0,
      "failOnRegression": true
    }
  },
  "timeouts": {
    "planExecutionSeconds": 300,
    "pollIntervalMs": 2000,
    "pollMaxRetries": 150,
    "startupTimeoutSeconds": 60,
    "serviceHealthCheckTimeoutSeconds": 120
  },
  "docker": {
    "composeFile": "docker/compose/docker-compose.yml",
    "project": "inji-conformance"
  },
  "python": {
    "executable": "python3",
    "runTestPlanScript": "/path/to/conformance-suite/scripts/run-test-plan.py",
    "venv": "/opt/conformance-venv"
  },
  "reporting": {
    "outputDir": "./conformance-reports",
    "uploadToS3": true,
    "s3Bucket": "inji-conformance-reports",
    "s3Region": "us-east-1"
  },
  "logging": {
    "level": "INFO",
    "file": "./logs/conformance.log"
  }
}
```

### issuer-plan-1.0.json (Test Plan Template)

```json
{
  "plan_name": "Inji Certify OpenID 1.0 Issuer Conformance",
  "specification": "openid-for-verifiable-credentials-1.0",
  "plan_type": "issuer",
  "modules": [
    "openid4vc-verifiable-credentials-issuance"
  ],
  "variants": ["oid4vc_sd_jwt", "oid4vc_jwt_vc"],
  "actions": [
    "credentialOffer",
    "credentialRequest",
    "credentialResponse",
    "deferredCredentialRequest",
    "deferredCredentialResponse"
  ],
  "endpoint": "PLACEHOLDER_ENDPOINT",
  "public_client": true,
  "discoveryUrl": null,
  "config": {
    "issuer_url": "PLACEHOLDER_ENDPOINT",
    "client_id": "test-client",
    "response_type": "code",
    "response_mode": "form_post"
  }
}
```

### docker-compose.yml

```yaml
version: '3.8'

services:
  # OpenID Conformance Suite
  conformance-server:
    image: openidcert/conformance:latest
    container_name: conformance-server
    environment:
      - DATABASE_URL=mongodb://conformance-mongodb:27017/conformance
      - JAVA_OPTS=-Xms1024m -Xmx2048m
    ports:
      - "8080:8080"
    depends_on:
      - conformance-mongodb
    networks:
      - inji-conformance
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 10s
      timeout: 5s
      retries: 5

  conformance-mongodb:
    image: mongo:4.4
    container_name: conformance-mongodb
    environment:
      - MONGO_INITDB_DATABASE=conformance
    volumes:
      - conformance-db:/data/db
    networks:
      - inji-conformance
    healthcheck:
      test: ["CMD", "mongo", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      timeout: 5s
      retries: 5

  conformance-httpd:
    image: httpd:2.4
    container_name: conformance-httpd
    ports:
      - "8081:80"
    volumes:
      - ./docker/httpd.conf:/usr/local/apache2/conf/httpd.conf:ro
    networks:
      - inji-conformance

  # Inji Certify
  inji-certify:
    build:
      context: ./inji-certify-source
      dockerfile: Dockerfile
    container_name: inji-certify
    environment:
      - SERVER_PORT=8091
      - SPRING_PROFILES_ACTIVE=dev
    ports:
      - "8091:8091"
    networks:
      - inji-conformance
    depends_on:
      - conformance-server
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8091/health"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Inji Verify
  inji-verify:
    build:
      context: ./inji-verify-source
      dockerfile: Dockerfile
    container_name: inji-verify
    environment:
      - SERVER_PORT=8092
      - SPRING_PROFILES_ACTIVE=dev
    ports:
      - "8092:8092"
    networks:
      - inji-conformance
    depends_on:
      - conformance-server
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8092/health"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  conformance-db:

networks:
  inji-conformance:
    driver: bridge
```

---

## Shell Scripts & Execution

### run-conformance.sh (Main Entry Point)

```bash
#!/bin/bash

set -e

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# Configuration
PROJECT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
DOCKER_COMPOSE_FILE="${PROJECT_DIR}/docker/compose/docker-compose.yml"
LOG_DIR="${PROJECT_DIR}/logs"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

# Ensure logs directory exists
mkdir -p "${LOG_DIR}"

# Log file
LOG_FILE="${LOG_DIR}/conformance_${TIMESTAMP}.log"

# Functions
log_info() {
    echo -e "${GREEN}[INFO]${NC} $1" | tee -a "${LOG_FILE}"
}

log_warn() {
    echo -e "${YELLOW}[WARN]${NC} $1" | tee -a "${LOG_FILE}"
}

log_error() {
    echo -e "${RED}[ERROR]${NC} $1" | tee -a "${LOG_FILE}"
}

cleanup_on_exit() {
    local exit_code=$?
    log_info "Cleaning up resources..."
    docker-compose -f "${DOCKER_COMPOSE_FILE}" down >> "${LOG_FILE}" 2>&1 || true
    exit ${exit_code}
}

trap cleanup_on_exit EXIT

# Parse arguments
COMPONENT="${1:---combined}"
RUN_MODE="${2:-default}"

log_info "=== Inji Conformance Testing Harness ==="
log_info "Component: ${COMPONENT}"
log_info "Mode: ${RUN_MODE}"
log_info "Log file: ${LOG_FILE}"

# Spin up containers
log_info "Spinning up Docker containers..."
docker-compose -f "${DOCKER_COMPOSE_FILE}" up -d >> "${LOG_FILE}" 2>&1

# Wait for services
log_info "Waiting for services to be healthy..."
sleep 15

# Run Maven tests
case "${COMPONENT}" in
    --component)
        if [ "$2" = "certify" ]; then
            log_info "Running Certify conformance tests..."
            cd "${PROJECT_DIR}"
            mvn clean test -Dtest=OpenIDConformanceTest#testIssuerConformance \
                -DfailIfNoTests=false \
                -Dorg.slf4j.simpleLogger.defaultLogLevel=info \
                2>&1 | tee -a "${LOG_FILE}"
        elif [ "$2" = "verify" ]; then
            log_info "Running Verify conformance tests..."
            cd "${PROJECT_DIR}"
            mvn clean test -Dtest=OpenIDConformanceTest#testVerifierConformance \
                -DfailIfNoTests=false \
                2>&1 | tee -a "${LOG_FILE}"
        else
            log_error "Invalid component: $2"
            exit 1
        fi
        ;;
    --combined)
        log_info "Running combined (Certify ↔ Verify) conformance tests..."
        cd "${PROJECT_DIR}"
        mvn clean test -Dtest=OpenIDConformanceTest \
            -DfailIfNoTests=false \
            2>&1 | tee -a "${LOG_FILE}"
        ;;
    *)
        log_error "Usage: $0 [--component certify|verify|--combined]"
        exit 1
        ;;
esac

# Check exit code
if [ $? -eq 0 ]; then
    log_info "✓ Conformance tests completed successfully"
    log_info "Reports available at: ${PROJECT_DIR}/conformance-reports"
else
    log_error "✗ Conformance tests failed"
    exit 1
fi
```

### run-conformance-certify.sh

```bash
#!/bin/bash
exec "$(dirname "$0")/run-conformance.sh" --component certify
```

### run-conformance-verify.sh

```bash
#!/bin/bash
exec "$(dirname "$0")/run-conformance.sh" --component verify
```

### run-conformance-combined.sh

```bash
#!/bin/bash
exec "$(dirname "$0")/run-conformance.sh" --combined
```

### cleanup-conformance.sh

```bash
#!/bin/bash

set -e

PROJECT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
DOCKER_COMPOSE_FILE="${PROJECT_DIR}/docker/compose/docker-compose.yml"

echo "Stopping and removing containers..."
docker-compose -f "${DOCKER_COMPOSE_FILE}" down -v

echo "Cleanup completed"
```

---

## GitHub Actions Workflow

### .github/workflows/full-stack-conformance.yml

```yaml
name: Full Stack Conformance Testing

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]
  schedule:
    # Daily at 2 AM UTC
    - cron: '0 2 * * *'

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      artifact-path: ${{ steps.build.outputs.artifact-path }}
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Java
        uses: actions/setup-java@v3
        with:
          java-version: '11'
          distribution: 'temurin'
          cache: maven
      
      - name: Build project
        id: build
        run: |
          mvn clean package -DskipTests
          echo "artifact-path=$(find target -name '*.jar' | head -1)" >> $GITHUB_OUTPUT

  conformance-certify:
    runs-on: ubuntu-latest
    needs: build
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Java
        uses: actions/setup-java@v3
        with:
          java-version: '11'
          distribution: 'temurin'
          cache: maven
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2
      
      - name: Docker Compose up
        run: |
          docker-compose -f docker/compose/docker-compose.yml up -d
          sleep 30
      
      - name: Run Certify conformance tests
        run: |
          mvn clean test \
            -Dtest=OpenIDConformanceTest#testIssuerConformance \
            -DfailIfNoTests=false
      
      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: certify-reports
          path: conformance-reports/
      
      - name: Docker Compose down
        if: always()
        run: docker-compose -f docker/compose/docker-compose.yml down -v

  conformance-verify:
    runs-on: ubuntu-latest
    needs: build
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Java
        uses: actions/setup-java@v3
        with:
          java-version: '11'
          distribution: 'temurin'
          cache: maven
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2
      
      - name: Docker Compose up
        run: |
          docker-compose -f docker/compose/docker-compose.yml up -d
          sleep 30
      
      - name: Run Verify conformance tests
        run: |
          mvn clean test \
            -Dtest=OpenIDConformanceTest#testVerifierConformance \
            -DfailIfNoTests=false
      
      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: verify-reports
          path: conformance-reports/
      
      - name: Docker Compose down
        if: always()
        run: docker-compose -f docker/compose/docker-compose.yml down -v

  conformance-combined:
    runs-on: ubuntu-latest
    needs: [conformance-certify, conformance-verify]
    if: always()
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Java
        uses: actions/setup-java@v3
        with:
          java-version: '11'
          distribution: 'temurin'
          cache: maven
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2
      
      - name: Docker Compose up
        run: |
          docker-compose -f docker/compose/docker-compose.yml up -d
          sleep 30
      
      - name: Run combined conformance tests
        run: |
          mvn clean test \
            -Dtest=OpenIDConformanceTest#testCombinedConformance \
            -DfailIfNoTests=false
      
      - name: Upload combined reports
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: combined-reports
          path: conformance-reports/
      
      - name: Docker Compose down
        if: always()
        run: docker-compose -f docker/compose/docker-compose.yml down -v
      
      - name: Comment on PR
        if: github.event_name == 'pull_request' && always()
        uses: actions/github-script@v6
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: 'OpenID Conformance tests completed. Check artifacts for detailed reports.'
            })

  publish:
    runs-on: ubuntu-latest
    needs: [conformance-certify, conformance-verify, conformance-combined]
    if: success() && github.event_name == 'push' && github.ref == 'refs/heads/main'
    
    permissions:
      contents: read
      packages: write
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Java
        uses: actions/setup-java@v3
        with:
          java-version: '11'
          distribution: 'temurin'
          cache: maven
      
      - name: Build and publish artifact
        run: |
          mvn deploy
```

---

## Data Flow & Models

### ConformanceResult Model

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class ConformanceResult {
    private ComponentType component;
    private String planId;
    private LocalDateTime executionTime;
    private long executionDurationMs;
    
    private List<TestResult> testResults;
    private int totalTests;
    private int passedTests;
    private int failedTests;
    private int skippedTests;
    private double passPercentage;
    
    private boolean benchmarkPassed;
    private Map<String, String> metadata; // Additional context
}
```

### TestResult Model

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class TestResult {
    private String testId;
    private String testName;
    private String description;
    private TestStatus status; // PASS, FAIL, SKIP
    private String errorMessage;
    private long executionTimeMs;
    private Map<String, Object> details;
}
```

### Full Data Flow

```
OpenID Conformance Suite
  ↓
REST API Response (JSON)
  ↓
RestApiClient (HTTP layer)
  ↓
ResultMapper (JSON → Java models)
  ↓
ConformanceResult
  ↓
BenchmarkGateway (Validate)
  ↓
TestNGResultAdapter (Map to TestNG lifecycle)
  ↓
ExtentReportGenerator (HTML report)
  ↓
S3 Bucket + Test Reporting Dashboard
```

---

## Testing Strategy

### Unit Tests (45% of test suite)
```
Target Coverage: 80%+ code coverage

Tests:
├── ConfigValidator Tests
│   ├── Valid config parsing
│   ├── Invalid config rejection
│   └── Endpoint validation
├── ResultMapper Tests
│   ├── JSON parsing
│   ├── Status mapping
│   └── Percentage calculation
├── BenchmarkGateway Tests
│   ├── Pass/fail threshold logic
│   ├── Regression detection
│   └── Baseline comparison
└── RestApiClient Tests (Mock)
    ├── Request building
    ├── Response parsing
    └── Error handling
```

### Integration Tests (35% of test suite)
```
Tests:
├── Docker Compose Tests
│   ├── Container startup
│   ├── Service health checks
│   └── Cleanup
├── RestApiClient Tests (Real)
│   ├── Create plan
│   ├── Run plan
│   ├── Check status
│   └── Get results
├── PythonProcessBridge Tests
│   ├── Process invocation
│   ├── Output capture
│   └── Error handling
└── End-to-End Tests
    ├── Single component run
    ├── Combined run
    └── Report generation
```

### E2E Tests (20% of test suite)
```
Tests:
├── Full stack execution (Certify + Verify)
├── Report generation & validation
├── S3 upload
├── Benchmark gating
└── Regression detection
```

---

## Documentation Plan

| Document | Purpose | Audience | Length |
|----------|---------|----------|--------|
| **README.md** | Quick start guide | Everyone | 1 page |
| **ARCHITECTURE.md** | System design & components | Developers | 5-10 pages |
| **SETUP_GUIDE.md** | Installation & env setup | DevOps/Developers | 3-5 pages |
| **API_AUTOMATION.md** | REST API details & Python bridge | Developers | 3-5 pages |
| **DEVELOPMENT.md** | Local dev setup & contribution | Developers | 3-5 pages |
| **TROUBLESHOOTING.md** | Common issues & solutions | DevOps | 3-5 pages |
| **CERTIFICATION_WORKFLOW.md** | OpenID submission process | Product | 2-3 pages |
| **Architecture Diagrams** | System & flow visualizations | Everyone | Mermaid/PNG |

---

## Success Criteria & Metrics

### Mandatory (Must Have)
- ✅ Docker Compose brings up all 3 services
- ✅ REST API calls create & execute test plans (issuer & verifier)
- ✅ Results mapped to TestNG models
- ✅ Per-module test suites integrated into api-testrig
- ✅ Extent reports generated & visible
- ✅ Benchmark gating works (pass/fail)
- ✅ Single `run-conformance.sh` command for all modes
- ✅ CI/CD workflow executes automatically

### Good-to-Have (Should Have)
- ✅ Result diff utility (`diff-results.py`)
- ✅ Expected-failures tracking
- ✅ GitHub Actions workflow active
- ✅ Full documentation
- ✅ Logging & observability

### Bonus (Nice to Have)
- ✅ Parallel execution (Certify + Verify simultaneously)
- ✅ Dashboard/badge for certification status
- ✅ Automated regression detection

### Quality Metrics
- **Code Coverage:** > 80% (unit + integration)
- **Test Success Rate:** > 95%
- **Documentation Completeness:** 100%
- **Build Time:** < 10 minutes (end-to-end)
- **Mean Time to Detection (MTTD):** < 1 minute for regressions

---

## Deployment & Release

### Build Artifact
```bash
mvn clean package
# Output: target/inji-conformance-harness-1.0.0.jar
```

### Docker Image
```bash
docker build -t inji-conformance:1.0.0 .
docker push ghcr.io/inji/inji-conformance:1.0.0
```

### Deployment Steps
```bash
# 1. Pull latest image
docker pull ghcr.io/inji/inji-conformance:1.0.0

# 2. Run conformance (all modes)
docker run --rm -v $(pwd)/conformance-reports:/app/reports \
  ghcr.io/inji/inji-conformance:1.0.0 \
  ./run-conformance.sh --combined
```

### Release Checklist
- [ ] All unit & integration tests passing (CI green)
- [ ] Code coverage > 80%
- [ ] Documentation complete & reviewed
- [ ] CHANGELOG.md updated
- [ ] Version bumped (semantic versioning)
- [ ] Artifact published to Maven Central / Artifactory
- [ ] Docker image published to registry
- [ ] GitHub release with changelog
- [ ] Team trained on operation

---

## Troubleshooting Guide

### Container Issues

**Problem:** Containers fail to start
```bash
# Debug:
docker-compose -f docker/compose/docker-compose.yml logs -f
docker ps -a

# Solution:
docker-compose down -v
docker-compose up -d --build
```

**Problem:** Services not healthy
```bash
# Check health:
docker inspect --format='{{json .State.Health.Status}}' conformance-server

# Wait longer:
sleep 60 && docker-compose ps
```

### REST API Issues

**Problem:** Test plan creation fails
```log
[ERROR] Failed to create test plan. Status: 500

Solution:
1. Check OpenID Suite is running: curl http://localhost:8080/health
2. Verify endpoints in conformance-config.json
3. Check conformance-suite logs: docker logs conformance-server
```

**Problem:** Plan execution times out
```bash
# Solution:
1. Increase timeout in config:
   "planExecutionSeconds": 600  # from 300

2. Check component logs:
   docker logs inji-certify
   docker logs inji-verify
```

### Reporting Issues

**Problem:** Reports not generated
```bash
# Check:
ls -la conformance-reports/

# If empty:
1. Ensure TestNG execution succeeded
2. Check ExtentReportGenerator logs
3. Verify S3 credentials if uploading
```

### Python Bridge Issues

**Problem:** Python script not found
```bash
Error: run-test-plan.py not found

Solution:
1. Check path: ls /path/to/conformance-suite/scripts/
2. Update conformance-config.json:
   "runTestPlanScript": "/correct/path/run-test-plan.py"
```

### Common Errors & Solutions

| Error | Cause | Solution |
|-------|-------|----------|
| `ConnectException: Connection refused` | Service not running | Check Docker logs, increase startup timeout |
| `JsonMappingException` | Response JSON doesn't match model | Update model, check API response format |
| `BenchmarkGatewayException: 45% < 95%` | Test results below threshold | Review OpenID compliance, check logs for failures |
| `TimeoutException: Poll exceeded max retries` | Test plan stuck | Check OpenID Suite, restart containers |
| `FileNotFoundException: conformance-config.json` | Config file missing | Run from project root, check path |

---

## Summary

This project structure provides **production-grade automation** for OpenID conformance testing across Inji Certify and Inji Verify. The layered architecture ensures **separation of concerns**, the documented roadmap guides **phased implementation**, and the comprehensive testing strategy guarantees **reliability at scale**.

### Key Takeaways
- **13-week delivery timeline** with clear phase gates
- **5 architectural layers** for maintainability
- **Java + TestNG + Docker** stack matches existing MOSIP patterns
- **Per-module + combined run modes** for flexible validation
- **CI/CD-ready** with GitHub Actions integration
- **Production-deployable** with comprehensive documentation

---

**Document Version:** 1.0  
**Last Updated:** September 2026  
**Status:** Ready for Implementation
