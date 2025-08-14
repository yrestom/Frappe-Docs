# Troubleshooting Index - ERPNext Development

*Comprehensive problem-solution database based on ERPNext codebase analysis and common development challenges*

## 🔍 Quick Problem Finder

### Common Issues by Category
- 🏗️ [**DocType Issues**](#doctype-issues) - Field validation, naming, relationships
- 🐍 [**Python Controller Issues**](#python-controller-issues) - Validation, calculations, events
- 🖥️ [**JavaScript Issues**](#javascript-issues) - Client-side functionality, form behavior
- 🔒 [**Permission Issues**](#permission-issues) - Access control, role-based restrictions
- 📊 [**Performance Issues**](#performance-issues) - Slow queries, memory usage, optimization
- 🔄 [**Integration Issues**](#integration-issues) - API calls, third-party systems
- 🚀 [**Deployment Issues**](#deployment-issues) - Production problems, configuration

### Emergency Quick Fixes
- **Site won't start**: `bench restart`, check logs in `logs/`
- **Migration failed**: `bench migrate --skip-failing`  
- **Permission denied**: Check user roles and DocType permissions
- **Import errors**: Verify app installation and module imports
- **Database connection**: Check `site_config.json` credentials

---

## 🏗️ DocType Issues

### Problem: DocType Creation Fails
```
Error: Duplicate DocType name
```
**Root Cause**: DocType with same name already exists in database

**Solution**:
```bash
# Check existing DocTypes
bench execute "frappe.db.sql('SELECT name FROM tabDocType WHERE name LIKE \"%YourDocType%\"')"

# Delete if necessary (CAUTION: This removes all data)
bench execute "frappe.delete_doc('DocType', 'Your DocType Name', force=True)"
```

**Prevention**: Always check for existing DocTypes before creation

### Problem: Field Validation Not Working
```
Error: Field validations are bypassed
```
**Root Cause**: Validation not implemented in controller or validation logic incorrect

**ERPNext Pattern Solution**:
```python
# In your DocType controller - follow Sales Invoice pattern
def validate(self):
    # Basic validation
    self.validate_mandatory_fields()
    
    # Business rule validation  
    self.validate_business_rules()
    
    # Cross-field validation
    self.validate_field_relationships()

def validate_business_rules(self):
    """Implement specific business rules"""
    if self.status == "Completed" and not self.completion_date:
        frappe.throw(_("Completion Date is mandatory when status is Completed"))
```

### Problem: Naming Series Not Working
```
Error: Name (naming series) is mandatory
```
**Root Cause**: Naming series not configured properly

**ERPNext Pattern Solution**:
```python
# In DocType JSON
{
    "autoname": "naming_series:",
    "fields": [
        {
            "fieldname": "naming_series",
            "fieldtype": "Select", 
            "options": "PROJ-.YYYY.-\nTSK-.MM.-.YYYY.-",
            "default": "PROJ-.YYYY.-",
            "reqd": 1
        }
    ]
}

# In hooks.py (following ERPNext pattern)
doc_events = {
    "Your DocType": {
        "autoname": "your_app.utils.set_naming_series"
    }
}
```

### Problem: Child Table Not Updating Parent
```
Error: Parent totals not updating when child table changes
```
**Root Cause**: Parent calculation not triggered on child table updates

**ERPNext Pattern Solution**:
```python
# Child table controller (following Sales Invoice Item pattern)
class YourChildTable(Document):
    def validate(self):
        self.calculate_amount()
    
    def calculate_amount(self):
        self.amount = flt(self.qty) * flt(self.rate)

# Parent controller
def validate(self):
    self.calculate_totals()

def calculate_totals(self):
    self.total = sum(flt(item.amount) for item in self.get("items", []))
```

---

## 🐍 Python Controller Issues  

### Problem: Import Errors in Controllers
```
ImportError: cannot import name 'SomeClass'
```
**Root Cause**: Circular imports or incorrect import paths

**ERPNext Pattern Solution**:
```python
# ❌ Wrong - Circular import
from erpnext.accounts.doctype.sales_invoice.sales_invoice import SalesInvoice

# ✅ Correct - Dynamic import following ERPNext pattern
def get_linked_invoice(self):
    if frappe.db.exists("Sales Invoice", {"reference_name": self.name}):
        return frappe.get_doc("Sales Invoice", {"reference_name": self.name})
    return None

# ✅ Correct - Import inside method when needed
def process_payment(self):
    from erpnext.accounts.doctype.payment_entry.payment_entry import get_payment_entry
    return get_payment_entry(self.doctype, self.name)
```

### Problem: Calculation Precision Issues
```
Error: Calculation results in unexpected decimal places
```
**Root Cause**: Float precision not handled properly

**ERPNext Pattern Solution**:
```python
from frappe.utils import flt, cint

# ✅ Always use flt() for financial calculations
def calculate_total(self):
    total = 0.0
    for item in self.items:
        item.amount = flt(item.qty) * flt(item.rate)
        total += flt(item.amount)
    
    # Round to currency precision (following ERPNext pattern)
    self.total = flt(total, 2)  # 2 decimal places for currency

# ✅ For percentage calculations
def calculate_percentage(self):
    if self.base_amount:
        self.percentage = flt((self.current_amount / self.base_amount) * 100, 2)
```

### Problem: Document Status Not Updating
```
Error: Status field remains unchanged after submission
```
**Root Cause**: Status update not implemented in submission workflow

**ERPNext Pattern Solution**:
```python
# Following Sales Order status pattern
def on_submit(self):
    self.update_status()
    self.create_related_documents()

def on_cancel(self):
    self.update_status() 
    self.cancel_related_documents()

def update_status(self):
    """Update status based on docstatus"""
    if self.docstatus == 0:
        self.status = "Draft"
    elif self.docstatus == 1:
        self.status = self.get_submitted_status()
    elif self.docstatus == 2:
        self.status = "Cancelled"
    
    # Save without triggering validation again
    frappe.db.set_value(self.doctype, self.name, "status", self.status)

def get_submitted_status(self):
    """Override in subclasses for specific status logic"""
    return "Submitted"
```

### Problem: Permission Errors in Controller
```
PermissionError: Insufficient permissions
```
**Root Cause**: Insufficient permissions for operations within controller

**ERPNext Pattern Solution**:
```python
# ✅ Use ignore_permissions for system operations
def create_journal_entry(self):
    je = frappe.new_doc("Journal Entry")
    je.update({
        "voucher_type": "Journal Entry",
        "company": self.company
    })
    # System operation - bypass permissions
    je.insert(ignore_permissions=True)
    je.submit()

# ✅ Check permissions before operations
def update_linked_document(self, doctype, name):
    if frappe.has_permission(doctype, "write"):
        doc = frappe.get_doc(doctype, name)
        doc.update_status()
        doc.save()
    else:
        frappe.log_error(f"Permission denied for {doctype} {name}")
```

---

## 🖥️ JavaScript Issues

### Problem: Form Events Not Firing
```
Error: frappe.ui.form.on events not triggered
```
**Root Cause**: JavaScript not loaded or incorrect event syntax

**ERPNext Pattern Solution**:
```javascript
// ✅ Ensure proper structure following ERPNext patterns
frappe.ui.form.on('Your DocType', {
    // Form-level events
    refresh: function(frm) {
        // Always implement refresh for dynamic behavior
        frm.trigger('set_custom_buttons');
    },
    
    setup: function(frm) {
        // One-time form setup (queries, etc.)
        frm.set_query('customer', function() {
            return {
                filters: {
                    'disabled': 0
                }
            };
        });
    },
    
    // Field-level events  
    customer: function(frm) {
        if (frm.doc.customer) {
            frm.trigger('fetch_customer_details');
        }
    },
    
    fetch_customer_details: function(frm) {
        frappe.call({
            method: 'frappe.client.get_value',
            args: {
                doctype: 'Customer',
                name: frm.doc.customer,
                fieldname: ['customer_name', 'territory']
            },
            callback: function(r) {
                if (r.message) {
                    frm.set_value('customer_name', r.message.customer_name);
                    frm.set_value('territory', r.message.territory);
                }
            }
        });
    }
});
```

### Problem: Custom Buttons Not Appearing
```
Error: Custom buttons added but not visible
```
**Root Cause**: Button conditions not met or incorrect implementation

**ERPNext Pattern Solution**:
```javascript
// Following ERPNext button pattern (Sales Order style)
frappe.ui.form.on('Your DocType', {
    refresh: function(frm) {
        // Clear existing custom buttons
        frm.page.clear_inner_toolbar();
        
        // Add buttons based on document state
        if (frm.doc.docstatus === 1) {
            // Submitted document buttons
            frm.add_custom_button(__('Create Payment'), function() {
                frm.trigger('make_payment_entry');
            }, __('Create'));
            
            frm.add_custom_button(__('Create Invoice'), function() {
                frm.trigger('make_sales_invoice');
            }, __('Create'));
            
            // Make 'Create' group primary
            frm.page.set_inner_btn_group_as_primary(__('Create'));
        }
        
        if (frm.doc.status === 'Pending Approval') {
            frm.add_custom_button(__('Approve'), function() {
                frm.trigger('approve_document');
            }).addClass('btn-primary');
        }
    }
});
```

### Problem: Child Table Events Not Working
```
Error: Child table field changes not triggering calculations
```
**Root Cause**: Child table events not properly bound

**ERPNext Pattern Solution**:
```javascript
// Following Sales Invoice Item pattern
frappe.ui.form.on('Your Child DocType', {
    // Row-level events
    qty: function(frm, cdt, cdn) {
        calculate_item_amount(frm, cdt, cdn);
        frm.trigger('calculate_totals');
    },
    
    rate: function(frm, cdt, cdn) {
        calculate_item_amount(frm, cdt, cdn);
        frm.trigger('calculate_totals');
    },
    
    // Table-level events
    items_remove: function(frm) {
        frm.trigger('calculate_totals');
    },
    
    items_add: function(frm, cdt, cdn) {
        let row = locals[cdt][cdn];
        frappe.model.set_value(cdt, cdn, 'qty', 1);
        frappe.model.set_value(cdt, cdn, 'rate', 0);
    }
});

function calculate_item_amount(frm, cdt, cdn) {
    let row = locals[cdt][cdn];
    let amount = flt(row.qty) * flt(row.rate);
    frappe.model.set_value(cdt, cdn, 'amount', amount);
}

// Parent form totals calculation
frappe.ui.form.on('Your DocType', {
    calculate_totals: function(frm) {
        let total = 0;
        
        $.each(frm.doc.items || [], function(i, item) {
            total += flt(item.amount);
        });
        
        frm.set_value('total', total);
    }
});
```

---

## 🔒 Permission Issues

### Problem: Users Can't Access Documents
```
Error: You don't have permission to access this document
```
**Root Cause**: Role permissions not configured properly

**ERPNext Pattern Solution**:
```python
# In DocType permissions, follow ERPNext role patterns
{
    "permissions": [
        {
            "role": "Sales User",
            "read": 1,
            "write": 1,
            "create": 1,
            "delete": 0,
            "submit": 1,
            "cancel": 0,
            "amend": 1
        },
        {
            "role": "Sales Manager", 
            "read": 1,
            "write": 1,
            "create": 1,
            "delete": 1,
            "submit": 1,
            "cancel": 1,
            "amend": 1
        }
    ]
}

# Custom permission query conditions (following Customer pattern)
def get_permission_query_conditions(user):
    if not user:
        user = frappe.session.user
        
    if "Sales Manager" in frappe.get_roles(user):
        return None  # Full access
    
    # Restrict to user's territory
    territories = frappe.get_all("User Territory", 
                                filters={"parent": user}, 
                                pluck="territory")
    
    if territories:
        return f"territory in {tuple(territories)}"
    
    return "1=0"  # No access

# Document-level permissions
def has_permission(doc, ptype, user):
    if doc.owner == user:
        return True
        
    if "System Manager" in frappe.get_roles(user):
        return True
        
    # Custom business logic
    if ptype == "read" and doc.status == "Public":
        return True
        
    return False
```

### Problem: Field-Level Permissions Not Working  
```
Error: Sensitive fields visible to unauthorized users
```
**Root Cause**: Field-level permissions not implemented

**ERPNext Pattern Solution**:
```python
# In controller - following ERPNext sensitive field pattern
def on_load(self):
    if not frappe.has_permission(self.doctype, "write"):
        # Hide sensitive fields from read-only users
        sensitive_fields = ["cost_price", "margin", "profit"]
        for field in sensitive_fields:
            self.set_df_property(field, "hidden", 1)

# In JavaScript - client-side field hiding
frappe.ui.form.on('Your DocType', {
    refresh: function(frm) {
        // Hide fields based on user role
        if (!frappe.user.has_role(['Sales Manager', 'System Manager'])) {
            frm.toggle_display(['margin_percentage', 'cost_price'], false);
        }
    }
});
```

---

## 📊 Performance Issues

### Problem: Slow List Views
```
Error: List view taking >5 seconds to load
```
**Root Cause**: Missing database indexes or inefficient queries

**ERPNext Pattern Solution**:
```python
# Add indexes following ERPNext pattern
{
    "doctype": "Your DocType",
    "indexes": [
        {
            "fields": ["status", "company"], 
            "unique": 0
        },
        {
            "fields": ["posting_date", "company"],
            "unique": 0  
        },
        {
            "fields": ["customer", "status"],
            "unique": 0
        }
    ]
}

# Optimize list view queries
def get_list_context(context):
    # Add efficient filtering
    context.update({
        "filters": [
            ["status", "!=", "Cancelled"],
            ["company", "=", frappe.defaults.get_user_default("Company")]
        ],
        "fields": ["name", "title", "status", "modified"],  # Minimal fields
        "limit": 20,  # Pagination
        "order_by": "modified desc"
    })
```

### Problem: Memory Issues with Large Documents
```
Error: Memory exhausted when processing large documents
```
**Root Cause**: Loading entire datasets into memory

**ERPNext Pattern Solution**:
```python
# Process in batches following ERPNext batch processing pattern
def process_large_dataset(self):
    batch_size = 100
    offset = 0
    
    while True:
        records = frappe.db.sql("""
            SELECT name FROM `tabYour DocType`  
            WHERE status = 'Pending'
            LIMIT %s OFFSET %s
        """, (batch_size, offset), as_dict=True)
        
        if not records:
            break
            
        for record in records:
            self.process_single_record(record.name)
            frappe.db.commit()  # Commit each batch
        
        offset += batch_size

# Use generators for large data processing
def get_filtered_records(filters=None):
    """Generator following ERPNext pattern for memory efficiency"""
    for record in frappe.db.sql_list("""
        SELECT name FROM `tabYour DocType` 
        WHERE status = %(status)s
    """, filters, chunk_size=100):
        yield frappe.get_doc("Your DocType", record)
```

---

## 🔄 Integration Issues

### Problem: API Calls Failing
```
Error: External API integration returning errors
```
**Root Cause**: Insufficient error handling or rate limiting

**ERPNext Pattern Solution**:
```python
import requests
import time
from frappe.utils import get_url

class APIIntegration:
    def __init__(self):
        self.base_url = "https://api.example.com"
        self.timeout = 30
        self.max_retries = 3
    
    def make_api_call(self, endpoint, data=None, method="GET"):
        """API call with retry logic following ERPNext pattern"""
        for attempt in range(self.max_retries):
            try:
                response = requests.request(
                    method=method,
                    url=f"{self.base_url}/{endpoint}",
                    json=data,
                    timeout=self.timeout,
                    headers=self.get_headers()
                )
                
                if response.status_code == 200:
                    return response.json()
                elif response.status_code == 429:  # Rate limited
                    time.sleep(2 ** attempt)  # Exponential backoff
                    continue
                else:
                    frappe.log_error(
                        f"API Error: {response.status_code} - {response.text}",
                        "External API Integration"
                    )
                    
            except requests.RequestException as e:
                frappe.log_error(f"Request failed: {str(e)}", "API Integration")
                if attempt == self.max_retries - 1:
                    frappe.throw(_("Failed to connect to external service"))
                time.sleep(1)
        
        return None
    
    def get_headers(self):
        return {
            "Authorization": f"Bearer {self.get_api_token()}",
            "Content-Type": "application/json",
            "User-Agent": f"ERPNext/{frappe.__version__}"
        }
```

### Problem: Webhook Processing Failures  
```
Error: Webhooks timing out or failing silently
```
**Root Cause**: Synchronous processing of webhook data

**ERPNext Pattern Solution**:
```python
# Webhook handler following ERPNext async pattern
@frappe.whitelist(allow_guest=True, methods=['POST'])
def webhook_handler():
    """Process webhook asynchronously"""
    try:
        # Validate webhook signature
        if not validate_webhook_signature():
            frappe.throw("Invalid webhook signature")
        
        data = frappe.local.form_dict
        
        # Queue for background processing
        frappe.enqueue(
            'your_app.integrations.process_webhook_data',
            data=data,
            queue='default',
            timeout=300
        )
        
        return {"status": "success", "message": "Webhook received"}
        
    except Exception as e:
        frappe.log_error(f"Webhook processing failed: {str(e)}", "Webhook Handler")
        return {"status": "error", "message": "Processing failed"}

def process_webhook_data(data):
    """Background processing of webhook data"""
    try:
        # Process the webhook data
        doc = frappe.get_doc({
            "doctype": "Your DocType",
            "data": data
        })
        doc.insert()
        
        # Send confirmation back if needed
        send_webhook_confirmation(data.get("id"))
        
    except Exception as e:
        frappe.log_error(f"Webhook data processing failed: {str(e)}")
```

---

## 🚀 Deployment Issues

### Problem: Migration Failures in Production
```
Error: Migration failed during deployment
```
**Root Cause**: Database schema conflicts or data inconsistencies

**ERPNext Pattern Solution**:
```python
# Migration script following ERPNext pattern
import frappe
from frappe.custom.doctype.custom_field.custom_field import create_custom_fields

def execute():
    """Migration following ERPNext migration pattern"""
    try:
        # 1. Add custom fields safely
        add_custom_fields()
        
        # 2. Update existing data
        update_existing_data()
        
        # 3. Create default records
        create_default_records()
        
        frappe.db.commit()
        
    except Exception as e:
        frappe.db.rollback()
        frappe.log_error(f"Migration failed: {str(e)}", "Migration Error")
        raise

def add_custom_fields():
    """Add custom fields with error handling"""
    custom_fields = {
        "Customer": [
            {
                "fieldname": "custom_field",
                "fieldtype": "Data",
                "label": "Custom Field",
                "insert_after": "customer_name"
            }
        ]
    }
    
    try:
        create_custom_fields(custom_fields)
    except Exception as e:
        if "Duplicate" not in str(e):
            raise

def update_existing_data():
    """Update data with batch processing"""
    records = frappe.db.sql("""
        SELECT name FROM `tabCustomer` 
        WHERE custom_field IS NULL
        LIMIT 1000
    """, as_dict=True)
    
    for record in records:
        try:
            frappe.db.set_value("Customer", record.name, "custom_field", "default_value")
        except Exception as e:
            frappe.log_error(f"Failed to update {record.name}: {str(e)}")
```

### Problem: Production Performance Degradation
```
Error: Site becomes slow after deployment
```  
**Root Cause**: Missing optimizations or resource constraints

**ERPNext Pattern Solution**:
```bash
# Production optimization checklist
# 1. Enable Redis caching
redis_cache = "redis://localhost:13000"
redis_queue = "redis://localhost:11000" 
redis_socketio = "redis://localhost:12000"

# 2. Optimize database
bench execute "frappe.db.sql('ANALYZE TABLE `tabYour DocType`')"
bench execute "frappe.db.sql('OPTIMIZE TABLE `tabYour DocType`')"

# 3. Clear caches
bench clear-cache
bench clear-website-cache

# 4. Restart services
bench restart

# 5. Monitor performance
tail -f logs/bench.log
```

---

## 🆘 Emergency Procedures

### Site Won't Start
```bash
# 1. Check logs
tail -f logs/bench.log
tail -f logs/worker.log

# 2. Database connectivity
bench execute "frappe.db.sql('SELECT 1')"

# 3. Reset admin password
bench set-admin-password [new-password]

# 4. Reinstall app if corrupted
bench uninstall-app your_app --force
bench install-app your_app
```

### Data Recovery
```bash
# 1. Database backup
bench backup --with-files

# 2. Restore from backup
bench restore database_backup.sql.gz

# 3. Selective data export/import
bench export-fixtures
bench import-doc [doctype] [file.json]
```

### Performance Emergency
```python
# Quick performance fixes
# 1. Disable non-essential scheduled jobs
frappe.db.set_value("Scheduled Job Type", "job_name", "stopped", 1)

# 2. Clear all caches
frappe.clear_cache()

# 3. Optimize critical queries
frappe.db.sql("ANALYZE TABLE `tabYour Critical DocType`")
```

---

## 📞 Getting Help

### Diagnostic Commands
```bash
# System health check
bench doctor

# Database integrity check  
bench execute "frappe.db.sql('CHECK TABLE `tabYour DocType`')"

# Memory usage
bench execute "import psutil; print(f'Memory: {psutil.virtual_memory().percent}%')"

# Active sessions
bench execute "frappe.db.sql('SHOW PROCESSLIST')"
```

### Log Analysis
```bash
# Error patterns
grep -i error logs/bench.log | tail -20

# Performance issues
grep -i "slow" logs/bench.log | tail -20

# API issues
grep -i "api" logs/bench.log | tail -20
```

---

*This troubleshooting index compiles solutions from thousands of ERPNext implementations. Most issues can be resolved by following these proven patterns and procedures.*