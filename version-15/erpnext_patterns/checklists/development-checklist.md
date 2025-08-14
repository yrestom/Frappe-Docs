# Development Checklist - ERPNext Best Practices

*Comprehensive quality assurance checklist based on ERPNext codebase analysis*

## Overview

This checklist ensures your Frappe framework development follows ERPNext's proven patterns and maintains enterprise-grade quality standards. Each section includes specific criteria extracted from ERPNext's production codebase.

---

## 🏗️ Project Setup & Architecture

### ✅ App Structure
- [ ] **App folder structure follows Frappe conventions**
  - [ ] `hooks.py` properly configured with all required hooks
  - [ ] `modules.txt` includes all custom modules
  - [ ] `requirements.txt` with pinned dependencies
  - [ ] `README.md` with clear installation and usage instructions
  - [ ] `.gitignore` excludes build files, caches, and sensitive data

- [ ] **Module organization follows ERPNext patterns**
  - [ ] Each module in separate folder under app directory
  - [ ] `__init__.py` files in all module directories
  - [ ] Logical grouping of related functionality

- [ ] **License and copyright headers**
  - [ ] All Python files include proper copyright headers
  - [ ] License file present and referenced correctly
  - [ ] Contributor guidelines documented

### ✅ Configuration Management
- [ ] **Hooks configuration complete**
  - [ ] Document events properly registered
  - [ ] Scheduled jobs configured with appropriate frequency
  - [ ] Website context processors if web views needed
  - [ ] Permission query conditions implemented
  - [ ] Boot session data configured if needed

- [ ] **Module configuration**
  - [ ] Module descriptions and icons set
  - [ ] Module dependencies properly defined
  - [ ] Color scheme and branding consistent

---

## 📄 DocType Development

### ✅ DocType Definition
- [ ] **Naming and structure**
  - [ ] DocType name follows CamelCase convention
  - [ ] Appropriate naming rule selected (Auto naming, Field, Prompt, etc.)
  - [ ] Title field set for better UX
  - [ ] Sort field and order configured appropriately

- [ ] **Field design follows ERPNext patterns**
  - [ ] Field names use snake_case convention
  - [ ] Required fields marked appropriately
  - [ ] Field types match data requirements
  - [ ] Options for Select fields complete and logical
  - [ ] Link fields point to correct DocTypes
  - [ ] Fetch from fields configured for derived data

- [ ] **Field organization and UX**
  - [ ] Related fields grouped in sections
  - [ ] Column breaks used appropriately for layout
  - [ ] Field order logical from user perspective
  - [ ] Print hide settings configured correctly
  - [ ] Read-only fields marked where appropriate

- [ ] **Child table design**
  - [ ] Child DocTypes created for repeating data
  - [ ] Parent-child relationships properly defined
  - [ ] Child table fields include required validations
  - [ ] Editable grid settings configured

### ✅ Field Configuration
- [ ] **Data validation**
  - [ ] Email fields have Email option set
  - [ ] Currency fields linked to appropriate currency field
  - [ ] Date fields have proper default values
  - [ ] Precision set for Float/Currency fields
  - [ ] Length limits set for Data fields

- [ ] **UI/UX configuration**
  - [ ] In list view fields selected strategically
  - [ ] In global search enabled for searchable fields
  - [ ] In standard filter enabled for commonly filtered fields
  - [ ] Bold formatting applied to important fields
  - [ ] Depends on conditions used for conditional display

- [ ] **Permissions and security**
  - [ ] No copy fields identified and marked
  - [ ] Set only once used for immutable fields
  - [ ] Ignore user permissions configured where needed
  - [ ] Hidden fields protected from client access

### ✅ DocType Settings
- [ ] **Submission and workflow**
  - [ ] Is submittable set correctly based on business process
  - [ ] Track changes enabled for audit requirements
  - [ ] Track seen configured for activity tracking
  - [ ] Allow events in timeline for communication history

- [ ] **Performance optimization**
  - [ ] Engine set to InnoDB for complex queries
  - [ ] Indexing considered for frequently searched fields
  - [ ] Max attachments limit set appropriately
  - [ ] Quick entry enabled for frequently created docs

---

## 🐍 Python Controller Development

### ✅ Code Structure and Organization
- [ ] **Class structure**
  - [ ] Controller inherits from appropriate base class
  - [ ] Auto-generated type hints preserved and maintained
  - [ ] Class docstring explains purpose and key methods
  - [ ] Methods organized logically (lifecycle, validation, calculation)

- [ ] **Import statements**
  - [ ] Standard library imports first
  - [ ] Third-party imports second
  - [ ] Frappe/ERPNext imports third
  - [ ] Local imports last
  - [ ] No unused imports
  - [ ] Imports sorted alphabetically within groups

### ✅ Validation Logic
- [ ] **Core validation methods**
  - [ ] `validate()` method implements all business rules
  - [ ] `before_save()` used for data preparation
  - [ ] `on_update()` handles post-save operations
  - [ ] Error messages use frappe.throw() with translatable strings

- [ ] **Data validation**
  - [ ] Required field validation beyond DocType level
  - [ ] Business rule validation implemented
  - [ ] Cross-field validation logic included
  - [ ] Duplicate prevention checks where needed
  - [ ] Date range validations implemented

- [ ] **Submission workflow**
  - [ ] `before_submit()` contains pre-submission checks
  - [ ] `on_submit()` handles status updates and integrations
  - [ ] `before_cancel()` validates cancellation eligibility
  - [ ] `on_cancel()` reverses submission effects

### ✅ Business Logic Implementation
- [ ] **Calculation methods**
  - [ ] Mathematical calculations use `flt()` for precision
  - [ ] Totals calculation follows ERPNext patterns
  - [ ] Tax calculations implemented correctly
  - [ ] Currency conversions handled properly

- [ ] **Status management**
  - [ ] Status field updates based on business rules
  - [ ] Status transitions validated
  - [ ] Workflow integration if applicable
  - [ ] Status changes logged appropriately

- [ ] **Integration handling**
  - [ ] GL entries created for financial documents
  - [ ] Stock ledger entries for inventory documents
  - [ ] Linked document updates implemented
  - [ ] Third-party system integrations secured

### ✅ Error Handling and Logging
- [ ] **Exception handling**
  - [ ] Try-catch blocks for external API calls
  - [ ] Database operation error handling
  - [ ] Graceful degradation for non-critical failures
  - [ ] Error logging for debugging

- [ ] **User feedback**
  - [ ] Clear error messages for validation failures
  - [ ] Success messages for important operations
  - [ ] Progress indicators for long operations
  - [ ] Warning messages for potential issues

---

## 🖥️ Client-Side JavaScript Development

### ✅ Form Script Organization
- [ ] **File structure**
  - [ ] JavaScript files named consistently with DocType
  - [ ] Code organized in logical functions
  - [ ] Event handlers grouped by trigger type
  - [ ] Utility functions separated from event handlers

- [ ] **Event handling**
  - [ ] `refresh` event configures form state
  - [ ] `setup` event initializes form-level settings
  - [ ] Field change events handle dependent field updates
  - [ ] Child table events manage row-level operations

### ✅ UI/UX Enhancement
- [ ] **Custom buttons and actions**
  - [ ] Custom buttons added based on document status
  - [ ] Button groups used for related actions
  - [ ] Button permissions checked before display
  - [ ] Button click handlers properly implemented

- [ ] **Form behavior**
  - [ ] Conditional field display/hide logic
  - [ ] Dynamic field property changes
  - [ ] Auto-fetch functionality for related data
  - [ ] Client-side validation for immediate feedback

- [ ] **Data visualization**
  - [ ] Indicators used for status visualization
  - [ ] Dashboard integration configured
  - [ ] Chart implementations optimized
  - [ ] Print format customizations applied

### ✅ Performance and User Experience
- [ ] **Client-side optimization**
  - [ ] Minimal API calls from client side
  - [ ] Caching used for frequently accessed data
  - [ ] Debouncing applied to search operations
  - [ ] Loading states shown for async operations

- [ ] **Mobile responsiveness**
  - [ ] Form layouts work on mobile devices
  - [ ] Touch-friendly button sizes
  - [ ] Scrolling behavior optimized
  - [ ] Essential fields prioritized in mobile view

---

## 🧪 Testing and Quality Assurance

### ✅ Test Coverage
- [ ] **Unit tests**
  - [ ] All validation methods have unit tests
  - [ ] Calculation methods tested with edge cases
  - [ ] Error conditions tested and verified
  - [ ] Test data setup and teardown implemented

- [ ] **Integration tests**
  - [ ] Document creation/submission flow tested
  - [ ] Cross-document integrations verified
  - [ ] API endpoints tested for all scenarios
  - [ ] Permission scenarios covered

### ✅ Code Quality
- [ ] **Code standards**
  - [ ] PEP 8 compliance for Python code
  - [ ] ESLint rules followed for JavaScript
  - [ ] No hardcoded strings (use translation functions)
  - [ ] Code comments explain complex business logic

- [ ] **Security considerations**
  - [ ] Input sanitization implemented
  - [ ] SQL injection prevention verified
  - [ ] Cross-site scripting (XSS) protection
  - [ ] Permission checks in all API methods

---

## 🔧 Configuration and Settings

### ✅ System Configuration
- [ ] **Default values**
  - [ ] Appropriate defaults set for new installations
  - [ ] Company-specific defaults configured
  - [ ] User preference defaults established
  - [ ] Regional defaults supported

- [ ] **Integration settings**
  - [ ] Email settings configured and tested
  - [ ] Print settings optimized
  - [ ] Notification settings functional
  - [ ] Backup settings configured

### ✅ Custom Fields and Properties
- [ ] **Field customization**
  - [ ] Custom fields added through proper migration
  - [ ] Field properties modified systematically
  - [ ] Custom field permissions set correctly
  - [ ] Translation keys added for custom labels

---

## 📊 Performance and Scalability

### ✅ Database Optimization
- [ ] **Query optimization**
  - [ ] Database queries use indexes effectively
  - [ ] Pagination implemented for large datasets
  - [ ] Filtering options provided for list views
  - [ ] Unnecessary data fetching avoided

- [ ] **Caching strategy**
  - [ ] Frequently accessed data cached appropriately
  - [ ] Cache invalidation logic implemented
  - [ ] Redis usage optimized
  - [ ] Memory usage monitored

### ✅ Background Processing
- [ ] **Async operations**
  - [ ] Long-running processes moved to background jobs
  - [ ] Job status tracking implemented
  - [ ] Error handling for background jobs
  - [ ] Job retry logic configured

---

## 🔒 Security and Permissions

### ✅ Access Control
- [ ] **Role-based permissions**
  - [ ] Roles defined based on business requirements
  - [ ] Permission levels set appropriately (read, write, create, delete)
  - [ ] Document-level permissions implemented
  - [ ] Field-level permissions configured where needed

- [ ] **Data security**
  - [ ] Sensitive data protected appropriately
  - [ ] Audit trail maintained for critical operations
  - [ ] Data export/import restrictions implemented
  - [ ] Personal data handling compliant with regulations

### ✅ API Security
- [ ] **Authentication and authorization**
  - [ ] API endpoints protected with authentication
  - [ ] Rate limiting implemented for public APIs
  - [ ] CORS settings configured securely
  - [ ] API key management implemented

---

## 📱 Web and Mobile

### ✅ Web Views
- [ ] **Public web pages**
  - [ ] Templates follow Frappe web framework patterns
  - [ ] SEO optimization implemented
  - [ ] Mobile-responsive design
  - [ ] Loading performance optimized

- [ ] **Portal integration**
  - [ ] Customer/supplier portal access configured
  - [ ] Portal permissions set correctly
  - [ ] Portal page navigation intuitive
  - [ ] Portal-specific functionality working

---

## 🚀 Deployment and Production

### ✅ Production Readiness
- [ ] **Configuration management**
  - [ ] Environment-specific settings externalized
  - [ ] Secrets management implemented
  - [ ] Database migration scripts tested
  - [ ] Rollback procedures documented

- [ ] **Monitoring and logging**
  - [ ] Application logging configured
  - [ ] Error monitoring set up
  - [ ] Performance monitoring enabled
  - [ ] Health check endpoints implemented

### ✅ Documentation
- [ ] **User documentation**
  - [ ] User manual created for new features
  - [ ] Administrator guide updated
  - [ ] API documentation generated
  - [ ] Installation instructions verified

- [ ] **Developer documentation**
  - [ ] Code documentation updated
  - [ ] Architecture decisions documented
  - [ ] Database schema documented
  - [ ] Troubleshooting guide maintained

---

## ✅ Final Review Checklist

### Pre-Deployment Verification
- [ ] **Functionality testing**
  - [ ] All user stories tested and verified
  - [ ] Edge cases handled appropriately
  - [ ] Error scenarios tested
  - [ ] Performance requirements met

- [ ] **Code review**
  - [ ] Peer code review completed
  - [ ] Security review conducted
  - [ ] Documentation review completed
  - [ ] Test coverage review passed

- [ ] **Deployment preparation**
  - [ ] Migration scripts tested on staging
  - [ ] Backup procedures verified
  - [ ] Rollback plan prepared
  - [ ] Go-live checklist completed

---

## 📋 Summary

This checklist represents the distilled wisdom from ERPNext's production codebase, covering every aspect of enterprise-grade Frappe framework development. Use it systematically to ensure your applications meet the highest standards of quality, security, and maintainability.

**Key Success Metrics:**
- ✅ 100% checklist completion before deployment
- ✅ Zero critical security vulnerabilities
- ✅ 90%+ test coverage for core functionality
- ✅ Sub-2 second response times for key operations
- ✅ Mobile-responsive design across all features

**Remember:** These patterns have been battle-tested in thousands of ERPNext installations worldwide. Following them ensures your applications will scale, perform, and maintain the same level of enterprise reliability that ERPNext is known for.