# Frappe Framework - Quick Reference Guide

> **Essential commands, patterns, and code snippets for rapid development**

## Table of Contents

- [Bench Commands](#bench-commands)
- [Python API Quick Reference](#python-api-quick-reference)
- [JavaScript API Quick Reference](#javascript-api-quick-reference)
- [Database Operations](#database-operations)
- [Common Patterns](#common-patterns)
- [Form Scripts Cheat Sheet](#form-scripts-cheat-sheet)
- [API Development](#api-development)
- [Error Handling](#error-handling)
- [File Operations](#file-operations)
- [Utilities & Helpers](#utilities--helpers)

## Bench Commands

### Development Commands
```bash
# Create new site
bench new-site site_name
bench use site_name

# App management
bench new-app app_name
bench install-app app_name
bench uninstall-app app_name

# Development server
bench start                    # Start all services
bench start --no-dev          # Production mode
bench start --only-web        # Only web server

# Database operations
bench migrate                  # Run database migrations
bench backup                  # Create backup
bench restore backup_file     # Restore from backup
bench mariadb                 # Access database shell

# Cache operations
bench clear-cache             # Clear all caches
bench clear-website-cache     # Clear website cache
bench build                   # Build assets

# Site operations
bench console                 # Python console
bench execute "python_code"   # Execute Python
bench set-config key value    # Set configuration
bench doctor                  # Diagnose issues
```

### Production Commands
```bash
# Setup production
sudo bench setup production user

# SSL setup
sudo bench setup lets-encrypt site_name

# Supervisor management
sudo bench setup supervisor
sudo supervisorctl restart all

# Nginx configuration
sudo bench setup nginx
sudo service nginx reload

# Update operations
bench update                  # Update all apps
bench update --patch         # Patch updates only
bench migrate-to version     # Migrate to specific version
```

## Python API Quick Reference

### Document Operations
```python
# Create new document
doc = frappe.new_doc("DocType Name")
doc.field_name = "value"
doc.insert()

# Get existing document
doc = frappe.get_doc("DocType", "document_name")
doc = frappe.get_doc({"doctype": "DocType", "field": "value"})

# Update document
doc.field_name = "new_value"
doc.save()

# Delete document
doc.delete()
frappe.delete_doc("DocType", "document_name")

# Submit/Cancel (for submittable docs)
doc.submit()
doc.cancel()
```

### Database Queries
```python
# Get list of documents
docs = frappe.get_all("DocType", 
    filters={"field": "value"},
    fields=["field1", "field2"],
    order_by="creation desc",
    limit_page_length=20
)

# Get single value
value = frappe.db.get_value("DocType", "document_name", "field_name")
values = frappe.db.get_value("DocType", "document_name", ["field1", "field2"])

# Set single value (bypass validation)
frappe.db.set_value("DocType", "document_name", "field_name", "value")

# Check existence
exists = frappe.db.exists("DocType", "document_name")
exists = frappe.db.exists("DocType", {"field": "value"})

# Count documents
count = frappe.db.count("DocType", {"field": "value"})

# Raw SQL
results = frappe.db.sql("SELECT * FROM `tabDocType` WHERE field = %s", ("value",))
results = frappe.db.sql("SELECT * FROM `tabDocType` WHERE field = %(field)s", 
    {"field": "value"}, as_dict=True)
```

### Common Patterns
```python
# Error handling
try:
    doc = frappe.get_doc("DocType", "name")
    doc.save()
except frappe.DoesNotExistError:
    frappe.throw("Document not found")
except frappe.ValidationError as e:
    frappe.throw(str(e))

# Permissions
if frappe.has_permission("DocType", "read", "document_name"):
    # User has read permission

# Session info
user = frappe.session.user
roles = frappe.get_roles()
user_info = frappe.get_user()

# Translations
message = _("Hello World")  # Translatable string
frappe.msgprint(_("Document saved successfully"))

# Background jobs
frappe.enqueue(
    method="module.function",
    queue="default",
    timeout=300,
    arg1="value1",
    arg2="value2"
)

# Cache operations
frappe.cache.set_value("key", "value", expires_in_sec=3600)
value = frappe.cache.get_value("key")
frappe.cache.delete_value("key")

# Email
frappe.sendmail(
    recipients=["user@example.com"],
    subject="Subject",
    message="Message",
    template="template_name",
    args={"name": "value"}
)
```

## JavaScript API Quick Reference

### Form Events
```javascript
frappe.ui.form.on('DocType', {
    setup: function(frm) {},           // Form setup
    onload: function(frm) {},          // Every load
    refresh: function(frm) {},         // After load/save
    validate: function(frm) {},        // Before save
    before_save: function(frm) {},     // Before save
    after_save: function(frm) {},      // After save
    
    // Field events
    field_name: function(frm) {},      // When field changes
    
    // Custom methods
    custom_method: function(frm) {}
});

// Child table events
frappe.ui.form.on('Child DocType', {
    child_table_add: function(frm, cdt, cdn) {},
    field_name: function(frm, cdt, cdn) {},
    child_table_remove: function(frm, cdt, cdn) {}
});
```

### Form Manipulation
```javascript
// Field operations
frm.set_value('field_name', 'value');
frm.get_value('field_name');
frm.toggle_display('field_name', condition);
frm.toggle_enable('field_name', condition);
frm.toggle_reqd('field_name', condition);

// Field properties
frm.set_df_property('field_name', 'read_only', 1);
frm.set_df_property('field_name', 'options', 'new_options');
frm.set_df_property('field_name', 'description', 'Help text');

// Custom buttons
frm.add_custom_button(__('Button'), function() {
    // Action
}, __('Group'));

// Queries
frm.set_query('link_field', function() {
    return {
        filters: {
            'status': 'Active'
        }
    };
});
```

### API Calls
```javascript
// Basic call
frappe.call({
    method: 'module.api.method_name',
    args: {
        'arg1': 'value1',
        'arg2': 'value2'
    },
    callback: function(r) {
        if (r.message) {
            console.log(r.message);
        }
    }
});

// With loading indicator
frappe.call({
    method: 'module.api.method_name',
    freeze: true,
    freeze_message: __('Processing...'),
    callback: function(r) {
        // Handle response
    }
});

// Async/await
async function callAPI() {
    try {
        const response = await frappe.call({
            method: 'module.api.method_name',
            args: { param: 'value' }
        });
        return response.message;
    } catch (error) {
        frappe.msgprint(__('API call failed'));
    }
}
```

### UI Components
```javascript
// Messages
frappe.msgprint(__('Message'));
frappe.throw(__('Error message'));
frappe.show_alert({
    message: __('Alert message'),
    indicator: 'green'
});

// Confirm dialog
frappe.confirm(__('Are you sure?'), function() {
    // Yes action
}, function() {
    // No action
});

// Progress indicator
frappe.show_progress(__('Loading'), 50, 100, 'Please wait...');
frappe.hide_progress();

// Custom dialog
const dialog = new frappe.ui.Dialog({
    title: __('Dialog Title'),
    fields: [
        {
            fieldname: 'field1',
            fieldtype: 'Data',
            label: __('Field 1'),
            reqd: 1
        }
    ],
    primary_action_label: __('Save'),
    primary_action: function(values) {
        console.log(values);
        dialog.hide();
    }
});
dialog.show();
```

## Database Operations

### Query Builder
```python
# Using Frappe Query Builder
from frappe.query_builder import DocType

Member = DocType('Library Member')
Issue = DocType('Book Issue')

# Simple select
members = (
    frappe.qb.from_(Member)
    .select(Member.name, Member.member_name)
    .where(Member.status == 'Active')
    .orderby(Member.member_name)
).run(as_dict=True)

# Join query
results = (
    frappe.qb.from_(Member)
    .join(Issue).on(Member.name == Issue.member)
    .select(Member.member_name, Issue.book, Issue.issue_date)
    .where(Issue.status == 'Issued')
).run(as_dict=True)

# Aggregation
stats = (
    frappe.qb.from_(Issue)
    .select(
        Issue.member,
        frappe.qb.functions.Count('*').as_('total_issues'),
        frappe.qb.functions.Max(Issue.issue_date).as_('last_issue')
    )
    .groupby(Issue.member)
).run(as_dict=True)
```

### Filters & Conditions
```python
# Filter operators
filters = {
    'field': 'value',                    # Equals
    'field': ['!=', 'value'],           # Not equals
    'field': ['>', 100],                # Greater than
    'field': ['>=', 100],               # Greater than or equal
    'field': ['<', 100],                # Less than
    'field': ['<=', 100],               # Less than or equal
    'field': ['like', '%text%'],        # LIKE pattern
    'field': ['not like', '%text%'],    # NOT LIKE
    'field': ['in', ['val1', 'val2']],  # IN list
    'field': ['not in', ['val1']],      # NOT IN list
    'field': ['between', [10, 20]],     # BETWEEN values
    'field': ['is', 'set'],             # IS NOT NULL
    'field': ['is', 'not set']          # IS NULL
}

# OR filters
docs = frappe.get_all('DocType',
    filters={'status': 'Active'},
    or_filters=[
        {'type': 'Premium'},
        {'priority': 'High'}
    ]
)
```

## Common Patterns

### Document Lifecycle
```python
class MyDocType(Document):
    def validate(self):
        self.validate_data()
        self.calculate_totals()
    
    def before_save(self):
        self.set_title()
        self.update_status()
    
    def after_insert(self):
        self.create_related_records()
        self.send_notifications()
    
    def on_submit(self):
        self.update_ledger()
        self.create_journal_entry()
    
    def on_cancel(self):
        self.reverse_transactions()
        self.cancel_related_docs()
```

### Error Handling
```python
# Custom exceptions
class CustomError(frappe.ValidationError):
    pass

# Throwing errors
frappe.throw(_('Error message'))
frappe.throw(_('Error with format {0}').format(value))
frappe.throw('Error', CustomError)

# Validation errors
if not self.email:
    frappe.throw(_('Email is required'))

if self.amount <= 0:
    frappe.throw(_('Amount must be positive'))
```

### Hooks Configuration
```python
# hooks.py
app_name = "my_app"

# Document Events
doc_events = {
    "User": {
        "before_insert": "my_app.hooks.user.before_user_insert"
    },
    "*": {
        "on_update": "my_app.hooks.common.log_update"
    }
}

# Page & Report Hooks
web_include_js = ["assets/my_app/js/global.js"]
web_include_css = ["assets/my_app/css/global.css"]

# Scheduled Events
scheduler_events = {
    "daily": [
        "my_app.tasks.daily_cleanup"
    ],
    "cron": {
        "0 2 * * *": [  # Daily at 2 AM
            "my_app.tasks.backup_data"
        ]
    }
}

# Fixtures
fixtures = [
    "Role",
    {"dt": "Custom Field", "filters": [["fieldname", "=", "my_custom_field"]]}
]
```

## Form Scripts Cheat Sheet

### Field Manipulation
```javascript
// Show/Hide fields
frm.toggle_display(['field1', 'field2'], condition);
frm.set_df_property('field', 'hidden', 1);

// Enable/Disable fields
frm.toggle_enable('field', condition);
frm.set_df_property('field', 'read_only', 1);

// Required fields
frm.toggle_reqd('field', condition);

// Field options
frm.set_df_property('select_field', 'options', 'Option 1\nOption 2');

// Field queries
frm.set_query('link_field', function() {
    return {
        query: 'module.queries.custom_query',
        filters: {'status': 'Active'}
    };
});
```

### Child Table Operations
```javascript
// Add row
let row = frm.add_child('child_table');
row.field_name = 'value';
frm.refresh_field('child_table');

// Clear table
frm.clear_table('child_table');
frm.refresh_field('child_table');

// Get table data
let table_data = frm.doc.child_table || [];

// Remove specific row
frappe.model.remove_from_locals('Child DocType', cdn);
frm.refresh_field('child_table');
```

### Calculations
```javascript
// Calculate totals
frappe.ui.form.on('Item', {
    qty: function(frm, cdt, cdn) {
        calculate_row_total(frm, cdt, cdn);
    },
    rate: function(frm, cdt, cdn) {
        calculate_row_total(frm, cdt, cdn);
    }
});

function calculate_row_total(frm, cdt, cdn) {
    let row = locals[cdt][cdn];
    row.amount = (row.qty || 0) * (row.rate || 0);
    frm.refresh_field('items');
    
    // Calculate grand total
    let total = 0;
    (frm.doc.items || []).forEach(item => {
        total += item.amount || 0;
    });
    frm.set_value('total_amount', total);
}
```

## API Development

### Whitelisted Methods
```python
# API endpoint
@frappe.whitelist()
def get_data(doctype, name):
    doc = frappe.get_doc(doctype, name)
    return {
        'name': doc.name,
        'data': doc.as_dict()
    }

# Guest access
@frappe.whitelist(allow_guest=True)
def public_api():
    return {'message': 'Public data'}

# Rate limiting
@frappe.whitelist()
@frappe.rate_limit(limit=10, window=60)  # 10 calls per minute
def limited_api():
    return {'status': 'success'}
```

### REST API Usage
```javascript
// GET request
fetch('/api/resource/DocType/document_name')
    .then(response => response.json())
    .then(data => console.log(data));

// POST request
fetch('/api/resource/DocType', {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json',
        'X-Frappe-CSRF-Token': frappe.csrf_token
    },
    body: JSON.stringify({
        field1: 'value1',
        field2: 'value2'
    })
})
.then(response => response.json())
.then(data => console.log(data));

// Custom method
fetch('/api/method/my_app.api.custom_method', {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json',
        'X-Frappe-CSRF-Token': frappe.csrf_token
    },
    body: JSON.stringify({
        arg1: 'value1',
        arg2: 'value2'
    })
})
.then(response => response.json())
.then(data => console.log(data));
```

## Error Handling

### Python Error Patterns
```python
try:
    doc = frappe.get_doc('DocType', 'name')
    doc.save()
except frappe.DoesNotExistError:
    frappe.throw(_('Document does not exist'))
except frappe.ValidationError as e:
    frappe.throw(_('Validation failed: {0}').format(str(e)))
except frappe.PermissionError:
    frappe.throw(_('Insufficient permissions'))
except Exception as e:
    frappe.log_error(f'Unexpected error: {str(e)}')
    frappe.throw(_('An unexpected error occurred'))
```

### JavaScript Error Patterns
```javascript
try {
    // Risky operation
    let result = some_operation();
} catch (error) {
    console.error('Error:', error);
    frappe.msgprint(__('Operation failed: {0}', [error.message]));
}

// API error handling
frappe.call({
    method: 'method_name',
    args: {},
    callback: function(r) {
        if (r.message) {
            // Success
        }
    },
    error: function(r) {
        frappe.msgprint(__('API call failed'));
    }
});
```

## File Operations

### File Upload & Management
```python
# Save file
from frappe.utils.file_manager import save_file

file_doc = save_file(
    filename='document.pdf',
    content=file_content,
    dt='DocType',
    dn='document_name',
    is_private=1
)

# Get file content
import frappe.utils.file_manager as fm
content = fm.get_file_content('file_url')

# Delete file
frappe.delete_doc('File', file_name)
```

### File Handling in JavaScript
```javascript
// File upload
frm.attachments.upload_file(function(attachment) {
    frm.set_value('attachment', attachment.file_url);
});

// Download file
window.open('/api/method/frappe.utils.print_format.download_pdf', {
    method: 'POST',
    args: {
        doctype: frm.doc.doctype,
        name: frm.doc.name,
        format: 'Standard'
    }
});
```

## Utilities & Helpers

### Date & Time Operations
```python
from frappe.utils import nowdate, now, getdate, get_datetime, add_days, date_diff

# Current date/time
today = nowdate()                    # Current date
now_time = now()                     # Current datetime
current_dt = get_datetime()          # Datetime object

# Date manipulation
future_date = add_days(today, 30)    # Add 30 days
past_date = add_days(today, -30)     # Subtract 30 days
diff = date_diff(future_date, today) # Difference in days

# Parsing
date_obj = getdate('2024-03-15')     # Parse date string
dt_obj = get_datetime('2024-03-15 10:30:00')
```

### String & Data Utilities
```python
from frappe.utils import cint, flt, cstr, sbool

# Type conversion
integer_val = cint('123')            # Convert to int
float_val = flt('123.45')           # Convert to float
string_val = cstr(123)              # Convert to string
bool_val = sbool('1')               # Convert to boolean

# Formatting
formatted_currency = frappe.format(1234.56, {'fieldtype': 'Currency'})
formatted_date = frappe.format(today, {'fieldtype': 'Date'})
```

### JavaScript Utilities
```javascript
// Date utilities
frappe.datetime.get_today();                    // Current date
frappe.datetime.add_days('2024-03-15', 30);    // Add days
frappe.datetime.get_diff('2024-03-20', '2024-03-15'); // Date difference

// Format utilities
frappe.format(1234.56, {fieldtype: 'Currency'});
frappe.format('2024-03-15', {fieldtype: 'Date'});

// Validation
frappe.utils.validate_type('email@domain.com', 'email'); // true/false
```

## Advanced API Development

### OAuth 2.0 Implementation
```python
# OAuth server setup
@frappe.whitelist(allow_guest=True)
def oauth_authorization():
    client_id = frappe.request.json.get("client_id")
    redirect_uri = frappe.request.json.get("redirect_uri")
    
    # Validate client
    client = frappe.get_doc("OAuth Client", client_id)
    if client.redirect_uri != redirect_uri:
        frappe.throw("Invalid redirect URI")
    
    # Generate authorization code
    auth_code = frappe.generate_hash(length=32)
    frappe.cache.set_value(f"oauth_code_{auth_code}", {
        "client_id": client_id,
        "user": frappe.session.user
    }, expires_in_sec=600)
    
    return {"authorization_code": auth_code}

# Token exchange
@frappe.whitelist(allow_guest=True)
def oauth_token():
    auth_code = frappe.request.json.get("code")
    cached_data = frappe.cache.get_value(f"oauth_code_{auth_code}")
    
    if not cached_data:
        frappe.throw("Invalid authorization code")
    
    # Generate access token
    access_token = frappe.generate_hash(length=32)
    frappe.cache.set_value(f"oauth_token_{access_token}", {
        "user": cached_data["user"],
        "client_id": cached_data["client_id"]
    }, expires_in_sec=3600)
    
    return {"access_token": access_token, "token_type": "Bearer"}
```

### Webhook Implementation
```python
# Webhook configuration
def setup_webhooks():
    webhook_doc = frappe.get_doc({
        "doctype": "Webhook",
        "webhook_doctype": "Sales Invoice",
        "webhook_docevent": "on_submit",
        "request_url": "https://external-system.com/webhook",
        "request_structure": "JSON",
        "webhook_headers": [
            {
                "key": "Authorization",
                "value": "Bearer {api_token}"
            }
        ],
        "webhook_data": [
            {"fieldname": "name", "key": "invoice_id"},
            {"fieldname": "customer", "key": "customer_name"}
        ]
    })
    webhook_doc.insert()

# Custom webhook handler
@frappe.whitelist()
def handle_external_webhook():
    data = frappe.request.json
    
    # Verify webhook signature
    signature = frappe.request.headers.get("X-Webhook-Signature")
    if not verify_webhook_signature(data, signature):
        frappe.throw("Invalid webhook signature")
    
    # Process webhook data
    process_external_data(data)
    
    return {"status": "success"}
```

### API Rate Limiting
```python
# Custom rate limiter
class APIRateLimiter:
    def __init__(self, limit=100, window=3600):
        self.limit = limit
        self.window = window
        
    def is_allowed(self, user_id, endpoint):
        key = f"rate_limit:{user_id}:{endpoint}"
        current_count = frappe.cache.get_value(key) or 0
        
        if current_count >= self.limit:
            return False
            
        frappe.cache.set_value(key, current_count + 1, expires_in_sec=self.window)
        return True

# Rate limiting decorator
def rate_limit(limit=100, window=3600):
    def decorator(func):
        def wrapper(*args, **kwargs):
            limiter = APIRateLimiter(limit, window)
            user_id = frappe.session.user
            endpoint = func.__name__
            
            if not limiter.is_allowed(user_id, endpoint):
                frappe.throw("Rate limit exceeded", frappe.RateLimitExceededError)
                
            return func(*args, **kwargs)
        return wrapper
    return decorator

@frappe.whitelist()
@rate_limit(limit=10, window=60)
def limited_api_endpoint():
    return {"message": "API response"}
```

## Security Implementation

### Multi-Factor Authentication
```python
# TOTP setup
def setup_totp_for_user(user):
    import pyotp
    secret = pyotp.random_base32()
    totp = pyotp.TOTP(secret)
    
    # Store encrypted secret
    encrypted_secret = encrypt_data(secret)
    frappe.db.set_value("User", user, "totp_secret", encrypted_secret)
    
    # Generate QR code URI
    qr_uri = totp.provisioning_uri(
        name=user,
        issuer_name=frappe.local.site
    )
    
    return {
        "secret": secret,
        "qr_uri": qr_uri,
        "backup_codes": generate_backup_codes(user)
    }

# TOTP verification
def verify_totp(user, token):
    import pyotp
    encrypted_secret = frappe.db.get_value("User", user, "totp_secret")
    if not encrypted_secret:
        return False
        
    secret = decrypt_data(encrypted_secret)
    totp = pyotp.TOTP(secret)
    return totp.verify(token, valid_window=1)
```

### Field-Level Encryption
```python
# Encrypt sensitive fields
def encrypt_field_data(doctype, fieldname, data):
    from cryptography.fernet import Fernet
    
    # Get field-specific encryption key
    key = get_field_encryption_key(doctype, fieldname)
    cipher_suite = Fernet(key)
    
    return cipher_suite.encrypt(data.encode()).decode()

def decrypt_field_data(doctype, fieldname, encrypted_data):
    from cryptography.fernet import Fernet
    
    key = get_field_encryption_key(doctype, fieldname)
    cipher_suite = Fernet(key)
    
    return cipher_suite.decrypt(encrypted_data.encode()).decode()

# Custom encrypted field type
class EncryptedData:
    def __init__(self, doctype, fieldname):
        self.doctype = doctype
        self.fieldname = fieldname
        
    def encrypt(self, value):
        return encrypt_field_data(self.doctype, self.fieldname, value)
        
    def decrypt(self, encrypted_value):
        return decrypt_field_data(self.doctype, self.fieldname, encrypted_value)
```

### Role-Based Access Control
```python
# Advanced permission checking
def check_advanced_permission(doctype, name, permission_type, user=None):
    user = user or frappe.session.user
    
    # Check basic permissions
    if not frappe.has_permission(doctype, permission_type, name, user=user):
        return False
    
    # Check attribute-based permissions
    doc = frappe.get_doc(doctype, name)
    user_attributes = get_user_attributes(user)
    resource_attributes = get_resource_attributes(doc)
    
    return evaluate_abac_policy(user_attributes, resource_attributes, permission_type)

# Custom permission hook
def apply_user_permission_query(user, query, doctype):
    """Apply user-specific filters to queries"""
    if user == "Administrator":
        return query
        
    # Apply territory-based filtering
    user_territories = get_user_territories(user)
    if user_territories:
        query = query.where(DocType(doctype).territory.isin(user_territories))
        
    return query
```

## Testing Framework Utilities

### Test Data Factories
```python
# Test data factory
class TestDataFactory:
    @staticmethod
    def create_customer(name=None, **kwargs):
        customer_data = {
            "doctype": "Customer",
            "customer_name": name or frappe.generate_hash(length=8),
            "customer_type": "Individual",
            "customer_group": "Individual",
            "territory": "Rest Of The World"
        }
        customer_data.update(kwargs)
        
        customer = frappe.get_doc(customer_data)
        customer.insert(ignore_permissions=True)
        return customer
    
    @staticmethod
    def create_item(item_code=None, **kwargs):
        item_data = {
            "doctype": "Item",
            "item_code": item_code or frappe.generate_hash(length=8),
            "item_name": kwargs.get("item_name", "Test Item"),
            "item_group": "All Item Groups",
            "stock_uom": "Nos"
        }
        item_data.update(kwargs)
        
        item = frappe.get_doc(item_data)
        item.insert(ignore_permissions=True)
        return item

# Test fixture management
class TestFixtures:
    @classmethod
    def setup_test_data(cls):
        cls.customer = TestDataFactory.create_customer("Test Customer")
        cls.item = TestDataFactory.create_item("TEST001")
        
    @classmethod
    def teardown_test_data(cls):
        frappe.delete_doc("Customer", cls.customer.name)
        frappe.delete_doc("Item", cls.item.name)
```

### Performance Testing
```python
# Performance test utilities
import time
import functools

def performance_test(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        start_time = time.time()
        result = func(*args, **kwargs)
        execution_time = time.time() - start_time
        
        print(f"{func.__name__} executed in {execution_time:.4f} seconds")
        
        # Log to performance metrics
        frappe.local.performance_metrics = getattr(frappe.local, 'performance_metrics', [])
        frappe.local.performance_metrics.append({
            "function": func.__name__,
            "execution_time": execution_time,
            "timestamp": time.time()
        })
        
        return result
    return wrapper

# Load testing helper
def simulate_concurrent_requests(endpoint, num_requests=100, concurrency=10):
    import concurrent.futures
    import requests
    
    def make_request():
        response = requests.post(f"{frappe.utils.get_url()}{endpoint}")
        return response.status_code, response.elapsed.total_seconds()
    
    results = []
    with concurrent.futures.ThreadPoolExecutor(max_workers=concurrency) as executor:
        futures = [executor.submit(make_request) for _ in range(num_requests)]
        for future in concurrent.futures.as_completed(futures):
            results.append(future.result())
    
    return analyze_performance_results(results)
```

## Advanced Database Operations

### Query Builder Advanced Patterns
```python
# Complex joins and subqueries
def get_sales_analytics():
    Customer = frappe.qb.DocType('Customer')
    SalesInvoice = frappe.qb.DocType('Sales Invoice')
    SalesInvoiceItem = frappe.qb.DocType('Sales Invoice Item')
    
    # Subquery for customer totals
    customer_totals = (
        frappe.qb.from_(SalesInvoice)
        .select(
            SalesInvoice.customer,
            frappe.qb.functions.Sum(SalesInvoice.grand_total).as_('total_sales'),
            frappe.qb.functions.Count(SalesInvoice.name).as_('invoice_count')
        )
        .where(SalesInvoice.docstatus == 1)
        .groupby(SalesInvoice.customer)
    ).as_('customer_totals')
    
    # Main query with join to subquery
    return (
        frappe.qb.from_(Customer)
        .join(customer_totals).on(Customer.name == customer_totals.customer)
        .select(
            Customer.name,
            Customer.customer_name,
            customer_totals.total_sales,
            customer_totals.invoice_count
        )
        .where(customer_totals.total_sales > 10000)
        .orderby(customer_totals.total_sales, order=frappe.qb.Order.desc)
    ).run(as_dict=True)

# Window functions
def get_sales_ranking():
    SalesInvoice = frappe.qb.DocType('Sales Invoice')
    
    return (
        frappe.qb.from_(SalesInvoice)
        .select(
            SalesInvoice.customer,
            SalesInvoice.grand_total,
            frappe.qb.functions.RowNumber().over(
                SalesInvoice.customer
            ).orderby(SalesInvoice.grand_total, order=frappe.qb.Order.desc).as_('rank')
        )
        .where(SalesInvoice.docstatus == 1)
    ).run(as_dict=True)
```

### Database Optimization
```python
# Index creation and optimization
def optimize_database_performance():
    # Create composite index for common query patterns
    frappe.db.sql("""
        CREATE INDEX IF NOT EXISTS idx_sales_invoice_customer_date 
        ON `tabSales Invoice` (customer, posting_date)
    """)
    
    # Create index for filtered queries
    frappe.db.sql("""
        CREATE INDEX IF NOT EXISTS idx_item_group_disabled 
        ON `tabItem` (item_group, disabled)
    """)
    
    # Analyze table statistics
    frappe.db.sql("ANALYZE TABLE `tabSales Invoice`")

# Connection pooling configuration
def setup_connection_pool():
    import pymysql
    from pymysql.connections import Connection
    
    pool_config = {
        'host': frappe.conf.db_host,
        'user': frappe.conf.db_name,
        'password': frappe.conf.db_password,
        'database': frappe.conf.db_name,
        'max_connections': 20,
        'stale_timeout': 300
    }
    
    return pymysql.pool.Pool(**pool_config)
```

## Production Deployment Commands

### Docker Operations
```bash
# Build and deploy with Docker
docker build -t frappe-app:latest .
docker run -d --name frappe-container \
  -p 8000:8000 \
  -e FRAPPE_SITE_NAME=mysite.com \
  -e DB_HOST=db-server \
  -e REDIS_CACHE_URL=redis://redis-server:6379/0 \
  frappe-app:latest

# Docker Compose for development
# docker-compose.yml
version: '3.8'
services:
  frappe:
    build: .
    ports:
      - "8000:8000"
    environment:
      - FRAPPE_SITE_NAME=development.localhost
    depends_on:
      - mariadb
      - redis
  
  mariadb:
    image: mariadb:10.6
    environment:
      - MYSQL_ROOT_PASSWORD=frappe
    
  redis:
    image: redis:alpine

# Kubernetes deployment
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frappe-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frappe
  template:
    metadata:
      labels:
        app: frappe
    spec:
      containers:
      - name: frappe
        image: frappe-app:latest
        ports:
        - containerPort: 8000
        env:
        - name: FRAPPE_SITE_NAME
          value: "production.site.com"
EOF
```

### Monitoring Setup
```bash
# Prometheus monitoring
bench setup prometheus
bench config set_prometheus_config

# Grafana dashboard setup
bench setup grafana-dashboard

# Health check endpoint
curl -f http://localhost:8000/api/method/ping || exit 1

# Log monitoring with ELK stack
# filebeat.yml
filebeat.inputs:
- type: log
  paths:
    - /home/frappe/frappe-bench/logs/*.log
  fields:
    service: frappe
    
output.elasticsearch:
  hosts: ["elasticsearch:9200"]
```

## Troubleshooting Tools

### Diagnostic Commands
```bash
# Comprehensive system diagnosis
bench doctor --verbose

# Database connectivity test
bench execute "frappe.db.sql('SELECT 1')"

# Redis connectivity test
bench execute "frappe.cache.get_value('test')"

# Performance profiling
bench execute "
import cProfile
import frappe

def profile_function():
    # Your function to profile
    frappe.get_list('Item', limit=1000)

cProfile.run('profile_function()', 'profile.stats')
"

# Memory usage analysis
bench execute "
import psutil
import os

process = psutil.Process(os.getpid())
print(f'Memory usage: {process.memory_info().rss / 1024 / 1024:.2f} MB')
"

# Background job monitoring
bench show-pending-jobs
bench worker --queue default --burst  # Test worker
bench purge-jobs --queue default      # Clear stuck jobs
```

### Debug Utilities
```python
# Debug logging
def enable_debug_logging():
    import logging
    
    # Set up detailed logging
    logging.basicConfig(
        level=logging.DEBUG,
        format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
        handlers=[
            logging.FileHandler('/tmp/frappe_debug.log'),
            logging.StreamHandler()
        ]
    )
    
    # Enable SQL query logging
    frappe.conf.logging = 1
    frappe.local.dev_mode = 1

# Performance monitoring decorator
def monitor_performance(func):
    def wrapper(*args, **kwargs):
        import time
        start_time = time.time()
        
        # Monitor database queries
        initial_query_count = len(frappe.local.db._queries) if hasattr(frappe.local, 'db') else 0
        
        result = func(*args, **kwargs)
        
        execution_time = time.time() - start_time
        final_query_count = len(frappe.local.db._queries) if hasattr(frappe.local, 'db') else 0
        query_count = final_query_count - initial_query_count
        
        print(f"""
        Function: {func.__name__}
        Execution Time: {execution_time:.4f}s
        Database Queries: {query_count}
        """)
        
        return result
    return wrapper
```

## Framework Internals

### Custom Hook Implementation
```python
# Advanced hooks configuration
# hooks.py
app_name = "my_advanced_app"

# Document lifecycle hooks
doc_events = {
    "*": {
        "before_insert": "my_app.hooks.document.log_document_creation",
        "on_update": "my_app.hooks.document.track_changes"
    },
    "User": {
        "before_insert": "my_app.hooks.user.validate_user_creation",
        "after_insert": "my_app.hooks.user.setup_user_defaults"
    }
}

# Custom API endpoints
website_route_rules = [
    {"from_route": "/api/v2/<path:app_path>", "to_route": "my_app.api.v2_handler"},
]

# Background job hooks
scheduler_events = {
    "cron": {
        "0 2 * * *": [  # Daily at 2 AM
            "my_app.tasks.daily_maintenance"
        ],
        "*/15 * * * *": [  # Every 15 minutes
            "my_app.tasks.sync_external_data"
        ]
    }
}

# Email hooks
email_hooks = [
    "my_app.hooks.email.process_inbound_email"
]

# Permission query hooks
permission_query_conditions = {
    "Sales Invoice": "my_app.hooks.permissions.get_sales_invoice_conditions"
}
```

### Meta Programming Patterns
```python
# Dynamic DocType creation
def create_dynamic_doctype(name, fields):
    doctype_dict = {
        "doctype": "DocType",
        "name": name,
        "module": "Custom",
        "custom": 1,
        "fields": fields,
        "permissions": [
            {
                "role": "System Manager",
                "read": 1, "write": 1, "create": 1, "delete": 1
            }
        ]
    }
    
    doctype_doc = frappe.get_doc(doctype_dict)
    doctype_doc.insert()
    return doctype_doc

# Dynamic method injection
def add_method_to_doctype(doctype, method_name, method_func):
    # Get the DocType class
    doctype_class = frappe.get_doc_class(doctype)
    
    # Add the method to the class
    setattr(doctype_class, method_name, method_func)
    
    # Clear the cached class
    frappe.controllers.pop(doctype, None)

# Example usage
def custom_validation_method(self):
    if not self.custom_field:
        frappe.throw("Custom field is required")

add_method_to_doctype("Sales Invoice", "validate_custom_field", custom_validation_method)
```

---

This comprehensive quick reference now covers all advanced patterns and commands from the complete 12-file Frappe Framework documentation set. It includes production-ready examples for API development, security implementation, testing automation, database optimization, deployment strategies, troubleshooting tools, and framework internals.