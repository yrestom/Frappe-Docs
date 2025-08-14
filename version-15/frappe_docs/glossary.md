# Frappe Framework - Glossary

> **Comprehensive terminology reference for Frappe Framework concepts and components**

## A

**API (Application Programming Interface)**  
RESTful endpoints automatically generated for all DocTypes, allowing external systems to interact with Frappe applications.

**App**  
A modular application built on Frappe Framework. ERPNext is an app, and developers can create custom apps with specific business logic.

**Auto-naming**  
System for automatically generating unique document names using patterns like naming series, field values, or custom expressions.

**Awesome Bar**  
Global search interface in Frappe's UI that allows quick navigation to documents, reports, and system functions.

## B

**Background Jobs**  
Asynchronous task processing system using Redis and RQ (Redis Queue) for long-running operations without blocking the user interface.

**Bench**  
Command-line tool for managing Frappe sites, apps, and development environments. Essential for installation, updates, and maintenance.

**BaseDocument**  
Core Python class that all document controllers inherit from, providing basic CRUD operations and lifecycle management.

## C

**Child Table**  
Table fields within a DocType that store related records. Examples include items in an invoice or contact details in a customer record.

**Controller**  
Python class that implements business logic for a DocType. Contains methods like validate(), before_save(), after_insert().

**Custom Field**  
Additional fields added to existing DocTypes without modifying core framework code. Stored separately and merged at runtime.

**CSRF (Cross-Site Request Forgery)**  
Security mechanism implemented in Frappe to prevent unauthorized requests. Requires tokens for state-changing operations.

## D

**DocField**  
Individual field definition within a DocType, specifying field type, properties, validation rules, and display characteristics.

**DocPerm**  
Permission rules defining what roles can do with a DocType (read, write, create, delete, submit, cancel).

**DocStatus**  
Document status indicator: 0 (Draft), 1 (Submitted), 2 (Cancelled). Controls document lifecycle and permissions.

**DocType**  
Metadata definition that describes the structure of a document type. Think of it as a table schema with additional properties.

**Document**  
Instance of a DocType. Represents a single record like a specific customer, invoice, or user account.

**Dynamic Link**  
Field type that can reference different DocTypes based on another field's value. Useful for flexible relationships.

## E

**Email Queue**  
System for managing email sending, including templates, scheduling, and delivery tracking.

**Event Hooks**  
Mechanism to execute custom code when specific events occur (before_save, after_insert, etc.).

## F

**Field Level Permissions**  
Granular access control that restricts read/write access to specific fields based on user roles.

**Form Script**  
JavaScript code that customizes form behavior, validation, and user interface interactions.

**Frappe.call()**  
JavaScript method for making AJAX requests to server-side Python methods.

**Fixtures**  
Pre-defined data records that are installed with an app, typically master data like roles, custom fields, or configurations.

## G

**Global Search**  
System-wide search functionality that indexes document content for fast retrieval across all DocTypes.

**Grid**  
User interface component for displaying and editing child table data in a tabular format.

## H

**Hooks**  
Configuration mechanism allowing apps to extend framework behavior without modifying core code.

**HTML Field**  
Field type for displaying rich HTML content, often used for instructions or dynamic content display.

## I

**Integration**  
Connection between Frappe and external systems through APIs, webhooks, or data synchronization.

**InnoDB**  
Default database engine for Frappe applications, providing ACID compliance and transaction support.

## J

**Jinja2**  
Template engine used for email templates, print formats, and dynamic content generation.

**JSON Field**  
Field type for storing structured data in JSON format, useful for configuration or flexible data storage.

## L

**Link Field**  
Field type that creates a relationship to another DocType, similar to a foreign key in traditional databases.

**List View**  
Interface for displaying multiple documents of the same DocType with filtering, sorting, and bulk operations.

**Login Manager**  
System component handling user authentication, session management, and security policies.

## M

**Metadata**  
Data about data - the DocType definitions, field properties, and permissions that define application structure.

**Migration**  
Process of updating database schema and data when app versions change. Handled automatically by Frappe.

**Module**  
Organizational unit within an app that groups related DocTypes and functionality. Examples: Accounting, HR, Assets.

## N

**Naming Series**  
Pattern-based system for generating sequential document names like INV-2024-00001, PO-NYC-00001.

**Nginx**  
Web server used in production deployments to serve static files and proxy requests to Frappe applications.

## O

**ORM (Object Relational Mapping)**  
Frappe's system for interacting with the database using Python objects instead of raw SQL queries.

**Override**  
Technique for modifying existing functionality by creating custom methods that replace standard behavior.

## P

**Permission Level**  
Numeric levels (0, 1, 2, etc.) that control access to different sets of fields within a DocType.

**Print Format**  
Template system for generating formatted documents like invoices, reports, and certificates.

**Portal**  
Web interface for external users (customers, suppliers) to access limited functionality without full system access.

## Q

**Query Builder**  
PyPika-based system for constructing database queries programmatically with type safety and IDE support.

**Query Report**  
Interactive report type that allows users to apply filters and view results in real-time.

**Queue**  
Background job processing queues (short, default, long) for different types of asynchronous tasks.

## R

**Rate Limiting**  
Security mechanism that restricts the number of API requests per user within a time window.

**Redis**  
In-memory database used for caching, session storage, and background job queuing.

**Role**  
User permission group that defines what DocTypes and functions a user can access.

**Route**  
URL pattern that maps to specific views, forms, or reports within the Frappe application.

## S

**Script Report**  
Report type where logic is implemented in Python, generating data that's displayed in a standard format.

**Session**  
User's authenticated session containing user information, roles, and temporary data.

**Single DocType**  
Special DocType that stores only one record, typically used for application settings and configurations.

**SocketIO**  
Real-time communication system for live updates, notifications, and collaborative features.

**Submittable**  
DocType property that enables submit/cancel workflow, making documents immutable after submission.

## T

**Table Field**  
Field type that contains child table records, creating a one-to-many relationship structure.

**Template**  
Reusable HTML/Jinja2 template for emails, print formats, or web pages.

**Thread Local**  
Python pattern used by Frappe to maintain request-specific data (current user, database connection, etc.).

**Timeline**  
Activity feed showing document changes, comments, and related activities in chronological order.

**Tree DocType**  
Hierarchical document structure where records can have parent-child relationships.

## U

**User Permission**  
Document-level restriction that limits which specific records a user can access within a DocType.

**User Type**  
Classification of users (System User, Website User) that determines access levels and interface.

## V

**Validation**  
Process of checking data integrity and business rules before saving documents to the database.

**Virtual DocType**  
DocType that doesn't store data in the database but represents external data sources or computed views.

## W

**Webhook**  
HTTP callbacks triggered by document events, allowing external systems to receive real-time updates.

**Whitelist**  
Security mechanism requiring explicit decoration (@frappe.whitelist()) for methods accessible via HTTP requests.

**Workflow**  
Configurable approval process that routes documents through different states and users for review.

**Workspace**  
Customizable dashboard interface where users can organize shortcuts, charts, and reports.

**WSGI (Web Server Gateway Interface)**  
Python web application interface used by Frappe for handling HTTP requests and responses.

## Technical Terms

**frappe.db**  
Global database connection object providing methods for queries, transactions, and database operations.

**frappe.local**  
Thread-local storage containing request-specific data like current user, site configuration, and form data.

**frappe.qb**  
Query builder instance for constructing type-safe database queries using PyPika syntax.

**frappe.session**  
Current user session object containing authentication status and user information.

**frm**  
JavaScript variable representing the current form instance in client-side scripts.

**cur_frm**  
Global JavaScript variable for accessing the currently active form (legacy pattern).

**locals**  
JavaScript object containing all document data loaded in the current form, including child table records.

**cdt/cdn**  
Child DocType and Child Document Name - parameters passed to child table event handlers.

---

## Acronyms & Abbreviations

- **API**: Application Programming Interface
- **CRUD**: Create, Read, Update, Delete
- **CSS**: Cascading Style Sheets
- **CSV**: Comma Separated Values
- **CSRF**: Cross-Site Request Forgery
- **DB**: Database
- **DNS**: Domain Name System
- **ERP**: Enterprise Resource Planning
- **HTML**: HyperText Markup Language
- **HTTP**: HyperText Transfer Protocol
- **HTTPS**: HTTP Secure
- **JS**: JavaScript
- **JSON**: JavaScript Object Notation
- **JWT**: JSON Web Token
- **ORM**: Object Relational Mapping
- **PDF**: Portable Document Format
- **REST**: Representational State Transfer
- **RQ**: Redis Queue
- **SQL**: Structured Query Language
- **SSL**: Secure Sockets Layer
- **TLS**: Transport Layer Security
- **UI**: User Interface
- **URL**: Uniform Resource Locator
- **UUID**: Universally Unique Identifier
- **WSGI**: Web Server Gateway Interface
- **XSS**: Cross-Site Scripting

---

## Advanced Framework Terms (Files 05-12)

### API & Integration Terms

**API Key Authentication**  
Token-based authentication system where clients include an API key in request headers for authorization.

**API Versioning**  
System for managing different versions of API endpoints (v1, v2) to maintain backward compatibility.

**Bearer Token**  
Authentication method where clients include `Authorization: Bearer <token>` header in API requests.

**Circuit Breaker Pattern**  
Design pattern that prevents system failures by temporarily blocking requests to failing external services.

**CORS (Cross-Origin Resource Sharing)**  
Security feature that controls which external domains can make API requests to the Frappe server.

**GraphQL**  
Query language for APIs that allows clients to request specific data fields (experimental in Frappe).

**JWT (JSON Web Token)**  
Compact token format used for secure transmission of authentication information between systems.

**OAuth 2.0**  
Industry-standard authorization protocol allowing secure third-party access without sharing passwords.

**Rate Limiting**  
Mechanism to control the number of API requests a client can make within a specific time window.

**REST API**  
Architectural style for web APIs using HTTP methods (GET, POST, PUT, DELETE) for resource manipulation.

**Webhook**  
HTTP callback that sends data to external systems when specific events occur in Frappe applications.

### Security Terms

**ABAC (Attribute-Based Access Control)**  
Advanced permission system that evaluates user, resource, and environment attributes for access decisions.

**Authentication Hook**  
Custom code that intercepts and modifies the standard user authentication process.

**Encryption at Rest**  
Data protection technique that encrypts stored data in databases and file systems.

**Field-Level Encryption**  
Security feature that encrypts specific sensitive fields within documents using field-specific keys.

**MFA (Multi-Factor Authentication)**  
Security system requiring multiple verification methods (password + token/SMS) for user access.

**RBAC (Role-Based Access Control)**  
Permission model where access rights are assigned to roles, and users are assigned to roles.

**Security Audit Trail**  
Comprehensive logging system that tracks all security-related events and permission changes.

**Session Hijacking Prevention**  
Security measures to prevent unauthorized access to user sessions through token validation.

**TOTP (Time-based One-Time Password)**  
Authentication method generating temporary codes that change every 30 seconds (Google Authenticator).

### Testing Framework Terms

**CI/CD Pipeline**  
Continuous Integration/Continuous Deployment automation for testing and deploying Frappe applications.

**Coverage Report**  
Analysis showing which parts of code are executed during testing to identify untested areas.

**Fixture Factory**  
Pattern for creating test data objects with randomized or predefined values for testing scenarios.

**Integration Test**  
Testing methodology that verifies interactions between different components or external systems.

**Mock Object**  
Test double that simulates external dependencies or complex objects during unit testing.

**Performance Testing**  
Testing approach that measures system response times, throughput, and resource usage under load.

**Test Database**  
Isolated database environment used exclusively for running automated tests without affecting production.

**Test Runner**  
System component that executes test suites and reports results, typically using pytest in Frappe.

### Database Advanced Terms

**Connection Pooling**  
Database optimization technique that maintains reusable database connections to improve performance.

**Database Sharding**  
Horizontal partitioning strategy that distributes data across multiple database instances.

**Index Optimization**  
Process of creating and tuning database indexes to improve query performance.

**Migration Script**  
Code that modifies database schema or data structure when applications are updated.

**Query Builder Pattern**  
Programming approach using PyPika to construct SQL queries programmatically with type safety.

**Query Cache**  
System that stores frequently executed query results in memory to reduce database load.

**Read Replica**  
Database copy optimized for read operations to improve performance and distribute load.

**Schema Migration**  
Process of evolving database structure while preserving existing data during application updates.

### App Development Terms

**App Hooks System**  
Framework mechanism allowing apps to extend or modify core functionality without changing source code.

**Custom Field Extension**  
System for adding fields to existing DocTypes without modifying the original DocType definition.

**Module Loading**  
Framework process that dynamically loads and initializes app modules and their components.

**Property Setter**  
Customization tool that overrides default DocType properties without creating custom apps.

### Production & Deployment Terms

**Auto-Scaling**  
System capability to automatically adjust server resources based on application load and performance metrics.

**Blue-Green Deployment**  
Deployment strategy using two identical production environments to minimize downtime during updates.

**Container Orchestration**  
Management of containerized applications using tools like Kubernetes for scaling and deployment.

**Docker Container**  
Lightweight, portable package containing Frappe application and all its dependencies.

**Health Check**  
Automated monitoring system that verifies application and service availability and performance.

**Load Balancer**  
Network component that distributes incoming requests across multiple server instances.

**Prometheus Metrics**  
Monitoring system that collects and stores time-series performance data from Frappe applications.

**Reverse Proxy**  
Server (typically Nginx) that forwards client requests to backend Frappe servers and handles SSL.

### Monitoring & Observability Terms

**Distributed Tracing**  
Monitoring technique that tracks requests across multiple services to identify performance bottlenecks.

**Grafana Dashboard**  
Visualization tool that displays metrics, logs, and performance data in customizable charts and graphs.

**Log Aggregation**  
Process of collecting logs from multiple servers and services into a centralized monitoring system.

**Metrics Collection**  
System for gathering quantitative data about application performance, usage, and system health.

**OpenTelemetry**  
Framework for collecting, processing, and exporting telemetry data (metrics, logs, traces).

**SLA (Service Level Agreement)**  
Performance standards and uptime guarantees for production Frappe applications.

### Framework Internals Terms

**Dependency Injection**  
Design pattern that provides dependencies to objects rather than having them create dependencies internally.

**Event-Driven Architecture**  
System design where components communicate through events and event handlers rather than direct calls.

**Framework Hooks**  
Extension points in the Frappe core that allow customization without modifying framework source code.

**Memory Pool**  
Memory management technique that pre-allocates memory blocks to improve performance for large datasets.

**Meta Programming**  
Programming technique where programs have the ability to treat other programs as their data.

**Service Mesh**  
Infrastructure layer that handles service-to-service communication in distributed applications.

**Thread Local Storage**  
Memory area where data is stored per thread, used by Frappe for request-specific data isolation.

---

## Framework-Specific Patterns

**get_doc Pattern**  
Standard method for retrieving documents: `frappe.get_doc("DocType", "name")`

**whitelist Pattern**  
Required decorator for HTTP-accessible methods: `@frappe.whitelist()`

**throw Pattern**  
Standard error handling: `frappe.throw(_("Error message"))`

**msgprint Pattern**  
User notification: `frappe.msgprint(_("Information message"))`

**enqueue Pattern**  
Background job creation: `frappe.enqueue(method="path.to.method", queue="default")`

**cache Pattern**  
Data caching: `frappe.cache.set_value("key", value, expires_in_sec=3600)`

**db Pattern**  
Database operations: `frappe.db.get_value("DocType", "name", "field")`

**API Pattern**  
REST endpoint creation: `@frappe.whitelist()` + `frappe.request.json`

**Webhook Pattern**  
Event callback: `frappe.call_hook("webhook", doc=doc, method=method)`

**Test Pattern**  
Unit testing: `class TestDocType(FrappeTestCase):`

**Migration Pattern**  
Schema updates: `frappe.reload_doc(module, doctype, doctype_name)`

**Background Job Pattern**  
Async processing: `frappe.enqueue(method, queue="long", timeout=3600)`

This comprehensive glossary covers all terminology from the complete Frappe Framework documentation set, including advanced concepts for production deployment, security implementation, testing frameworks, and framework internals.